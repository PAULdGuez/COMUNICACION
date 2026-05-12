# Plan de Implementacion: Eliminar One-Shot Crons CallPicker + Batch Search Optimizado

Fecha: 2026-05-08
Estado: LISTO PARA IMPLEMENTAR
Owner: paul@itgreen.io
Prioridad: Alta (bloqueante para proxima campana 7000+ llamadas/dia)
Esfuerzo: M (medium)

---

## Contexto

Los crons one-shot de CallPicker crean un registro `ir.cron` nuevo por cada llamada completada para intentar matchear el webhook con el `callcenter.call.log` ~10 segundos despues. El problema: el mecanismo de desactivacion (`_deactivate_oneshot_cron` con `ilike`) no funciona correctamente, causando acumulacion de crons zombies (1200+ en produccion). Ademas, el cron recurrente `_cron_process_pending` (cada 1 min) ya cubre el mismo ciclo de vida, haciendo los one-shots redundantes.

Con la proxima campana de 7000+ llamadas/dia, el patron actual (5 queries secuenciales por registro pending) no escala. Se necesita batch search para mantener la carga constante sin importar el volumen.

---

## Paso 1: ANALIZAR — estado actual del sistema

### Flujo actual (con one-shots)

```
Webhook llega (outbound_call_completed)
  │
  ├─► Crea registro en callcenter.webhook.pending (state=pending)
  │
  ├─► Crea cron one-shot ir.cron (delay ~10 seg)
  │     │
  │     └─► _run_delayed_fetch_for_call_id(call_id)
  │           ├─ Busca pending por call_id
  │           ├─ Intenta match individual (5 queries)
  │           ├─ Si matchea → aplica + intenta desactivar cron (FALLA)
  │           └─ Si no → deja al cron recurrente
  │
  └─► Cron recurrente (cada 1 min)
        └─► _cron_process_pending()
              ├─ Busca TODOS los pending
              ├─ Para CADA uno: _try_apply() → 5 queries secuenciales
              └─ Expira los vencidos
```

### Problemas identificados

| # | Severidad | Categoria | Descripcion |
|---|-----------|-----------|-------------|
| 1 | Critica | Acumulacion | One-shots no se desactivan → 1200+ crons zombies en produccion |
| 2 | Critica | Escalabilidad | 5 queries × N pending por ciclo → no escala a 7000 llamadas/dia |
| 3 | Alta | Redundancia | El cron recurrente ya hace el mismo trabajo que los one-shots |
| 4 | Alta | BD | INSERT/UPDATE constante en ir.cron (tabla critica de Odoo) causa locks |
| 5 | Media | Codigo muerto | `_deactivate_oneshot_cron` con ilike nunca funciono correctamente |
| 6 | Media | Duplicacion | Una llamada puede generar 2 one-shots (L532 + L769 del controller) |

### Archivos involucrados

| Archivo | Que tiene | Que cambiar |
|---------|-----------|-------------|
| `campaign_callcenter/models/webhook_pending.py` | `_run_delayed_fetch_for_call_id`, `_deactivate_oneshot_cron`, `_cron_process_pending` | Eliminar one-shot methods, reescribir cron con batch search |
| `campaign_callcenter/models/callpicker_service.py` | `_schedule_post_call_fetch` (L2920-2973) | Eliminar metodo completo |
| `campaign_callcenter/controllers/callpicker_webhook.py` | 2 llamadas a `_schedule_post_call_fetch` (L528-538, L765-775) | Eliminar ambas llamadas |
| `campaign_callcenter/data/cron_data.xml` | `cron_process_pending_webhooks` interval=1min | Cambiar a interval=15 segundos |
| `campaign_callcenter/__manifest__.py` | Version actual | Bump minor (feature) |

---

## Paso 2: DOCUMENTAR — cambios tecnicos especificos

### Cambio 1: Eliminar patron one-shot (3 archivos)

#### Problema actual

