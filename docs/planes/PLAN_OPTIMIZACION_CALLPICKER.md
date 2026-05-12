# Plan de Optimizacion: CallPicker — Deduplicacion, Circuit Breaker y Escalabilidad

Fecha: 2026-05-09
Estado: PENDIENTE REVISION SENIOR
Modulo: campaign_callcenter
Rama: marketin
Prioridad: Alta (proxima campana 7000+ llamadas/dia)

---

## Contexto

Analisis de logs de produccion (2026-05-05 al 2026-05-09) revela que el sistema de llamadas CallPicker tiene un ciclo de amplificacion que causa errores 503 crecientes (4→19→63→81/dia), SERIALIZATION_FAILURE en PostgreSQL, y OOM kills en workers de odoo.sh. La proxima campana (7000+ llamadas/dia) hara esto insostenible.

El fix de one-shot crons (commit `00c0ca9`, ya en produccion) resolvio la acumulacion de crons zombies y optimizo el cron de webhooks pending con batch search. Los issues de este plan son el siguiente nivel de optimizacion.

---

## Diagnostico Completo

### Flujo actual de una llamada (con problemas identificados)

```
Agente clickea "Llamar"
  │
  ├─ callpicker_service.py:2434 → POST /calls/dial
  │     ├─ EXITO: crea call_log con result="no_answer", duration=0
  │     └─ ERROR 503: crea call_log con result="no_answer", notes=error
  │           └─ [PROBLEMA 1] Sin cooldown → agente reintenta en 3 seg
  │
  ├─ CallPicker envia webhooks de vuelta
  │     ├─ [PROBLEMA 2] Envia 3-5 webhooks por llamada (attempt 1,2,3)
  │     │     Todos intentan UPDATE al mismo call_log
  │     │     → SERIALIZATION_FAILURE en PostgreSQL
  │     │     → Odoo reintenta 3-4 veces cada uno
  │     │     → 15-20 queries por UNA llamada
  │     │
  │     └─ Webhooks que no matchean → webhook.pending
  │           └─ [PROBLEMA 3] Cron procesa stubborn con _try_apply
  │                 → _find_matching_log hace reverse lookup HTTP a CallPicker
  │                 → Mas carga al API
  │
  ├─ [PROBLEMA 4] Token OAuth expira
  │     └─ 5 workers refrescan simultaneamente (TOCTOU race)
  │           → 5 POST /oauth/token al mismo tiempo
  │           → Ultimo en escribir gana, otros tokens invalidados
  │
  └─ [PROBLEMA 5] Vista/computed carga 700+ call_logs de golpe
        → Query de 990 seg → PostgreSQL mata el proceso
```

### Evidencia de logs

**Webhooks duplicados (llamada o-149042285):**
```
00:51:25 → outbound_call_completed  attempt=1  → UPDATE log 7124
00:51:31 → route_extension_completed attempt=3 → UPDATE log 7124
00:51:31 → route_extension_completed attempt=3 → UPDATE log 7124 (duplicado)
00:51:35 → route_extension_completed attempt=3 → UPDATE log 7124 (duplicado)
00:51:36 → route_extension_completed attempt=2 → UPDATE log 7124 (fuera de orden)
```

**Serialization failures:**
```
ERROR: could not serialize access due to concurrent update
SERIALIZATION_FAILURE, 3 tries left, try again in 2.5686 sec...
SERIALIZATION_FAILURE, 4 tries left, try again in 0.6833 sec...
SERIALIZATION_FAILURE, 3 tries left, try again in 1.6267 sec...
```

**Query timeout 990s:**
```
PostgreSQL backend process was terminated after 990 seconds
SELECT ... FROM callcenter_call_log WHERE id IN (3200, 3256, ... 7077)
-- ~700 IDs en un solo SELECT
```

**OOM kills:**
```
Worker process (pid=4) was killed by signal 9 (memory limit)
Worker process (pid=8177) was killed by signal 9 (memory limit)
```

**Errores 503 escalando:**
```
May 5:   4 errores
May 6:  19 errores (x4.7)
May 7:  63 errores (x3.3)
May 8:  81 errores (x1.3)
May 9:   3 errores (parcial, solo 1 hora de datos)
```

