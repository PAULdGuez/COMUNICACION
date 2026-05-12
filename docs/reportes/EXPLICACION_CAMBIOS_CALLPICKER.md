# Explicacion de Cambios CallPicker — Para Revision Senior

Fecha: 2026-05-09
Commit: 9c438fa (local, pendiente push a marketin)
Modulo: campaign_callcenter v19.0.9.11.0

---

## Pregunta 1: "El circuit breaker bloquea — el boton no se podra usar?"

**El boton SIEMPRE se puede usar.** No se desactiva, no se oculta, no cambia de color.

Lo que pasa internamente:

```
ANTES (sin circuit breaker):
  Agente clickea "Llamar"
    → POST a CallPicker → 503 error
    → Agente ve: "No se pudo iniciar la llamada"
    → Call_log se crea (evidencia de trabajo)
    → Agente clickea de nuevo 3 seg despues
    → POST a CallPicker → 503 error (CallPicker sigue caido)
    → Agente clickea otra vez...
    → 10 requests en 30 seg → CallPicker mas saturado → mas 503

AHORA (con circuit breaker):
  Agente clickea "Llamar"
    → POST a CallPicker → 503 error
    → Circuit breaker se activa por 30 seg
    → Agente ve: "El servicio de llamadas esta saturado. Reintenta en 30 segundos."
    → Call_log se crea (evidencia de trabajo)
    → Agente clickea de nuevo 5 seg despues
    → El sistema NO hace POST a CallPicker (sabe que esta caido)
    → Agente ve: "El servicio de llamadas esta saturado. Reintenta en 25 segundos."
    → Call_log se crea (evidencia de trabajo)
    → 30 seg pasan, circuit breaker se desactiva automaticamente
    → Agente clickea → POST a CallPicker → funciona → llamada exitosa
    → Circuit breaker se resetea
```

**Beneficio para el agente:** En vez de hacer 10 clicks inútiles durante un fallo, hace 1-2 y sabe exactamente cuándo reintentar. Su trabajo queda registrado igual.

**Beneficio para el sistema:** En vez de 10 requests a un servidor caído (que lo hunden más), hace 1. CallPicker se recupera más rápido.

**El call_log SIEMPRE se crea**, con o sin circuit breaker. El agente tiene evidencia de que intentó.

---

## Pregunta 2: "Que significa que el token OAuth expira?"

CallPicker usa un sistema de autenticación llamado OAuth2. Funciona así:

```
1. Odoo le pide a CallPicker: "dame un pase temporal para hacer llamadas"
2. CallPicker responde: "aqui tienes un pase, vale por 1 hora"
3. Odoo guarda ese pase (token) y lo usa para cada llamada
4. Después de 1 hora, el pase expira
5. Odoo tiene que pedir otro pase nuevo
```

**El problema ANTES:**
```
5 agentes clickean "Llamar" al mismo tiempo justo cuando el pase expira
  → Worker 1 detecta: "el pase expiró, voy a pedir uno nuevo"
  → Worker 2 detecta lo mismo: "yo también voy a pedir uno nuevo"
  → Worker 3, 4, 5: lo mismo
  → Los 5 piden un pase nuevo A LA VEZ
  → CallPicker genera 5 pases, pero solo el ultimo es valido
  → Los workers 1-4 tienen pases invalidos → error 401
  → Las llamadas de 4 agentes fallan
```

**El problema AHORA (solucionado):**
```
5 agentes clickean "Llamar" al mismo tiempo
  → Worker 1 detecta: "el pase expiró" → adquiere un "candado" en la BD
  → Worker 2 detecta lo mismo, pero ve el candado: "alguien ya lo está renovando"
    → Espera 1 segundo → relee el pase → ya está renovado → lo usa
  → Workers 3, 4, 5: igual, esperan y reusan el pase renovado
  → Solo 1 solicitud de pase a CallPicker en vez de 5
  → Todas las llamadas funcionan
```

---

## Pregunta 3: "Como se refresca el token automaticamente?"