Cada `outbound_call_completed` crea un `ir.cron` nuevo. El mecanismo de desactivacion busca por nombre con `ilike` pero falla porque:
- El `call_id` pasado puede diferir del original (numerico vs prefixed)
- El cron recurrente puede procesar el pending antes que el one-shot, pero el one-shot sigue activo
- No hay `numbercall=1` en Odoo 19 (campo eliminado)

#### Comportamiento deseado

Sin one-shots. El webhook crea el registro pending y el cron recurrente lo procesa en el siguiente ciclo (maximo 15 seg).

#### Cambios tecnicos

##### models/callpicker_service.py
1. **Eliminar** metodo `_schedule_post_call_fetch` (lineas ~2920-2973 completas)
2. **Conservar** `_get_post_call_delay` (puede usarse para otra cosa) — o eliminar si no tiene otro uso
3. **Conservar** `_run_post_call_fetch` (el mini-backfill ICN, se usa como fallback en batch)

##### controllers/callpicker_webhook.py
1. **Eliminar** bloque L527-538: schedule post-call fetch en el buffer path
   ```python
   # ELIMINAR: lineas 527-538
   # For outbound_call_completed, also schedule a one-shot delayed fetch
   if event == "outbound_call_completed":
       try:
           safe_call_id = identifier_info.get("numeric") or call_id or ""
           if safe_call_id:
               request.env["callpicker.service"].sudo()._schedule_post_call_fetch(safe_call_id)
       except Exception as sched_exc:
           ...
   ```

2. **Eliminar** bloque L762-775: schedule post-call fetch en el matched log path
   ```python
   # ELIMINAR: lineas 762-775
   if event == "outbound_call_completed" and not log.recording_url:
       try:
           safe_call_id = identifier_info.get("numeric") or call_id or ""
           if safe_call_id:
               request.env["callpicker.service"].sudo()._schedule_post_call_fetch(safe_call_id)
       except Exception as sched_exc:
           ...
   ```

##### models/webhook_pending.py
1. **Eliminar** metodo `_run_delayed_fetch_for_call_id` (lineas 377-440)
2. **Eliminar** metodo `_deactivate_oneshot_cron` (lineas 442-451)

---

### Cambio 2: Reescribir `_cron_process_pending` con batch search

#### Problema actual

```python
# ACTUAL: O(n × 5) queries
for rec in pending:
    rec._try_apply()  # cada _try_apply hace _find_matching_log con 5 searches
```

Con 300 pending = 1500 queries por ciclo. Con 7000 llamadas/dia = potencialmente 500+ pending = 2500+ queries/min.

#### Comportamiento deseado

```python
# NUEVO: O(5) queries constante
# 1. Recopilar todos los identificadores
# 2. Hacer 1 query por tipo de criterio (4 queries total)
# 3. Mapear resultados a pending records
# 4. Aplicar matches en batch
```

#### Cambios tecnicos

##### models/webhook_pending.py — nuevo `_cron_process_pending`