### Analisis de agentes (datos reales, ultimos 7 dias)

**4131 llamadas CC + 157 llamadas Agencia = 4288 total en 7 dias (~613/dia)**

| Agente | Total | NoAnswer | 0s dur | % basura | Rafagas (<30s) |
|--------|-------|----------|--------|----------|----------------|
| CC Roberto Ramos | 1342 | 154 | 149 | 11.1% | 103 |
| Antonio Cortes | 821 | 99 | 98 | 11.9% | 24 |
| Amairani Bonilla | 728 | 120 | 117 | 16.1% | 96 |
| CC Supervisor | 640 | 174 | 172 | 26.9% | 129 |
| CC intermitente | 532 | 28 | 27 | 5.1% | 12 |

**384 rafagas detectadas en 7 dias (97% CC, 3% Agencia).**

**Patron dominante de rafagas:** no_answer → no_answer (70 veces, 54%). El agente ve "Llamada iniciada" con 0s y reintenta porque no sabe si fue error del sistema o si el cliente no contesto.

### Anatomia de las llamadas 0s + no_answer

El codigo en `callpicker_service.py:2443-2444` SIEMPRE crea el call_log con `result="no_answer"` y `duration=0`, sin importar si la llamada salio o no:

```python
# callpicker_service.py:2443-2444
log_vals = {
    "result": "no_answer",                                    # SIEMPRE
    "notes": error_notes or "Llamada iniciada via CallPicker.",  # error o default
}
```

Esto causa 3 situaciones indistinguibles en la UI:

| Caso | Nota en el log | La llamada salio? | Llega webhook? |
|------|---------------|-------------------|----------------|
| A: Error 503/timeout | "No se pudo iniciar..." | NO | NO |
| B: Cliente no contesta | "Llamada iniciada via CallPicker." | SI | SI (con duration=0) |
| C: Webhook no llego | "Llamada iniciada via CallPicker." | SI | NO |

**Conclusion: el agente NO tiene informacion para decidir si reintentar. Su comportamiento es racional.**

---

## Issues Propuestos (ordenados por impacto)

### Issue 1: Deduplicar webhooks por request_id

**Problema:** CallPicker envia el mismo webhook 3-5 veces con `attempt` incremental. Todos intentan UPDATE al mismo call_log, causando SERIALIZATION_FAILURE y 15-20 queries por llamada.

**Solucion:** Guardar `request_id` de cada webhook procesado. Ignorar si ya existe.

**Cambios tecnicos:**

#### controllers/callpicker_webhook.py
1. Al inicio del handler, extraer `request_id` del payload
2. Buscar si ya se proceso (cache en memoria o campo en call_log)
3. Si ya existe: retornar 200 inmediatamente sin procesar
4. Si es nuevo: procesar normalmente

**Opcion A — Campo en call_log:**
- Agregar campo `callpicker_last_request_id = fields.Char(index=True)`
- Antes de hacer UPDATE, verificar si el `request_id` ya esta guardado
- Pro: persistente, no se pierde en restart
- Con: un campo mas en la tabla

**Opcion B — Cache en memoria (dict global):**
- Dict `_processed_request_ids = {}` con TTL de 5 min
- Pro: cero carga en BD
- Con: se pierde en restart del worker, pero webhooks duplicados llegan en segundos

**Recomendacion:** Opcion B (cache en memoria) con TTL de 5 min. Los duplicados de CallPicker llegan en un rango de 1-10 segundos, no minutos.

**Criterio de aceptacion:**
- [ ] Webhooks con request_id ya procesado retornan 200 sin hacer UPDATE
- [ ] SERIALIZATION_FAILURE cae a ~0 por llamada
- [ ] Queries por llamada bajan de 15-20 a 3-5

**Impacto estimado:** Elimina 60-80% de queries de PostgreSQL. Es el fix de mayor impacto.

---

### Issue 2: Circuit breaker con backoff para errores 503

**Problema:** Despues de un 503 de CallPicker, el sistema permite otro request inmediatamente. El agente clickea de nuevo en 3 seg, amplificando la saturacion.

**Solucion:** Circuit breaker con backoff exponencial.

**Cambios tecnicos:**

#### models/callpicker_service.py

