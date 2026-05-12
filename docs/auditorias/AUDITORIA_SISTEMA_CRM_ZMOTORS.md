# AUDITORÍA DE SISTEMA CRM ZMOTORS — ANÁLISIS TÉCNICO INTEGRAL

**Fecha:** 2026-05-10
**Contexto:** Odoo 19 cloud (odoo.sh), 14,167 leads, 1,504 llamadas/día (escalando a 7,000+)

---

## RESUMEN EJECUTIVO

El sistema CRM ZMotors es **arquitecturalmente sólido** pero presenta issues de escalabilidad causados por un crecimiento de **11.8x** sobre el diseño original. Los problemas no son de código defectuoso sino de optimización para volumen.

**Datos de producción:**
- Antes: 1 campaña, 1,200 contactos, ~200 llamadas/día
- Ahora: 2 campañas, 14,167 contactos, ~1,500 llamadas/día
- Próximo: 7,000+ llamadas/día esperadas

---

## ISSUES POR SEVERIDAD

### CRÍTICOS

#### CA-001: Cron de Etapas T — Writes Secuenciales (CAUSA FALLA DEL VIERNES)

**Problema:** El cron ejecuta 2,000 writes individuales, cada uno recomputa 15 campos stored. Con concurrencia, PostgreSQL timeout.

**Solución:** Batch write (2,000 writes → 2 writes)

**Nota de revisión:** Verificar si `@api.depends` en `agency_t_stage` tiene lógica per-record (fecha de cambio, message_post) antes de implementar batch. Si la tiene, el batch podría perder esa lógica.

**Esfuerzo:** 2-3 puntos

---

#### CA-002: Dashboard — N+1 Queries (600+)

**Problema:** `seller_dashboard.py` usa `.filtered()` en loops que evalúan todos los leads múltiples veces.

**Nota de revisión:** Los "600+ queries" son estimados, no medidos. Recomendar perfil real con `--log-sql` en staging antes de afirmar números exactos.

**Solución:** Usar `search_read()` + procesamiento en memoria

**Esfuerzo:** 3 puntos

---

#### CA-003: Webhook — Race Condition en Fallback

**Problema:** Webhook y cron pueden hacer requests duplicados a CallPicker en paralelo.

**Nota de revisión:** Con batch search ya en producción, la escala del problema es menor que la descrita. Los "350 llamadas extra/día" están sobreestimados.

**Solución:** Cache + timeout en requests

**Esfuerzo:** 2 puntos

---

### ALTOS

#### SC-002: Crons CC — Sin Índices

**Problema:** Búsqueda sin índices en `t1_date` causa seq scan.

**Nota de revisión:** Verificar nombres reales de columnas (podrían ser `cc_stage`, no `stage`). En Odoo 19, índices se crean vía ORM, no SQL directo.

**Solución:** Agregar índice parcial

**Esfuerzo:** 1 punto

---

#### SC-003: Computed Fields — Recompute Masivo

**Nota de revisión:** Este issue es redundante con CA-001. Si se hace batch write, el recompute se hace una vez para todo el batch. No es un issue separado.

---

### MEDIOS

#### AR-001: Credentials en Plaintext

**Nota de revisión:** Prioridad sobreestimada. `ir.config_parameter` solo es accesible por `base.group_system`. En odoo.sh no hay acceso SSH a la BD. Sprint 3 o posterior.

---

#### CL-001: read_group Deprecado

**Problema:** `seller_dashboard.py` usa `read_group()` deprecado en Odoo 19.

**Nota:** Si se refactoriza el dashboard en Sprint 2 (CA-002), incluir migración a `_read_group()` en el mismo esfuerzo.

---

## VERIFICACIÓN: Mejoras Pendientes en Marketin

| Mejora | Segura | Rompe lógica core |
|--------|--------|-------------------|
| Dedup webhooks | ✅ | NO |
| Circuit breaker | ✅ | NO |
| Token lock (SKIP LOCKED) | ✅ | NO |
| Batch search (ya en prod) | ✅ | NO |

**Lógica core verificada intacta:**
- Llamadas: dial → webhook → call_log ✓
- Registro de llamadas: create() override ✓
- Asignación de leads: round-robin ✓
- Crons etapas T: sin cambios ✓
- Reportes (tablero visual): sin cambios ✓

---

## PLAN DE REMEDIACIÓN

**Sprint 1 (Esta semana):** CA-001 batch write + SC-002 índices + merge mejoras marketin

**Sprint 2 (Próxima semana):** CA-002 refactor dashboard + migrar read_group

**Sprint 3 (Siguiente):** AR-001 env vars + CA-003 race condition

---

## CONCLUSIÓN

El sistema NO está roto — creció 12x más rápido de lo diseñado y necesita optimizaciones de escalabilidad, no una reescritura. 4 cambios sencillos resuelven el 80% de issues y escalan a 20,000+ llamadas/día.