```python
@api.model
def _cron_process_pending(self):
    now = fields.Datetime.now()

    # ── 1. Early exit ──
    pending = self.search([
        ("state", "=", "pending"),
        "|", ("expires_at", "=", False), ("expires_at", ">", now),
    ], limit=500)

    if not pending:
        self._expire_stale_records(now)
        return

    # ── 2. Recopilar identificadores ──
    uid_map = {}       # unique_call_id → [rec, ...]
    route_map = {}     # route_id → [rec, ...]
    callid_map = {}    # call_id → [rec, ...]
    uidprefix_map = {} # call_id con o-/i- prefix → [rec, ...]

    for rec in pending:
        if rec.call_id:
            callid_map.setdefault(rec.call_id, []).append(rec)
            if rec.call_id.lower().startswith(("o-", "i-")):
                uidprefix_map.setdefault(rec.call_id, []).append(rec)
        if rec.unique_call_id:
            uid_map.setdefault(rec.unique_call_id, []).append(rec)
        if rec.route_id:
            route_map.setdefault(rec.route_id, []).append(rec)

    CallLog = self.env["callcenter.call.log"].sudo()
    matched = {}  # rec.id → log

    # ── 3. Batch search: 4 queries (prioridad descendente) ──

    # 3a. UID prefix como unique_call_id (o-xxx / i-xxx)
    if uidprefix_map:
        logs = CallLog.search([
            ("callpicker_unique_call_id", "in", list(uidprefix_map.keys()))
        ])
        for log in logs:
            for rec in uidprefix_map.get(log.callpicker_unique_call_id, []):
                matched.setdefault(rec.id, log)

    # 3b. unique_call_id directo
    if uid_map:
        remaining_uids = [k for k, recs in uid_map.items()
                          if not all(r.id in matched for r in recs)]
        if remaining_uids:
            logs = CallLog.search([
                ("callpicker_unique_call_id", "in", remaining_uids)
            ])
            for log in logs:
                for rec in uid_map.get(log.callpicker_unique_call_id, []):
                    matched.setdefault(rec.id, log)

    # 3c. route_id
    if route_map:
        remaining_routes = [k for k, recs in route_map.items()
                            if not all(r.id in matched for r in recs)]
        if remaining_routes:
            logs = CallLog.search([
                ("callpicker_route_id", "in", remaining_routes)
            ])
            for log in logs:
                for rec in route_map.get(log.callpicker_route_id, []):
                    matched.setdefault(rec.id, log)

    # 3d. call_id
    if callid_map:
        remaining_cids = [k for k, recs in callid_map.items()
                          if not all(r.id in matched for r in recs)]
        if remaining_cids:
            logs = CallLog.search([
                ("callpicker_call_id", "in", remaining_cids)
            ])
            for log in logs:
                for rec in callid_map.get(log.callpicker_call_id, []):
                    matched.setdefault(rec.id, log)

    # ── 4. Aplicar matches ──
    applied = 0
    for rec in pending:
        if rec.id in matched:
            try:
                log = matched[rec.id]
                payload = json.loads(rec.payload_json or "{}")
                update_vals = rec._build_update_vals(payload, log)
                if update_vals:
                    log.sudo().write(update_vals)
                rec.write({
                    "state": "applied",
                    "applied_log_id": log.id,
                    "last_error": False,
                })
                log.sudo()._notify_log_updated()
                applied += 1
            except Exception as exc:
                _logger.exception("Batch apply failed for pending %s: %s", rec.id, exc)
                rec.write({"last_error": str(exc)[:500]})
        else:
            # No match en batch — incrementar intentos
            rec.attempts += 1
            if rec.attempts >= MAX_RETRY_ATTEMPTS:
                rec.write({
                    "state": "failed",
                    "last_error": f"Max retries ({MAX_RETRY_ATTEMPTS}) sin match."
                })

    # ── 5. Semantic match diferido (costoso) — solo pending con 3+ intentos ──
    stubborn = self.search([
        ("state", "=", "pending"),
        ("attempts", ">=", 3),
        ("attempts", "<", MAX_RETRY_ATTEMPTS),
        "|", ("expires_at", "=", False), ("expires_at", ">", now),
    ], limit=20)  # max 20 para no saturar
    for rec in stubborn:
        try:
            if rec._try_apply():
                applied += 1
        except Exception as exc:
            rec.write({"last_error": str(exc)[:500]})

    # ── 6. Expirar vencidos ──
    self._expire_stale_records(now)

    _logger.info(
        "WebhookPending cron: batch=%d, applied=%d, stubborn_checked=%d",
        len(pending), applied, len(stubborn),
    )


def _expire_stale_records(self, now=None):
    """Expira registros pending cuyo TTL ya vencio."""
    if not now:
        now = fields.Datetime.now()
    expired = self.search([
        ("state", "=", "pending"),
        ("expires_at", "!=", False),
        ("expires_at", "<=", now),
    ])
    if expired:
        expired.write({"state": "expired"})
        _logger.info("WebhookPending: expired %d stale records", len(expired))
```

---

### Cambio 3: Cron a 15 segundos

#### data/cron_data.xml

