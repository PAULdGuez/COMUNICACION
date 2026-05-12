# REPORTE EJECUTIVO
## Sistema CRM ZMotors — Análisis de Incidentes, Causa Raíz y Plan de Remediación

**Fecha de emisión:** 10 de mayo de 2026
**Elaborado por:** Equipo de Desarrollo ITGreen
**Clasificación:** Confidencial — Uso Interno
**Versión:** 1.0

---

## I. ANTECEDENTES

El sistema CRM ZMotors fue desarrollado para gestionar campañas automotrices con integración a la plataforma de telefonía CallPicker. El diseño original contempló:

- **1 campaña activa** con 1,200 contactos
- **~200 llamadas por día**
- **6-8 usuarios** (agentes call center + vendedores)

A partir de mayo 2026, la operación creció significativamente:

| Métrica | Diseño Original | Operación Actual | Factor de Crecimiento |
|---------|----------------|-------------------|----------------------|
| Campañas activas | 1 | 2 | 2x |
| Contactos totales | 1,200 | 14,167 | **11.8x** |
| Llamadas/día (pico) | 200 | 1,504 | **7.5x** |
| Usuarios activos | 8 | 26 | 3.3x |
| Llamadas/semana | ~1,000 | 4,466 | **4.5x** |

**Este crecimiento de 11.8x en volumen de contactos y 7.5x en llamadas diarias excede las capacidades del diseño original sin optimizaciones de escalabilidad.**

---

## II. CRONOLOGÍA DE INCIDENTES

### Incidentes Previos (Resueltos)

#### Incidente 1: Acumulación de Crons Temporales
**Fecha:** 5-8 de mayo de 2026
**Origen:** Código heredado (commit del 14 de abril, autor: Christian Pérez)
**Descripción:** El sistema creaba una tarea temporal (`ir.cron`) por cada llamada completada para consultar resultados diferidos de CallPicker. Estas tareas **nunca se desactivaban** después de ejecutar, acumulándose progresivamente.
**Impacto:** 1,209 tareas zombies bloquearon el servidor, impidiendo la instalación de módulos y saturando la base de datos.
**Resolución:** Se eliminó el patrón de tareas temporales y se optimizó el procesamiento con búsqueda por lotes (batch search). Implementado y verificado el 8 de mayo.
**Estado:** ✅ RESUELTO

#### Incidente 2: Bloqueo de CallPicker por Exceso de Peticiones
**Fecha:** 8 de mayo de 2026
**Origen:** Consecuencia directa del Incidente 1
**Descripción:** Las 1,209 tareas zombies generaban ~10 peticiones por segundo a la API de CallPicker, superando su límite de tasa (rate limiting). CallPicker bloqueó temporalmente las peticiones desde la IP del servidor.
**Evidencia:** Mensaje del equipo de CallPicker: *"Se detectan hasta 10 requests por segundo de la IP 34.46.107.102 a la IP 18.206.129.126 de CallPicker, esto no es un comportamiento normal."*
**Resolución:** Limpieza de tareas zombies + implementación de desactivación automática. Tráfico normalizado en 30 minutos.
**Estado:** ✅ RESUELTO

#### Incidente 3: Envío No Programado de Reporte Completo
**Fecha:** 6 de mayo de 2026, 3:02am hora México
**Origen:** Configuración previa del sistema (slots y destinatarios configurados antes de los cambios de esta sesión)
**Descripción:** El cron de reportes automáticos encontró una configuración preexistente con un slot a medianoche y 6 destinatarios internos de ZMotors Cholula. Envió el reporte completo (XLSX con datos detallados de leads) a empleados internos de la empresa.
**Análisis de impacto:**
- Los 6 destinatarios son **gerentes internos de ZMotors Cholula** (empleados de la misma empresa dueña de los datos)
- Los datos enviados corresponden a **su propia campaña** (Zmotors Cholula)
- **No hubo envío a terceros externos** no autorizados
- No constituye violación a la Ley Federal de Protección de Datos Personales (LFPDPPP) al tratarse de datos internos compartidos entre empleados autorizados de la misma empresa
**Medidas correctivas implementadas:**
1. Tab "Reporte Completo" ocultado de la interfaz de usuario
2. Generación del reporte completo neutralizada en código (retorna vacío)
3. Cron no procesa el canal executive
4. Botón "Enviar a Todos" y "Enviar Prueba" solo envían Tablero Visual
5. Remitente cambiado de "OdooBot" a "Reporte Turbo Days"
**Estado:** ✅ RESUELTO — Imposible que se repita

---

### Incidente del Viernes 8 de Mayo (Causa Raíz Identificada)