**ANTES:** Si el pase (token) expiraba DURANTE una llamada:
```
Agente clickea → Odoo usa el pase expirado → CallPicker dice "401 Unauthorized"
  → El agente ve: "La sesion expiró. Intenta de nuevo."
  → El agente tiene que clickear OTRA VEZ manualmente
```

**AHORA:** Si el pase expira durante una llamada:
```
Agente clickea → Odoo usa el pase expirado → CallPicker dice "401 Unauthorized"
  → Odoo AUTOMATICAMENTE pide un pase nuevo
  → Odoo reintenta la llamada con el pase nuevo
  → La llamada funciona
  → El agente ni se entera de que hubo un problema
```

Solo se reintenta UNA vez. Si el segundo intento también falla, ahí sí le muestra el error al agente.

---

## Pregunta 4: "Porque ahora los webhooks duplicados se ignoran?"

CallPicker tiene un sistema de reintentos propio. Cuando envía un webhook (notificacion de que una llamada terminó), lo envía 3-5 veces "por si acaso":

```
CallPicker:
  attempt=1 → envia webhook con datos de la llamada (duracion, grabacion)
  attempt=2 → envia EL MISMO webhook otra vez (por si no llego)
  attempt=3 → otra vez
```

**ANTES:** Odoo procesaba los 3-5 webhooks como si fueran diferentes:
```
  attempt=1 → UPDATE call_log SET duration=39, recording_url='...'
  attempt=2 → UPDATE call_log SET duration=39, recording_url='...' (mismo dato)
  attempt=3 → UPDATE call_log SET duration=39, recording_url='...' (mismo dato)
  → 3 escrituras a la BD con los mismos datos
  → Si llegan al mismo tiempo: SERIALIZATION_FAILURE (conflicto de escritura)
  → PostgreSQL reintenta cada uno 3-4 veces
  → Total: 12-15 operaciones de BD para UNA llamada
```

**AHORA:** Odoo revisa si el call_log YA tiene los datos antes de procesar:
```
  attempt=1 → call_log no tiene duracion ni grabacion → procesa → OK
  attempt=2 → call_log YA tiene duracion Y grabacion → ignora → retorna 200
  attempt=3 → igual → ignora → retorna 200
  → 1 escritura a la BD + 2 lecturas rapidas
  → Sin conflictos, sin retries
```

**Dato real de los logs:** Para la llamada o-149042285, CallPicker envió 5 webhooks en 11 segundos. Con el fix, solo se procesaria el primero.

---

## Pregunta 5: "Log cada hora o cada 20 minutos?"

### Recomendacion: cada 20 minutos

**Por que 20 min es mejor que 1 hora para este caso:**
- Con 7000 llamadas/dia, en 1 hora pueden pasar ~500 llamadas
- Si algo sale mal al minuto 5, no lo sabes hasta el minuto 60
- Con 20 min, detectas en maximo 20 min
- La carga del cron es casi nula (solo lee 6 numeros de ir.config_parameter)

**Por que NO menos de 20 min:**
- Cada 5 min generaria mucho ruido en los logs
- Los contadores serian muy bajos (pocos datos por ciclo)
- Difícil distinguir patron de ruido

**Formato del log (cada 20 min):**
```
CallPicker stats: 50 calls, 8 webhooks_duped, 0 circuit_breaks, 0 token_refreshes, 0 errors_503
```

**Si los numeros estan sanos:** `circuit_breaks=0`, `errors_503=0`
**Si algo anda mal:** `circuit_breaks=5`, `errors_503=12` → investigar

---

## Resumen ejecutivo

| Cambio | Que hace | Afecta al agente? | Afecta la UX? |
|--------|----------|-------------------|---------------|
| Circuit breaker | Evita saturar CallPicker cuando esta caido | Solo el mensaje de error cambia | No |
| Token lock | Evita que 5 workers pidan pase al mismo tiempo | No, es transparente | No |
| Token retry en 401 | Si el pase expira, lo renueva y reintenta solo | No, es transparente | No |
| Dedup webhooks | Ignora notificaciones repetidas de CallPicker | No | No |
| Log stats | Imprime numeros cada 20 min para monitoreo | No | No |

**Ningun cambio modifica la interfaz del agente.** Todo es infraestructura interna.