```xml
<!-- ANTES -->
<field name="interval_number">1</field>
<field name="interval_type">minutes</field>

<!-- DESPUES -->
<field name="interval_number">15</field>
<field name="interval_type">seconds</field>
```

**Nota:** Odoo 19 soporta `seconds` como interval_type en ir.cron. Verificar que funcione en odoo.sh (si no, usar `interval_number=1, interval_type=minutes` como fallback seguro y documentar).

---

### Cambio 4: Migration script para limpiar one-shots existentes

#### migrations/19.0.X.Y.0/pre-migrate.py (nuevo archivo)

```python
def migrate(cr, version):
    """Desactivar y eliminar crons one-shot acumulados."""
    cr.execute("""
        UPDATE ir_cron SET active = false
        WHERE name LIKE '%%[one-shot] CallPicker%%'
    """)
    cr.execute("""
        DELETE FROM ir_cron
        WHERE name LIKE '%%[one-shot] CallPicker%%'
        AND active = false
    """)
```

---

### Cambio 5: Version bump

#### __manifest__.py

```python
# ANTES
"version": "19.0.X.Y.Z",

# DESPUES (minor bump — feature)
"version": "19.0.X.(Y+1).0",
```

---

## Paso 3: AUDITAR — verificacion pre-implementacion

### Checklist

| # | Verificacion | Estado |
|---|-------------|--------|
| 1 | `_run_post_call_fetch` no se usa en otro lugar (excepto one-shot) | Verificar con grep |
| 2 | `_get_post_call_delay` no se usa en otro lugar | Verificar con grep |
| 3 | `ir.cron` soporta `interval_type=seconds` en Odoo 19 | Verificar en codigo fuente Odoo |
| 4 | `_try_apply` sigue funcionando individual (para semantic match fallback) | Si, no se modifica |
| 5 | `_build_update_vals` funciona fuera de `_try_apply` | Si, solo necesita payload + log |
| 6 | `_notify_log_updated` funciona con sudo() | Si, ya se usa asi |
| 7 | No hay otros callers de `_schedule_post_call_fetch` | Verificar con grep (solo 2 en controller) |
| 8 | No hay otros callers de `_run_delayed_fetch_for_call_id` | Verificar (solo one-shot crons via code) |
| 9 | Migration script funciona en odoo.sh | Usar `%%` en LIKE (escape Python de %) |
| 10 | `limit=500` en pending es suficiente para 7000 calls/dia | Si: 7000/dia ÷ 4 ciclos/min ÷ 60 min ≈ 1.2/ciclo promedio, picos ~50 |

---

## Paso 4: CONFIGURAR — contexto para el agente

Ya configurado en `AGENTS.md`. Reglas relevantes:
- Rama: `marketin`
- Version bump obligatorio
- Pull antes de push
- Actualizar `PROJECT_GUIDE.md` si cambia comportamiento
- No usar `_sql_constraints`, no extender `res.users`
- Crons no usan `numbercall`, `doall`, `priority`

---

## Paso 5: PLANIFICAR — issues

### Issue unica (no es un epic, es un fix autocontenido)

```
Titulo: fix: eliminar one-shot crons CallPicker + batch search optimizado
Descripcion:
  - Eliminar _schedule_post_call_fetch de callpicker_service.py
  - Eliminar 2 llamadas en callpicker_webhook.py (L528-538, L762-775)
  - Eliminar _run_delayed_fetch_for_call_id y _deactivate_oneshot_cron de webhook_pending.py
  - Reescribir _cron_process_pending con batch search (4 queries IN vs N×5 queries)
  - Agregar _expire_stale_records como metodo separado
  - Agregar semantic match diferido (solo pending con 3+ intentos, limit 20)
  - Cambiar intervalo del cron a 15 segundos
  - Migration script para limpiar one-shots acumulados en produccion
  - Version bump minor

Acceptance criteria:
  - [ ] No se crean nuevos ir.cron al recibir outbound_call_completed
  - [ ] _cron_process_pending usa batch search (maximo 5-8 queries por ciclo)
  - [ ] Cron corre cada 15 segundos
  - [ ] Pending records se matchean y aplican correctamente (recording_url, duration, result)
  - [ ] Records expirados se marcan como expired
  - [ ] Records con 65+ intentos se marcan como failed
  - [ ] Semantic match (fallback costoso) solo se ejecuta para pending con 3+ intentos
  - [ ] Migration limpia one-shots existentes en upgrade
  - [ ] No hay llamadas a _schedule_post_call_fetch en el codebase
  - [ ] No hay metodos _run_delayed_fetch_for_call_id ni _deactivate_oneshot_cron
  - [ ] Version bump incluido en el commit

Prioridad: Alta
Esfuerzo: M
```

