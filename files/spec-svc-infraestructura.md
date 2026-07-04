# Spec: svc-infraestructura — Secretaría de Gestión y Vinculación de Infraestructura

**Titular**: Luis Molinari  
**Estado**: draft | **Versión**: 0.1.0

---

## 1. Propósito

Gestión de proyectos de infraestructura eléctrica y de agua y saneamiento ejecutados o supervisados por la secretaría. Permite registrar proyectos, localidades beneficiadas, estado de avance y presupuesto.

## 2. Módulos internos

| Módulo | Descripción |
|--------|-------------|
| `electrica` | Proyectos de infraestructura eléctrica |
| `agua` | Proyectos de agua y saneamiento |
| `localidades` | Catálogo de localidades y departamentos |

---

## 3. Modelos de datos

### `inf_proyectos`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `codigo` | VARCHAR(30) | INF-YYYY-NNNNNN |
| `tipo` | ENUM | `ELECTRICA / AGUA_SANEAMIENTO` |
| `nombre` | VARCHAR(300) | |
| `descripcion` | TEXT | |
| `localidad` | VARCHAR(100) | |
| `departamento` | VARCHAR(100) | |
| `estado` | ENUM | ver abajo |
| `presupuesto_estimado` | NUMERIC(16,2) | |
| `presupuesto_ejecutado` | NUMERIC(16,2) | DEFAULT 0 |
| `fecha_inicio_estimada` | DATE | NULL |
| `fecha_inicio_real` | DATE | NULL |
| `fecha_fin_estimada` | DATE | NULL |
| `fecha_fin_real` | DATE | NULL |
| `poblacion_beneficiada` | INTEGER | NULL |
| `hogares_beneficiados` | INTEGER | NULL |
| `observaciones` | TEXT | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | NULL |
| `created_by` | VARCHAR(100) | |
| `updated_by` | VARCHAR(100) | |

Estados proyecto:
```python
class EstadoProyecto(str, Enum):
    IDENTIFICADO = "IDENTIFICADO"
    EN_FORMULACION = "EN_FORMULACION"
    APROBADO = "APROBADO"
    LICITADO = "LICITADO"
    EN_EJECUCION = "EN_EJECUCION"
    FINALIZADO = "FINALIZADO"
    SUSPENDIDO = "SUSPENDIDO"
```

### `inf_avances`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `proyecto_id` | UUID | FK |
| `fecha_informe` | DATE | |
| `porcentaje_avance` | NUMERIC(5,2) | 0..100 |
| `descripcion_avance` | TEXT | |
| `monto_certificado` | NUMERIC(16,2) | |
| `created_by` | VARCHAR(100) | |
| `created_at` | TIMESTAMPTZ | |

---

## 4. Endpoints

```
GET    /api/v1/infraestructura/proyectos
POST   /api/v1/infraestructura/proyectos
GET    /api/v1/infraestructura/proyectos/{id}
PATCH  /api/v1/infraestructura/proyectos/{id}
POST   /api/v1/infraestructura/proyectos/{id}/transicion
POST   /api/v1/infraestructura/proyectos/{id}/avances
GET    /api/v1/infraestructura/proyectos/{id}/avances
GET    /api/v1/infraestructura/proyectos/por-tipo/{tipo}
GET    /api/v1/infraestructura/proyectos/estadisticas
```

---

## 5. Eventos Pub/Sub

| Event type | Disparador |
|-----------|------------|
| `infraestructura.proyecto.creado` | POST proyecto |
| `infraestructura.proyecto.estado_cambiado` | POST transicion |
| `infraestructura.proyecto.avance_registrado` | POST avance |

---

## 6. Criterios de aceptación

- [ ] CRUD de proyectos con distinción por tipo (eléctrica vs agua)
- [ ] Registro de avances con porcentaje y monto certificado
- [ ] Filtros por tipo, estado, departamento, localidad
- [ ] Estadísticas: proyectos activos, inversión total, población beneficiada
- [ ] Tests unitarios cobertura > 80%
