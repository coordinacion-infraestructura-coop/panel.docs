# Spec: Estado Técnico de CC / CH / ML tomado del Checklist Técnico

**Estado**: approved
**Versión**: 1.0.0
**Servicio**: `svc-vivienda` (módulos `checklist_tecnico`, `cordon_cuneta`, `cordoba_hogar`, `mi_lugar`, `resumen_territorial`) + frontend `vivienda`
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-07

### Changelog
- **1.0.0 (2026-10-07)** — aprobado por el usuario sin cambios sobre la 0.1.0.
- **0.1.0 (2026-10-07)** — versión inicial, en revisión.

---

## 0. Origen

Hoy el "Estado Técnico" de cada fila de los paneles Cordón Cuneta, Córdoba Hogar y Mi Lugar lo
carga a mano el usuario del panel, eligiendo un valor del catálogo propio del programa
(`viv_cc_estados` / `viv_ch_estados` / `viv_ml_estados`). En paralelo, el área técnica de la DGV
mantiene el "Estado del expediente" de la misma localidad y programa en el Checklist Técnico
(`spec-checklist-tecnico-dgv.md`). Son dos datos sobre lo mismo, con catálogos distintos, que se
desfasan.

Pedido (2026-10-07): que el Estado Técnico de los tres paneles sea el que carga el área técnica
en su panel, con los mismos estados, y que se actualice solo.

## 1. Decisiones confirmadas (usuario, 2026-10-07)

| # | Decisión |
|---|---|
| D1 | El Estado Técnico de los paneles es el **"Estado del expediente"** del checklist (catálogo `viv_checklist_estado_expediente`, 9 valores, compartido por los 3 programas). No el "Estado de la documentación" por ítem. |
| D2 | En los paneles CC/CH/ML el Estado Técnico pasa a ser **de solo lectura**. Se cambia únicamente desde el Checklist Técnico. |
| D3 | Los valores de Estado Técnico cargados hoy en los paneles **se reemplazan** por lo que diga el checklist. Donde el checklist no tiene estado cargado, el panel muestra vacío ("—"). No se mapean estados viejos a nuevos. |
| D4 | En el % de avance, el Técnico aporta según su **posición en la ruta del checklist**; los estados de excepción (`en_ruta = false`) aportan 0. |

## 2. Mecanismo: lectura derivada, sin copia

El Estado Técnico **no se copia** a las tablas de los paneles. Los paneles lo leen de
`viv_checklist_tecnico.estado_expediente_id`, por el par `(programa, entidad_id)` que ya vincula
cada checklist con su fila de `viv_cordon_cuneta` / `viv_cordoba_hogar` / `viv_ml_proyectos`.

Consecuencias buscadas:
- Una sola fuente de verdad: no hay sincronización que pueda fallar ni desfase posible.
- Si un Admin renombra, reordena o agrega un estado en el catálogo del checklist, los tres
  paneles lo reflejan sin más cambios.
- **Sin migración de base y sin cambios de gateway.**

Las columnas `etecnico` de las tres tablas **no se eliminan ni se modifican**: quedan congeladas
con su último valor y dejan de leerse y de escribirse. Esto conserva los datos previos (D3) y
hace que la vuelta atrás sea solo de código.

## 3. Backend

### 3.1 `checklist_tecnico/estado_tecnico.py`
Módulo nuevo, de solo lectura. Va aparte de `service.py` porque ese archivo importa los
services de los 3 paneles, y los paneles necesitan importar esto (evita el ciclo):

```python
async def estados_expediente_por_entidad(db, programa: str) -> dict[str, int | None]
    # {entidad_id: estado_expediente_id} para todas las filas de viv_checklist_tecnico del programa
```

No crea filas de checklist (las entidades sin checklist simplemente no aparecen → `None`).

### 3.2 Lectura en los paneles
En las respuestas de `cordon_cuneta` (`get_full`, y las que devuelven un `MunicipioResponse`),
`cordoba_hogar` (ídem) y `mi_lugar` (`listar_proyectos_ml`, `obtener_proyecto_ml`, crear y
actualizar), el campo **`etecnico` se completa con el `estado_expediente_id` del checklist** en
vez de con la columna propia. El nombre del campo no cambia; cambia el catálogo al que refiere
su id (antes el del programa, ahora `viv_checklist_estado_expediente`).

### 3.3 Escritura en los paneles
- `etecnico` se quita de los schemas de alta y edición de los tres módulos
  (`MunicipioCreate`/`MunicipioUpdate`, sus equivalentes de CH, `ProyectoMLCreate`/`ProyectoMLUpdate`).
  Si un cliente lo envía, se ignora.
- El bucle que escribe `viv_*_estado_historial` deja de considerar `etecnico` (quedan
  `ejuridico` y `efinanciero`).
- `mi_lugar.listar_proyectos_ml`: el filtro `estado_id` deja de comparar contra `etecnico`.

### 3.4 Historial
`GET .../historial` de cada panel agrega, a las filas propias de `viv_*_estado_historial`, las
de `viv_checklist_estado_hist` del checklist de esa entidad, con:

- `campo = "etecnico_checklist"`
- `estado_nuevo_id` = el del registro; `estado_anterior_id` = el del registro inmediatamente
  anterior del mismo checklist (o `null` en el primero)
- `created_at` / `created_by` del registro

Orden final: `created_at` descendente. Las filas viejas con `campo = "etecnico"` se siguen
devolviendo tal cual (son historia real, con ids del catálogo viejo del programa).

### 3.5 `resumen_territorial`
`_estado_fields`: `subestados.tecnico` pasa a ser el label del estado del expediente del
checklist. Se refleja a partir del siguiente recálculo del snapshot.

### 3.6 Lo que no cambia
- Catálogos `viv_*_estados` y su flag `aplica_tecnico` (queda sin efecto; no se borra).
- Permisos: `GET /checklist-tecnico/catalogos` ya admite a todos los roles que leen los paneles
  (`ROLES_LECTURA`), así que no hace falta endpoint ni rol nuevo. `TecnicoDGV` sigue sin acceso
  a los paneles completos.
- `estado_general` sigue 100% manual.
- `informes/` no usa el Estado Técnico; no se toca.

## 4. Frontend (`CordonCunetaPage.tsx`, `CordobaHogarPage.tsx`, `MiLugarPage.tsx`)

Los tres paneles piden además `GET /checklist-tecnico/catalogos` y usan `estados_expediente`
para todo lo que refiere al Técnico:

1. **Columna y badge "Estado Técnico"**: label del catálogo del checklist. El catálogo del
   checklist no tiene color, así que el badge usa un estilo neutro único.
2. **Filtro "Estado Técnico"**: opciones = estados activos del catálogo del checklist.
3. **Orden por columna y export a Excel**: por el `orden` / label del catálogo del checklist.
4. **`EditModal`**: el Técnico se muestra como dato de solo lectura con la leyenda
   "Se actualiza desde el Checklist Técnico". **`AgregarModal`**: se quita el selector de Técnico.
5. **Historial**: las entradas `etecnico_checklist` se rotulan "Técnico" y resuelven sus labels
   en el catálogo del checklist; las `etecnico` viejas se rotulan "Técnico (anterior)" y siguen
   resolviendo en el catálogo del programa.
6. **Admin de estados del panel**: se quita el check "aplica a Técnico".
7. **% de avance** (D4):
   `avance = (posJ/maxPos + posF/maxPos + rT) / 3`, donde `posJ`, `posF` y `maxPos` son los de
   hoy (posición en el catálogo del programa) y
   `rT = índice del estado en la ruta / (largo de la ruta − 1)`, con
   ruta = estados `en_ruta = true` del checklist ordenados por `orden`.
   Sin estado o estado fuera de ruta → `rT = 0`. Con el catálogo actual: `A ESPERA de
   DOC.TÉCNICA` = 0 % … `OBRA TERMINADA` = 100 %.
8. **`ChecklistTecnicoPage.tsx`**: al cambiar el estado del expediente se invalidan las queries
   de los tres paneles, para que el cambio se vea sin recargar.
9. **`resumen-territorial/fichaMunicipio.ts`**: el "Estado Técnico" de Mi Lugar se resuelve
   contra el catálogo del checklist.

## 5. Efectos visibles que se aceptan

- El día del deploy, toda fila cuyo checklist no tenga "Estado del expediente" cargado pasa a
  mostrar Estado Técnico vacío, aunque antes tuviera uno.
- El % de avance de todas las filas cambia (el Técnico se mide contra otra escala).
- Los cambios de Estado Técnico no llevan `fecha_cambio` retroactiva: quedan con la fecha real
  en que el área técnica los hizo.
- Los estados del catálogo de un programa referenciados por la columna `etecnico` congelada
  siguen sin poder eliminarse desde el admin de estados ("estado en uso").
- `informes/evolucion_temporal` cuenta cambios de `viv_*_estado_historial`; los cambios técnicos
  nuevos no viven ahí, así que dejan de sumar en esa serie (mismo vacío conocido que ya tiene
  `estado_general`).

## 6. Fuera de alcance

- Borrar o anular las columnas `etecnico` y limpiar los estados "solo técnicos" de los catálogos
  de programa.
- Dar color a los estados del expediente.
- Vincular Estado Jurídico o Presupuestario con otros paneles.

## 7. Pruebas

Backend (`pytest`):
- El panel devuelve en `etecnico` el estado del expediente del checklist; `None` si no hay
  checklist o no tiene estado. Para CC, CH y ML.
- `PATCH` del estado en el checklist → la lectura siguiente del panel lo refleja.
- `PATCH` / `POST` del panel con `etecnico` en el body no cambia nada ni escribe historial.
- El historial del panel incluye las entradas `etecnico_checklist` con el anterior bien derivado.
- `resumen_territorial` usa el label del checklist en `subestados.tecnico`.

Frontend: `npm run build` + verificación manual en el navegador de los tres paneles.

## 8. Deploy

Backend (`svc-vivienda`) y frontend en el mismo momento: un frontend viejo contra el backend
nuevo resolvería el id de `etecnico` en el catálogo equivocado. Sin migración Alembic, sin config
nueva de gateway. Vuelta atrás: revertir ambos deploys; los datos previos siguen en la columna.