1. En `action_click_to_call` (antes de llamar `_dial_call`):
   - Leer `callpicker.circuit_breaker_until` de `ir.config_parameter`
   - Si `now < circuit_breaker_until`: raise UserError con mensaje claro y tiempo restante
   - Ejemplo: "CallPicker no esta disponible. Reintenta en 25 segundos."

2. En `_dial_call`, en el catch de 503:
   - Leer `callpicker.circuit_breaker_count` (contador de 503 consecutivos)
   - Calcular cooldown: `min(30 * 2^count, 120)` seg (30s, 60s, 120s max)
   - Escribir `circuit_breaker_until = now + cooldown`
   - Incrementar counter

3. En `_dial_call`, en caso de exito:
   - Reset: `circuit_breaker_count = 0`, `circuit_breaker_until = False`

**Mensaje al agente segun caso:**
- Error 503: "El servicio de llamadas esta saturado. Reintenta en X segundos."
- Error timeout: "El servicio no respondio. Reintenta en X segundos."
- Error generico: "No se pudo conectar. Verifica tu conexion."
- Exito pero no contesta: "Llamada iniciada via CallPicker." (sin cambio)

**Criterio de aceptacion:**
- [ ] Despues de un 503, el siguiente intento se bloquea por 30 seg minimo
- [ ] Despues de 3 errores 503 consecutivos, el bloqueo sube a 120 seg
- [ ] Un dial exitoso resetea el counter y el bloqueo
- [ ] El agente ve un mensaje claro con countdown ("reintenta en X seg")
- [ ] El call_log de error 503 se crea con notas distintas a las de no_answer

**Impacto estimado:** Corta el ciclo vicioso 503→reintento→mas 503. Reduce errores 503 en ~90%.

---

### Issue 3: Fix token refresh storm (TOCTOU race)

**Problema:** `_get_token()` (callpicker_service.py:44-123) tiene un race condition clasico: verifica si el token expiro (linea 64), luego lo refresca y escribe (linea 120-121). Sin lock entre ambas operaciones. Si 5 workers detectan expiracion al mismo tiempo, todos llaman a `/oauth/token` simultaneamente. El ultimo en escribir gana; los tokens anteriores se invalidan server-side, causando 401s en requests en vuelo.

**Solucion:** Lock con `SELECT FOR UPDATE` + retry en 401.

**Cambios tecnicos:**

#### models/callpicker_service.py — _get_token()

1. Usar `SELECT FOR UPDATE` en `ir.config_parameter` para el token:
   ```python
   self.env.cr.execute(
       "SELECT value FROM ir_config_parameter WHERE key = %s FOR UPDATE NOWAIT",
       [token_key]
   )
   ```
2. Si `NOWAIT` falla (otro worker tiene el lock): esperar 1 seg y releer el token (ya refrescado por el otro worker)
3. Agregar retry en 401:
   - En `_dial_call`, si 401: invalidar token cacheado, llamar `_get_token(force_refresh=True)`, reintentar UNA vez
   - Si el retry tambien falla: raise UserError como ahora

**Criterio de aceptacion:**
- [ ] Solo 1 worker refresca el token simultaneamente
- [ ] Otros workers esperan y reusan el token refrescado
- [ ] Un 401 durante _dial_call refresca el token y reintenta automaticamente
- [ ] Si el retry tambien es 401: raise UserError (no loop infinito)

**Impacto estimado:** Elimina ~80% de requests a /oauth/token. Elimina 401s por token invalidado.

---

### Issue 4: Investigar y resolver query de 990 seg timeout

**Problema:** Un SELECT de ~700 IDs en `callcenter_call_log` alcanza el timeout de 990 seg de PostgreSQL en odoo.sh. No sabemos que lo dispara — podria ser:
- Una vista tree/list que carga todos los logs de un agente
- Un campo computed store=True que recorre todos los logs
- El seller_dashboard controller
- Una accion en batch (export, report)

**Investigacion necesaria:**

1. Verificar si los IDs (3200-7077) corresponden a un agente, campana, o task especifico
2. Revisar si hay campos computed con `store=True` en `callcenter.call.log` o modelos relacionados que disparen recomputo masivo
3. Revisar `seller_dashboard.py` (ya tiene warnings de `read_group` deprecado)
4. Revisar si hay acciones del admin que carguen todos los logs (export XLSX, reportes)

