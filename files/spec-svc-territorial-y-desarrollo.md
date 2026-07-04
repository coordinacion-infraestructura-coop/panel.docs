# Spec: svc-territorial — Secretaría de Planificación y Articulación Territorial

**Titular**: Gabriel Fizza  
**Estado**: draft | **Versión**: 0.1.0

---

## 1. Propósito

Gestión del Programa de Fortalecimiento para Cooperativas. Registra cooperativas participantes, actividades de fortalecimiento realizadas (capacitaciones, asistencias técnicas, acompañamiento), y seguimiento de indicadores de impacto.

## 2. Módulos internos

| Módulo | Descripción |
|--------|-------------|
| `cooperativas` | Registro de cooperativas en el programa |
| `actividades` | Capacitaciones, asistencias técnicas, acompañamiento |
| `participaciones` | Registro de qué cooperativas participaron en qué actividades |

---

## 3. Modelos de datos

### `ter_cooperativas`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `cuit` | VARCHAR(13) | UNIQUE |
| `razon_social` | VARCHAR(200) | |
| `matricula_inaes` | VARCHAR(50) | |
| `tipo` | ENUM | `TRABAJO/SERVICIOS/CONSUMO/VIVIENDA/AGROPECUARIA/CREDITO` |
| `localidad` | VARCHAR(100) | |
| `departamento` | VARCHAR(100) | |
| `socios_activos` | INTEGER | |
| `fecha_constitucion` | DATE | NULL |
| `estado_programa` | ENUM | `ACTIVA/SUSPENDIDA/EGRESADA` |
| `fecha_ingreso_programa` | DATE | |
| `referente_nombre` | VARCHAR(200) | |
| `referente_email` | VARCHAR(200) | |
| `referente_telefono` | VARCHAR(20) | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | NULL |

### `ter_actividades`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `tipo` | ENUM | `CAPACITACION/ASISTENCIA_TECNICA/ACOMPAÑAMIENTO/TALLER/OTRO` |
| `titulo` | VARCHAR(300) | |
| `descripcion` | TEXT | |
| `fecha` | DATE | |
| `duracion_horas` | NUMERIC(5,1) | |
| `modalidad` | ENUM | `PRESENCIAL/VIRTUAL/HIBRIDA` |
| `lugar` | VARCHAR(300) | |
| `facilitador` | VARCHAR(200) | |
| `created_at` | TIMESTAMPTZ | |
| `created_by` | VARCHAR(100) | |

### `ter_participaciones`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `actividad_id` | UUID | FK |
| `cooperativa_id` | UUID | FK |
| `asistentes_count` | INTEGER | DEFAULT 1 |
| `observaciones` | TEXT | |
| `created_at` | TIMESTAMPTZ | |

---

## 4. Endpoints

```
GET    /api/v1/territorial/cooperativas
POST   /api/v1/territorial/cooperativas
GET    /api/v1/territorial/cooperativas/{id}
PATCH  /api/v1/territorial/cooperativas/{id}
GET    /api/v1/territorial/cooperativas/{id}/actividades

GET    /api/v1/territorial/actividades
POST   /api/v1/territorial/actividades
GET    /api/v1/territorial/actividades/{id}
POST   /api/v1/territorial/actividades/{id}/participaciones
GET    /api/v1/territorial/actividades/{id}/participaciones

GET    /api/v1/territorial/estadisticas
```

---

## 5. Criterios de aceptación

- [ ] Registro de cooperativas en el programa con datos de contacto
- [ ] Gestión de actividades de fortalecimiento
- [ ] Registro de participaciones (qué cooperativas en qué actividad)
- [ ] Estadísticas: cooperativas activas, actividades por tipo, alcance territorial
- [ ] Tests con cobertura > 80%


---
---


# Spec: svc-desarrollo — Secretaría de Desarrollo

**Titular**: Domingo Benso  
**Estado**: draft | **Versión**: 0.1.0

---

## 1. Propósito

Este servicio es un **adaptador** del sistema de Gestión de Cooperativas desarrollado por la UTN. No reemplaza al sistema externo: lo envuelve con una capa de integración que normaliza su API, autentica los requests con los tokens del ministerio, y republica los eventos relevantes al bus de Pub/Sub para el dashboard ejecutivo.

## 2. Arquitectura de integración

```
[Portal React]
      ↓
[API Gateway ministerial]
      ↓
[svc-desarrollo — FastAPI adapter]
      ↓ HTTP/REST
[Sistema UTN — API externa]
      ↓
[Respuesta normalizada al portal]
      + evento → Pub/Sub → BigQuery
```

## 3. Módulos internos

| Módulo | Descripción |
|--------|-------------|
| `proxy` | Forwarding de operaciones CRUD hacia UTN |
| `sync` | Sincronización periódica de datos UTN → BigQuery |
| `health` | Monitoreo del estado de la API UTN |

---

## 4. Endpoints expuestos (adapter)

Estos endpoints reciben requests del portal y los transforman al formato de la API UTN:

```
GET    /api/v1/desarrollo/cooperativas
GET    /api/v1/desarrollo/cooperativas/{id}
GET    /api/v1/desarrollo/cooperativas/buscar?cuit={cuit}
GET    /api/v1/desarrollo/cooperativas/{id}/socios
GET    /api/v1/desarrollo/cooperativas/{id}/actas
GET    /api/v1/desarrollo/estadisticas

GET    /api/v1/desarrollo/utn/health     → estado del sistema UTN
GET    /api/v1/desarrollo/utn/sync       → forzar sincronización manual
```

---

## 5. Configuración requerida (a completar)

> **PENDIENTE**: Obtener de UTN:
> - URL base de la API UTN
> - Mecanismo de autenticación (API key / OAuth / basic auth)
> - Formato de respuesta y errores
> - Endpoints disponibles y sus contratos
> - SLA y límites de rate

Estos datos deben documentarse en `docs/context/utn-api-contract.md` antes de implementar este servicio.

---

## 6. Manejo de errores del sistema externo

El adapter debe:
1. Devolver `503 Service Unavailable` si UTN no responde en 5 segundos
2. Incluir en el error el campo `"source": "utn"` para distinguirlo de errores propios
3. Reintentar automáticamente hasta 2 veces con backoff exponencial
4. Loguear todos los errores de UTN con nivel ERROR en Cloud Logging

---

## 7. Sincronización con BigQuery

Un Cloud Scheduler job (cada 6 horas) llama a `/api/v1/desarrollo/utn/sync` (endpoint interno protegido por service account). Este endpoint:
1. Consulta el listado completo de cooperativas desde UTN
2. Publica un evento `desarrollo.cooperativas.sincronizadas` a Pub/Sub
3. Un Dataflow job consume ese evento y actualiza la tabla `desarrollo_cooperativas` en BigQuery

---

## 8. Criterios de aceptación

- [ ] Documentar el contrato de la API UTN en `utn-api-contract.md`
- [ ] Adapter funcional para los endpoints de lectura
- [ ] Timeout y retry configurados
- [ ] Health check de UTN disponible
- [ ] Sincronización programada funcional
- [ ] Tests con mocks de la API UTN (no llamadas reales en CI)
- [ ] Cobertura > 70% (menor por ser un adapter)