---

## Paso 6: IMPLEMENTAR — orden de ejecucion

### Commit unico con todos los cambios:

```bash
# Orden de edicion:
1. callpicker_service.py      → eliminar _schedule_post_call_fetch
2. callpicker_webhook.py      → eliminar 2 bloques de llamada
3. webhook_pending.py          → eliminar one-shot methods + reescribir cron
4. cron_data.xml               → cambiar intervalo a 15 seg
5. migrations/pre-migrate.py   → script de limpieza
6. __manifest__.py             → version bump

# Verificacion pre-push:
grep -rn "_schedule_post_call_fetch" .  # debe dar 0 resultados
grep -rn "_run_delayed_fetch_for_call_id" .  # debe dar 0 resultados
grep -rn "_deactivate_oneshot_cron" .  # debe dar 0 resultados

# Commit:
git commit -m "fix: eliminar one-shot crons CallPicker + batch search optimizado

- Elimina patron one-shot que acumulaba 1200+ crons zombies
- Reescribe _cron_process_pending con batch search (4 queries IN)
- Cron a 15 seg para latencia comparable al one-shot
- Semantic match diferido solo para pending con 3+ intentos
- Migration script limpia one-shots existentes en produccion
- Escala a 7000+ llamadas/dia sin degradacion"
```

---

## Paso 7: ENTREGAR — verificacion post-deploy

### Verificacion inmediata (odoo.sh staging)

1. Verificar que el modulo se instala sin errores en el log
2. Verificar que NO se crean nuevos crons `[one-shot]` al hacer una llamada
3. Verificar que los pending se procesan cada ~15 seg
4. Verificar que los one-shots antiguos fueron eliminados por la migration

### Verificacion con carga

1. Simular 100 webhooks rapidos y verificar que el cron los procesa en batch
2. Revisar logs: debe decir `batch=N, applied=X` en vez de logs individuales
3. Monitorear queries/ciclo (debe ser ~5-8 constante)

### Verificacion en produccion (post-merge)

1. Ejecutar `forzar_limpieza_urgente.txt` si la migration no alcanzo
2. Verificar con `verificar_fix_oneshot_v3.txt`
3. Monitorear durante las primeras horas de la campana

---

## Numeros de escalabilidad final

| Escenario | Queries/ciclo | Ciclos/hora | Queries/hora | Viable |
|-----------|--------------|-------------|--------------|--------|
| 100 calls/dia | 5-8 | 240 | ~1,920 | Si |
| 5,000 calls/dia | 5-8 | 240 | ~1,920 | Si |
| 7,000 calls/dia | 5-8 | 240 | ~1,920 | Si |
| 20,000 calls/dia | 5-8 | 240 | ~1,920 | Si |

**La carga es CONSTANTE** porque batch search con `IN` no depende del volumen de pending (hasta el limit de 500 por ciclo, que cubre incluso picos extremos).

---

## Riesgo

**Bajo.**
- Es refactor interno del cron, no cambia la interfaz publica
- El webhook controller sigue creando registros pending igual
- `_try_apply` y `_build_update_vals` no se modifican
- Si el batch falla, el semantic match (fallback) cubre los edge cases
- La migration es idempotente (safe to re-run)

## Fallback

Si `interval_type=seconds` no funciona en odoo.sh:
- Cambiar a `interval_number=1, interval_type=minutes` (latencia 60 seg)
- Documentar como limitacion de odoo.sh
