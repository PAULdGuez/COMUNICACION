# PENDIENTES — CRM ZMotors (Turbo Days)

Última actualización: 2026-05-12

---

## URGENTE (Antes de Próxima Campaña)

### En Marketin — Pendientes de Merge a Producción
- [ ] Dedup webhooks (evita 3-5 webhooks por llamada)
- [ ] Circuit breaker (evita saturar CallPicker con reintentos)
- [ ] Token lock SKIP LOCKED (evita 5 workers refrescando token al mismo tiempo)

### No Implementado — Requiere Desarrollo
- [ ] **Batch write cron etapas T** — fix de la falla del viernes. El cron hace 14,167 writes individuales. Debe ser batch. VERIFICAR `@api.depends("agency_t_stage")` antes de implementar.
- [ ] **Índices en tablas de crons** — verificar nombres reales de columnas con grep antes de crear índices.
- [ ] **Bug 44 leads inconsistentes** — call_log actualizado a "contacted" pero etapa de agencia no avanzó. Investigar auto-avance en call_log.py:229-232 vs webhook flow.

---

## IMPORTANTE (Semana 2)

- [ ] Refactor dashboard seller_dashboard.py (N+1 queries, usar search_read)
- [ ] Migrar read_group deprecado a _read_group en Odoo 19
- [ ] Wizard selector para "Enviar a Todos" y "Enviar Prueba" (elegir canales)

---

## MEDIO (Semana 3+)

- [ ] Botón etapas — Opción B o C (quitar "Cita Confirmada" manual para vendedores)
- [ ] ir.config para habilitar/deshabilitar wizard referidos
- [ ] PDF visual (QWeb template para reportes programados)
- [ ] Cola WA con límite 50/hora por instancia
- [ ] Race condition en webhook fallback 3 (cache + timeout)
- [ ] Credentials en env vars (ir.config_parameter → os.getenv)

---

## FEATURES PENDIENTES

- [ ] Lead digital / Landing page (CC envía WA → cliente confirma interés → auto gestionado OK)
- [ ] Desglose referidos completo en dashboards (misma estructura que leads naturales)
- [ ] Separar crons pesados a horarios sin concurrencia (2-4am MX)
- [ ] Log counter cada 20 min (calls, webhooks_duped, circuit_breaks, token_refreshes, errors_503)

---

## INVESTIGACIONES PENDIENTES

- [ ] Diferencia de llamadas CRM vs CallPicker (240 vs 130 por agente) — probablemente webhooks duplicados
- [ ] Query de 990 segundos en PostgreSQL — identificar qué proceso genera SELECT de 700+ IDs
- [ ] Confirmar que `interval_type=seconds` NO funciona en Odoo 19 (ya verificado, se mantiene minutes)
- [ ] Evaluar migración a Ringover Business (análisis ya hecho, pendiente decisión del cliente)

---

## RESUELTO (Referencia)

- [x] One-shots CallPicker acumulaban 1,200+ crons zombies — eliminados + batch search
- [x] Envío accidental de reporte completo — tab oculto, generación neutralizada
- [x] Emisor OdooBot → "Reporte Turbo Days"
- [x] Quitar acceso de gerente al menú Reportes
- [x] Fix V5 Gestionado OK (ya no pide vehículo/fecha de cita)
- [x] Fix rechazado con llamadas Ocupado/Número Equivocado
- [x] Dark mode en dashboard
- [x] Temperatura estrellas (0-5) en vez de caliente/tibio/frio
- [x] Dashboard gerente de agencia completo
- [x] Wizard referidos desde CC
- [x] Reportes programados configurables (3 canales)
- [x] Excel visual con 3 tablas (Leads + Walk-ins + Referidos)
- [x] Botón inteligente Dashboard filtrado por campaña

---

## ERRORES COMUNES A EVITAR

1. `String()` en templates OWL — usar `'' + value`
2. `options="{'key': true}"` en XML — `true` minúscula se interpreta como domain
3. Variable usada antes de definirse
4. `_sql_constraints` — Odoo 19 usa `models.Constraint`
5. `<i>` sin `title` en vistas XML
6. `openpyxl` genera inline strings — usar `xlsxwriter`
7. Nombre de hoja Excel > 31 chars
8. Campos que no existen — SIEMPRE verificar con grep
9. `this.env.services.user.userId` — no disponible sin importar el servicio
10. Formatos xlsxwriter en scope incorrecto
11. `lead_origin != "walk_in"` olvida referidos — usar `not in ("walk_in", "referral")`
12. **NUNCA iniciar git en la carpeta del CRM** — el repo COMUNICACION está en `C:\Users\Usuario\COMUNICACION\`, separado del CRM en `C:\Users\Usuario\Documents\ESTANCIA IT GREE\CRM ZMotors\`

---

## REGLAS DE TRABAJO

- Rama de desarrollo: `marketin`
- Un solo push por vez (evitar builds simultáneos en odoo.sh)
- Probar en marketin → testear → merge a producción
- Verificar crons después de cada merge (el upgrade puede resetear nextcall)
- Ejecutar `verificar_fix_oneshot_v3.txt` después de cada deploy
- NUNCA pushear directo a producción sin probar en marketin