**Posibles soluciones (depende de la causa):**
- Si es un computed field: agregar `@api.depends` mas selectivo o quitar `store=True`
- Si es una vista: agregar paginacion o domain mas restrictivo
- Si es el dashboard: optimizar queries (ya se sabe que usa `read_group` deprecado)
- Si es un reporte: agregar limit o procesarlo en batches

**Criterio de aceptacion:**
- [ ] Identificar que proceso genera el SELECT de 700+ IDs
- [ ] Reducir el query a <5 seg de ejecucion
- [ ] No hay queries de >30 seg en logs de produccion

---

### Issue 5: Migrar read_group deprecado en seller_dashboard.py

**Problema:** `seller_dashboard.py` lineas 108, 136, 146 usan `read_group()` que esta deprecado en Odoo 19. Genera warnings en logs y puede dejar de funcionar en futuras versiones.

**Solucion:** Migrar a `_read_group()` (API interna de Odoo 19).

**Cambios tecnicos:**

#### controllers/seller_dashboard.py
- Linea 108: `CallLog.read_group(...)` → `CallLog._read_group(...)`
- Linea 136: `CallLog.read_group(...)` → `CallLog._read_group(...)`
- Linea 146: `CallLog.read_group(...)` → `CallLog._read_group(...)`

**Nota:** La API de `_read_group` en Odoo 19 tiene firma diferente a `read_group`. Revisar documentacion antes de migrar.

**Criterio de aceptacion:**
- [ ] Sin warnings de DeprecationWarning en logs
- [ ] Dashboard funciona igual que antes
- [ ] Queries del dashboard ejecutan en <2 seg

---

## Correcciones post-review

Observaciones incorporadas tras revision:

| # | Observacion | Accion |
|---|-------------|--------|
| 1 | Cache en memoria no funciona multi-worker (odoo.sh usa 4+ workers) | Issue 1 cambia a campo en call_log o check de campos existentes (si log ya tiene duration>0 y recording_url, ignorar webhook) |
| 2 | Circuit breaker via ir.config_parameter es global (bloquea todos los workers) | Es intencional — si CallPicker esta caido, esta caido para todos |
| 3 | FOR UPDATE NOWAIT lanza errores frecuentes bajo carga | Issue 3 cambia a FOR UPDATE SKIP LOCKED — mas robusto |
| 4 | Query 990s deberia investigarse antes de issues 1-3 | Se investiga EN PARALELO, no bloquea 1-3 (ocurrio 1 vez vs 503s que son cientos/dia) |
| 5 | Limpiar 1171 webhooks expirados | Tarea de mantenimiento, no issue formal |

---

## Orden de implementacion recomendado

```
Issues 1+2+3: Implementar y probar JUNTOS en staging
  │
  │  IMPORTANTE: Los 3 issues interactuan y deben probarse juntos.
  │  - Circuit breaker (2) reduce la carga
  │  - Deduplicacion (1) reduce queries
  │  - Token lock (3) previene 401s
  │  Probar solo 1 sin los otros puede dar mejoras falsas
  │  porque el volumen de prueba en staging es bajo.
  │
  ├─ Simular carga en staging: 50+ llamadas rapidas con multiples agentes
  ├─ Verificar que SERIALIZATION_FAILURE cae a ~0
  ├─ Verificar que 503 no se amplifican
  ├─ Verificar que el log counter (Issue 6) reporta numeros sanos
  │
  ↓ (en paralelo)
Issue 4: Investigar query 990s timeout
  ↓
Issue 5: read_group deprecado (menor prioridad)
  ↓
Issue 6: Log counter por hora (monitoreo continuo)
```

**Issues 1-3 deben estar en produccion JUNTOS, ANTES de la proxima campana.**

---

### Issue 6: Log counter por hora (monitoreo continuo)

**Problema:** Actualmente se depende de scripts manuales (`verificar_fix_oneshot_v4.txt`) para saber si algo esta mal. No hay monitoreo automatico.

**Solucion:** Agregar un log estructurado que se imprima cada hora con metricas clave.

**Implementacion:**