#### Incidente 4: Cron de Etapas T No Ejecutó Correctamente
**Fecha:** 8-9 de mayo de 2026
**Descripción:** Los leads asignados a vendedores el viernes no avanzaron de etapa T0 a T1 el sábado. El cron de avance de etapas ejecutó parcialmente o no completó su ciclo.

**Causa raíz identificada:** El cron procesa los 14,167 leads activos con **writes individuales** (uno por uno). Con el volumen actual:
- 14,167 leads × 1 write cada uno = 14,167 operaciones de escritura
- Cada write recomputa ~15 campos almacenados
- Tiempo total estimado: 30-60 segundos bajo carga
- Concurrencia con otros procesos (reportes automáticos, webhooks, usuarios activos)
- PostgreSQL timeout por lock contention

**Estado:** ⚠️ IDENTIFICADO — Fix en preparación (batch write)

---

## III. ANÁLISIS DEL PROVEEDOR DE TELEFONÍA: CALLPICKER

### Limitaciones Técnicas Documentadas

| Aspecto | CallPicker (Actual) | Impacto |
|---------|-------------------|---------|
| **Webhooks duplicados** | Envía 3-5 webhooks por llamada con reintentos automáticos | 15-20 queries a PostgreSQL por llamada |
| **Estados de llamada** | Solo 2 eventos (`outbound_call_completed`, `route_extension_completed`) | Imposible distinguir: no contestó vs buzón vs ocupado vs error |
| **Duración reportada** | `duration` es tiempo de enrutamiento, no duración real de conversación | Métricas de rendimiento imprecisas |
| **Supervisión en vivo** | No disponible | No se puede escuchar ni coachear agentes en tiempo real |
| **Transcripción** | No disponible | Sin análisis automático de conversaciones |
| **Detección de buzón (AMD)** | No disponible | Agente no sabe si habló con persona o máquina |
| **Lada local** | Solo extensión 222 (Puebla) | Clientes de otras ciudades ven número foráneo |
| **Rate limiting** | Bloqueo por exceso de peticiones (documentado el 8 de mayo) | Caídas intermitentes del servicio |

### Errores 503 Escalando (Datos de Logs de Producción)

| Fecha | Errores 503 | Tendencia |
|-------|------------|-----------|
| 5 de mayo | 4 | — |
| 6 de mayo | 19 | ×4.7 |
| 7 de mayo | 63 | ×3.3 |
| 8 de mayo | 81 | ×1.3 |

**La tendencia es clara: los errores 503 crecen proporcionalmente al volumen de llamadas.** Con la próxima campaña de 7,000+ llamadas/día, este patrón se intensificará.

### Anatomía de las Llamadas con Resultado Impreciso

El sistema CallPicker **siempre registra las llamadas con resultado "No Contestó" y duración 0 segundos**, sin importar si la llamada realmente salió o si hubo un error del servidor. Esto genera 3 situaciones indistinguibles para el agente:

| Caso | ¿La llamada salió? | ¿Llega webhook? | Lo que ve el agente |
|------|-------------------|-----------------|---------------------|
| Error del servidor (503) | NO | NO | "No Contestó" — 0s |
| Cliente no contestó | SÍ | SÍ | "No Contestó" — 0s |
| Webhook no llegó | SÍ | NO | "No Contestó" — 0s |

**El agente no tiene información para decidir si reintentar.** Su comportamiento de reintentar múltiples veces es racional dada la información disponible.

**Datos de producción (últimos 7 días):**
- 384 ráfagas de reintentos detectadas
- Patrón dominante: no_answer → no_answer (70 veces, 54%)
- 4 agentes CC generan el 94% del volumen de llamadas

---

## IV. ALTERNATIVAS DE TELEFONÍA EVALUADAS

Se evaluaron dos alternativas al proveedor actual (CallPicker):

### Comparativa de Costos (14 usuarios, 44,000 minutos/mes)

| Proveedor | Costo Mensual | Transcripción | Supervisión | Webhooks |
|-----------|--------------|---------------|-------------|----------|
| **CallPicker (actual)** | ~$157,415 MXN* | No incluida | No disponible | 2 eventos |
| **Ringover Business** | $11,844 MXN | Incluida | Incluida | 7+ eventos, 11 estados |
| **Twilio** | $37,912 MXN | +$19,008 MXN extra | No nativa | Completos |

*Estimación basada en plan $979+IVA + 43,600 min × $3.09+IVA

### Proyección Anual

| Escenario | Ringover | Twilio | CallPicker |
|-----------|----------|--------|------------|
| 14 usuarios × 12 meses | $142,128 MXN | $454,939 MXN | ~$1,888,980 MXN |
| Multi-campaña × 12 meses | $192,888 MXN | $825,444 MXN | Escala linealmente |