Nuevo cron (cada hora) o hook en el cron de webhooks pending que acumule contadores:

```
CallPicker hourly: 150 calls, 12 webhooks_duped, 0 circuit_breaks, 1 token_refresh, 0 errors_503
```

**Contadores:**
- `calls`: llamadas originadas via `action_click_to_call`
- `webhooks_duped`: webhooks ignorados por deduplicacion (request_id repetido)
- `circuit_breaks`: veces que el circuit breaker bloqueo una llamada
- `token_refresh`: veces que se refresco el token OAuth
- `errors_503`: errores 503 de CallPicker

**Almacenamiento:** Variables en `ir.config_parameter` con prefijo `callpicker.stats.*`, reset cada hora al imprimir.

**Criterio de aceptacion:**
- [ ] Cada hora aparece una linea en los logs con las 5 metricas
- [ ] Si `webhooks_duped` es >50% de calls, hay un problema de CallPicker
- [ ] Si `circuit_breaks` es >10/hora, CallPicker esta inestable
- [ ] Si `errors_503` sube consistentemente, escalar a CallPicker

---

### Mantenimiento: Limpiar webhooks expirados

No es un issue formal — ejecutar manualmente o agregar en la migration del proximo push:

```python
# En consola odoo.sh:
env['callcenter.webhook.pending'].sudo().search([('state', '=', 'expired')]).unlink()
env.cr.commit()

# O agregar auto-cleanup mensual en el cron de webhooks pending
```

---

## Impacto estimado combinado

| Metrica | Actual (613 calls/dia) | Con issues 1-3 | Escalado a 7000 calls/dia |
|---------|----------------------|----------------|--------------------------|
| Webhooks procesados/llamada | 3-5 | 1-2 (deduplicados) | 1-2 |
| Queries PostgreSQL/llamada | 15-20 | 3-5 | 3-5 |
| Errores 503/dia | 81 (creciendo) | <10 | <10 |
| SERIALIZATION_FAILURE/dia | Docenas | ~0 | ~0 |
| Token refreshes simultaneos | 5 (race) | 1 (locked) | 1 |
| OOM kills | Frecuentes | Raros | Raros |

---

## Riesgos

| Issue | Riesgo | Mitigacion |
|-------|--------|------------|
| 1 (dedup) | Bajo — solo agrega un check al inicio del handler | Cache en memoria con TTL corto, no afecta BD |
| 2 (circuit breaker) | Bajo — solo agrega validacion antes de _dial_call | El bloqueo es temporal (max 120s), no permanente |
| 3 (token lock) | Medio — `FOR UPDATE NOWAIT` puede fallar | Fallback: esperar y releer, no bloquear indefinidamente |
| 4 (query 990s) | Desconocido — depende de la causa raiz | Investigar antes de implementar |
| 5 (read_group) | Bajo — cambio de API documentado | Probar en staging primero |

---

## Datos de soporte

### Scripts de analisis disponibles (en Downloads)

- `verificar_fix_oneshot_v4.txt` — Estado de one-shots y webhooks pending
- `forzar_limpieza_urgente.txt` — Limpieza manual de one-shots (ORM, compatible Odoo 19)
- `analisis_llamadas_agentes.txt` — Top agentes, rafagas, errores 503 (CC vs Agencia)
- `analisis_cc_supervisor.txt` — Analisis profundo de patron de llamadas del supervisor

### Archivos del codebase relevantes

| Archivo | Lineas clave | Que tiene |
|---------|-------------|-----------|
| `callpicker_service.py` | 44-123 | `_get_token` — race condition TOCTOU |
| `callpicker_service.py` | 127-199 | `_dial_call` — sin manejo de 503, sin backoff |
| `callpicker_service.py` | 2341-2492 | `action_click_to_call` — siempre result=no_answer |
| `callpicker_webhook.py` | 256-845 | Webhook handler — sin deduplicacion |
| `callpicker_webhook.py` | 423,569,795 | Reverse lookups — 1-3 HTTP calls por webhook |
| `webhook_pending.py` | 326-495 | Cron batch — ya optimizado con batch search |
| `seller_dashboard.py` | 108,136,146 | read_group deprecado |
| `call_log.py` | 172-217 | create() override |