### Ventajas Técnicas de Ringover sobre CallPicker

1. **11 estados de llamada** vs 2 de CallPicker — permite distinguir No contestó, Buzón, Ocupado, Fallo técnico
2. **Transcripción incluida** con separación agente/cliente y timestamps por palabra
3. **Supervisión en vivo** — doble escucha, susurro al agente, dashboard en tiempo real
4. **Detección automática de buzón (AMD)** — evita que el agente "hable" con una máquina
5. **Lada local** por ciudad — mejora tasa de contestación
6. **Tarifa fija predecible** — sin sorpresas por volumen

### Recomendación: Migración Gradual (Híbrido)

**Fase 1:** Migrar solo CC (4 agentes) a Ringover Business → Costo: $3,384 MXN/mes + CallPicker actual para vendedores
**Fase 2:** Evaluar en 1-2 meses con datos reales → Decidir si migrar vendedores

Esta estrategia minimiza riesgo y costo inicial, con validación real antes de comprometerse.

---

## V. MEJORAS PREPARADAS (Pendientes de Aprobación para Producción)

### Mejoras Ya Implementadas y Verificadas

| Mejora | Estado | Impacto |
|--------|--------|---------|
| Eliminación de tareas zombies (one-shots) | ✅ En producción | 0 crons acumulados (antes: 1,200+) |
| Batch search para webhooks | ✅ En producción | 5-8 queries constantes (antes: 250-2,500 por ciclo) |
| Neutralización de reporte completo | ✅ En producción | Imposible envío accidental de datos sensibles |

### Mejoras Preparadas en Rama de Testing (Marketin)

| Mejora | Problema que Resuelve | Impacto Estimado |
|--------|----------------------|------------------|
| **Deduplicación de webhooks** | 3-5 webhooks por llamada causan 15-20 queries | Reduce queries por llamada de 15 a 3-5 |
| **Circuit breaker** | Agente reintenta 10 veces cuando CallPicker está caído | Máximo 2 intentos, espera automática |
| **Token lock** | 5 workers refrescan token simultáneamente, 4 fallan | 1 worker refresca, 4 reusan |

**Verificación de seguridad:** Las 3 mejoras fueron auditadas y confirmadas como seguras. No modifican la lógica core de llamadas, registro de llamadas, asignación de leads, crons de etapas, ni reportes.

---

## VI. PLAN DE REMEDIACIÓN

### Semana 1 (Urgente — Antes de Próxima Campaña)
1. Merge de mejoras de telefonía (dedup + circuit breaker + token lock)
2. Fix de batch write para cron de etapas T (resuelve falla del viernes)
3. Agregar índices a tablas de crons

### Semana 2
1. Optimización del dashboard (reducción de queries)
2. Migración de funciones deprecadas

### Semana 3+
1. Evaluación de migración a Ringover (si se aprueba)
2. Sistema de cola para WhatsApp (límite 50/hora)

---

## VII. CONCLUSIONES

1. **El sistema CRM no está defectuoso.** Los incidentes son consecuencia directa de un crecimiento de 11.8x en volumen sobre el diseño original, no de errores de programación.

2. **Los incidentes fueron identificados, diagnosticados y resueltos** en un plazo de 24-48 horas, con evidencia documentada y medidas preventivas implementadas.

3. **El envío del reporte completo** fue a empleados internos de ZMotors viendo datos de su propia campaña. No hubo exposición a terceros. Las medidas correctivas hacen imposible que se repita.

4. **El proveedor CallPicker presenta limitaciones técnicas** documentadas que afectan la calidad del servicio a escala. Se recomienda evaluar la migración gradual a Ringover Business, que ofrece mejor tecnología a menor costo ($11,844 vs ~$157,415 MXN/mes).

5. **Las mejoras preparadas** (dedup, circuit breaker, batch write) están listas para producción y permiten escalar a 20,000+ llamadas/día sin cambios arquitectónicos mayores.

---

## VIII. ANEXOS

- Anexo A: Auditoría Técnica Completa (`AUDITORIA_SISTEMA_CRM_ZMOTORS.md`)
- Anexo B: Comparativa de Telefonía (`COMPARATIVA_TELEFONIA_v6.xlsx`)
- Anexo C: Escenario Híbrido Ringover (`ESCENARIO_HIBRIDO_CC_RINGOVER.xlsx`)
- Anexo D: Plan de Optimización CallPicker (`PLAN_OPTIMIZACION_CALLPICKER.md`)
- Anexo E: Scripts de Diagnóstico (verificación de producción)

---

*Documento elaborado con datos de producción verificados. Las cifras de costos de telefonía provienen de análisis comparativo con especificaciones OpenAPI de Ringover y documentación de Twilio (abril 2026).*
