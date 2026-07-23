# Auditoría de código — GestorCooperativo

Registro vivo de auditorías de **calidad de código y bugs** (no de infraestructura/deploy — para eso ver `docs/deploy*.md` y el skill `deploy-servicio`).

**Regla para cualquier agente que vaya a auditar este código**: leer este documento completo ANTES de empezar.
- La sección "Verificado y correcto" evita re-derivar análisis ya hecho.
- La sección "Pendiente" evita re-descubrir bugs ya conocidos — verificá si siguen vigentes (el código pudo cambiar) en vez de asumirlos ciegamente, pero no vuelvas a auditar desde cero lo que ya está documentado acá.
- **Importante**: los hallazgos de una auditoría automatizada (agentes en paralelo) pueden tener falsos positivos. Antes de aplicar un fix, releer el código real señalado — ver la sección "Verificado y correcto" de más abajo para dos ejemplos concretos de hallazgos descartados tras verificar.

**Cómo actualizar este archivo al terminar una auditoría o una corrección**:
1. Hallazgos nuevos → sección "Pendiente" correspondiente (fecha, severidad, archivo:línea, escenario de falla concreto).
2. Algo que se corrigió → moverlo a "✅ Corregido" (fecha, qué se cambió, en qué archivo, qué test lo cubre).
3. Algo que se decidió no arreglar → moverlo a "❌ Descartado" (por qué, quién decidió).
4. Código nuevo que se verificó como correcto (o un hallazgo que resultó ser falso positivo) → sumarlo a "Verificado y correcto".
5. Actualizar la sección "Última auditoría" (fecha, alcance completo o incremental, qué se cubrió).

---

## Última auditoría

- **Fecha**: 2026-07-23
- **Alcance**: completa — backend `services/svc-vivienda` (10 módulos, 18 migraciones) + frontend completo (~25 archivos) + `infra/gateway/openapi.yaml` vs routers reales. Mismo día: se aplicaron y verificaron los fixes de todos los hallazgos crítico/medio y la mayoría de los menores (ver "✅ Corregido").
- **Cómo se hizo**: 2 subagentes en paralelo (backend, frontend) para el hallazgo inicial; luego verificación manual línea por línea de cada hallazgo antes de corregirlo (esto descartó 2 falsos positivos y afinó el alcance real de otro, ver "Verificado y correcto").
- **Qué falta cubrir**: `services/svc-privada-adapter/` (solo README, sin código propio), scripts de `infra/*.sh` (no auditado su contenido).
- **Verificación de la corrección**: 160/160 tests backend (`pytest`) verdes, incluyendo 8 tests nuevos. `npm run build` del frontend verde (`tsc -b && vite build`, sin errores de tipos).

---

## 🔴 Pendiente — Crítico

*(vacío — el único hallazgo crítico de esta auditoría fue corregido, ver "✅ Corregido" #1)*

---

## 🟠 Pendiente — Medio

*(vacío por ahora — todos los hallazgos medios de esta auditoría fueron corregidos)*

---

## 🟡 Pendiente — Menor

### Asimetrías de permisos (decisión: no tocar, 2026-07-23)
- `require_comunicaciones_write()` permite crear pedidos a cualquier rol de `infraestructura`/`supervision` (incluido `Consulta`), pero `eliminar_pedido` exige `ROLES_ESCRITURA` estricto — un `Consulta` de infraestructura puede crear un pedido que después no puede borrar él mismo.
- `eliminar_estado`/`eliminar_municipio`/`eliminar_localidad` usan `ROLES_TRANSICION` (Admin, Supervisor) mientras `eliminar_beneficiario` usa `ROLES_ELIMINACION` (solo Admin) para una operación conceptualmente similar (soft delete).
- **Decisión del usuario (2026-07-23)**: dejarlo como está — es más restrictivo, no una fuga de seguridad, y cambiarlo alteraría permisos reales de usuarios en producción sin pedido explícito. Revisar solo si en el futuro se pide explícitamente unificar el modelo de permisos.

### Checklist técnico CC sin limpieza de filas obsoletas (seguimiento, 2026-07-23)
- `sync_from_sheet()` (`services/svc-vivienda/app/cordon_cuneta/checklist_sync.py:262-336`) solo hace upsert de filas presentes en el Sheet; si el área técnica borra una localidad del Sheet, el `ChecklistTecnicoCC` correspondiente queda huérfano indefinidamente, sin ninguna marca de "ya no está en el sheet". Una lectura vacía se loguea igual que "sin cambios", indistinguible de un vaciado accidental del Sheet.
- **Decisión del usuario (2026-07-23)**: no implementar la limpieza ahora (requiere decidir semántica con el área técnica primero: ¿marcar como obsoleto?, ¿alertar?, ¿borrar?). **Se deja documentado a propósito acá** para que si aparece una falla vinculada (ej. un municipio con checklist técnico visiblemente desactualizado) se identifique rápido como este gap conocido y no como un bug nuevo.

### `xlsx@0.18.5` con CVEs conocidas sin parche en npm (decisión: no tocar, 2026-07-23)
- `frontend/src/shared/utils/exportTable.ts` usa `xlsx@0.18.5`, la última versión publicada por SheetJS en el registry de npm; las versiones parcheadas de las CVEs conocidas (prototype pollution, ReDoS) solo están disponibles vía el tarball de `cdn.sheetjs.com`, no en npm.
- **Decisión del usuario (2026-07-23)**: dejarlo como está — el uso actual es solo de escritura (`json_to_sheet`/`writeFile`), nunca se parsean archivos de terceros, así que el riesgo práctico es bajo. Reconsiderar si en algún momento se agrega una función de *importar* un .xlsx subido por un usuario.

---

## Gaps de cobertura de tests conocidos

- ~~Visibilidad jerárquica de `listar_pedidos` por secretaría~~ y ~~asignación automática de `secretaria` en `crear_pedido`~~ — **cerrados el 2026-07-23** (ver "✅ Corregido", 15 tests nuevos en `test_cordon_cuneta.py`/`test_cordoba_hogar.py`).
- **No hay test de concurrencia real** (dos requests simultáneas de verdad) para las 3 correcciones de `IntegrityError` en creaciones (municipio/localidad/beneficiario) — se verificó el código manualmente y el retry de `numero_expediente` sí tiene test (con mock simulando la colisión), pero las otras tres no. Es estructuralmente difícil de reproducir de forma determinística contra SQLite in-memory de un solo hilo sin mockear; se decidió no forzarlo para no tener un test frágil.
- **El frontend no tiene ningún framework de test configurado** (`package.json` no tiene `vitest`/`jest`, solo `dev`/`build`/`preview`). Toda la verificación de cambios de frontend es vía `tsc -b && vite build` (chequeo de tipos) — no hay tests de comportamiento de componentes ni de lógica de UI (sort, filtros, extractErrorMessage, etc.). Es una decisión de alcance mayor (elegir framework, configurar, escribir la primera batería) — no se tocó en esta sesión; evaluar si se justifica cuando el frontend crezca más.

---

## ✅ Verificado y correcto (no re-auditar salvo que el código cambie)

- **Unicidad a nivel DB real** en municipios CC, localidades CH, beneficiarios (DNI), usuarios de portal (email PK), programas, asignaciones, checklist técnico CC — todas con constraint real de DB.
- **SAVEPOINT por fila en `checklist_sync.sync_from_sheet()`** — protegido y con test de regresión (`tests/test_cc_checklist_sync.py`).
- **JSONB vía `text()`** en `app/audit.py` — único uso en todo el código, correcto (`CAST(:payload AS jsonb)` + `json.dumps()`).
- **Máquina de estados de expedientes** — no se puede bypassear vía PATCH genérico, bien cubierta por tests.
- **Endpoint interno de sync** (`app/internal/router.py`) — sin `Depends(get_current_user)` por diseño, no declarado en el gateway, no expuesto por ningún camino no intencional.
- **Doble-submit por UI**: todos los botones de submit revisados deshabilitan correctamente con `disabled={mutation.isPending}`.
- **Sort y export de tablas CC/CH**: copian el array sin mutar el original, nulls al final, exportan la lista filtrada/ordenada.
- **Gateway vs routers**: ya no hay endpoints faltantes (ver Corregido #2).

### Dos falsos positivos descartados al verificar el código real (2026-07-23)
Estos dos hallazgos venían de la auditoría automatizada inicial y **resultaron ser incorrectos** al leer los endpoints de error reales — quedan acá documentados para que nadie los "corrija" de nuevo sin necesidad:
- **`AdminUsuariosPage.tsx` — extracción de error `err?.response?.data?.detail?.message`**: se había marcado como bug asumiendo que FastAPI típicamente no devuelve `detail` como objeto `{message}`. Pero se verificó `app/portal/router.py` completo: **todos** sus `HTTPException` (EMAIL_DUPLICADO, ROL_INVALIDO, AUTO_ELIMINACION, NOT_FOUND, PERMISO_INSUFICIENTE) usan consistentemente `detail={"code": ..., "message": ...}` — el código del frontend es correcto para este backend específico.
- **`deleteMut` de estados en `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx` (catálogo)**: mismo caso — `eliminar_estado` en ambos `service.py` solo puede fallar con `ESTADO_EN_USO`, siempre `detail={"code": ..., "message": ...}` (dict). La lectura de `detail?.message` en el frontend es correcta, no hace falta unificarla con `extractErrorMessage`.

### Corrección de alcance: el bug de fechas en seed muerto NO aplicaba a Cordón Cuneta
El hallazgo original decía que `seed_cordon_cuneta()` (`app/cordon_cuneta/service.py`) tenía el mismo riesgo de fechas sin `date.fromisoformat()` que `seed_cordoba_hogar()`. Al revisar el modelo `MunicipioCordonCuneta` (`app/cordon_cuneta/models.py`) se confirmó que **no tiene ninguna columna de fecha** — el riesgo solo existía en Córdoba Hogar (`fecha_anuncio`), que sí se corrigió (ver Corregido #7).

---

## Mejoras de arquitectura sugeridas (no bugs — evaluar bajo demanda, no implementar sin que se pida)

1. **CC y CH son ~95% código duplicado** (backend `service.py`, frontend `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx`). Causa raíz de casi todas las asimetrías de este documento. Un mixin/componente genérico parametrizado por tipo de entidad reduciría el riesgo de que un fix futuro se aplique a uno y no al otro.
2. **Centralizar `extractErrorMessage`** en `frontend/src/shared/utils/` — hoy vive duplicado (aunque correctamente) en CC y CH, y con variantes locales en `BeneficiarioFormPage`, `GestionesListPage`.
3. **Hook `useEntityMutation`** que envuelva `useMutation` + extracción de error + invalidación — evitaría tener que acordarse de agregar `onError` en cada mutación nueva.
4. **Helper común para capturar `IntegrityError`** en los `crear_*` con pre-check — hoy son 3 lugares con el mismo patrón (`crear_municipio`, `crear_localidad`, `crear_beneficiario`); un helper (`with catch_unique_violation(code=...): ...`) lo centralizaría.
5. **Patrón "catálogo de estados con `orden` + mínimo entre 3 dimensiones"** (incluyendo `_recompute_all_estado_general`, hoy duplicado en `cordon_cuneta/service.py` y `cordoba_hogar/service.py`) podría vivir en un módulo común (`app/shared/estado_general.py`).
6. **`numero_expediente`**: el fix de reintento acotado (5 intentos) resuelve el caso práctico, pero a mayor escala de concurrencia real valdría la pena reemplazarlo por un `SELECT ... FOR UPDATE` sobre una fila de contador dedicada o una sequence de Postgres por año (requeriría migración).

---

## ✅ Corregido

Todos los fixes de esta sección son del **2026-07-23**, aplicados en la misma sesión que la auditoría, con aprobación explícita del usuario ("corrijamos todo"). Verificados con la suite completa de tests backend (160/160 verdes, 8 nuevos) y build de frontend (`tsc -b && vite build` sin errores).

1. **[Crítico] `AdminUsuariosPage.tsx` sin guard de rol** → `frontend/src/shared/auth/ProtectedRoute.tsx` ahora acepta una prop opcional `roles?: string[]` (consulta `usePortalUser()` y redirige a `/` si el rol no matchea); `frontend/src/App.tsx` envuelve la ruta `admin/usuarios` con `<ProtectedRoute roles={['Admin']}>`.
2. **[Medio] Endpoints faltantes en el gateway** → agregados `GET /api/v1/vivienda/programas/{programa_id}` y `PATCH /api/v1/vivienda/asignaciones/{asignacion_id}` (con su `options:` de CORS) a `infra/gateway/openapi.yaml`. **Pendiente de infraestructura**: falta desplegar una nueva versión de config del gateway (`ministerio-config-v{FECHA}`) para que el cambio llegue a producción — el YAML del repo ya está actualizado pero el deploy en sí no se ejecutó en esta sesión.
3. **[Medio] Race condition (TOCTOU) en creaciones** → `crear_municipio` (`cordon_cuneta/service.py`), `crear_localidad` (`cordoba_hogar/service.py`) y `crear_beneficiario` (`beneficiarios/service.py`) ahora capturan `IntegrityError` en el `flush()`, hacen `rollback()` y devuelven el mismo 409 legible del pre-check en vez de un 500 genérico. Cubierto indirectamente por los nuevos tests de creación (`test_crear_municipio`, `test_crear_localidad`), aunque la race en sí no tiene test dedicado (ver gaps de cobertura).
4. **[Medio] `estado_general` no se recomputaba realmente en el flujo normal de edición (más grave de lo que parecía)** → se descubrió que el `EditModal` de CC/CH siempre mandaba `estado_general` en el payload (el form nunca es parcial), lo que hacía que el "recálculo automático" del backend (`if "estado_general" not in updates`) casi nunca se disparara en la práctica real de uso, solo en tests que mandan payloads parciales directo a la API. Fix en frontend (`CordonCunetaPage.tsx` y `CordobaHogarPage.tsx`): el `EditModal` solo incluye `estado_general` en el submit si el usuario tocó explícitamente ese desplegable (`estadoGeneralTouched`); si no, se omite del payload y el backend recalcula solo. Además, `actualizar_estado` (ambos `service.py`) ahora recomputa `estado_general` de todos los registros activos cuando cambia el `orden` de un estado del catálogo (`_recompute_all_estado_general`). Cubierto por `test_reordenar_estado_recomputa_estado_general_de_municipios`/`..._localidades` (nuevos).
5. **[Medio] Regresión de `extractErrorMessage` en `CordobaHogarPage.tsx`** → reordenado para leer `detail` string antes del fallback de 409 (igual que CC). Impacto real hoy es bajo (ningún endpoint de localidad devuelve 409 con detail string todavía) pero previene una regresión futura silenciosa.
6. **[Medio] Race condition en `numero_expediente`** → `crear_expediente` (`expedientes/service.py`) reintenta hasta 5 veces ante `IntegrityError` (recalculando el número cada vez) en vez de propagar un 500. Cubierto por `test_numero_expediente_colision_se_resuelve_con_reintento` (nuevo, simula la colisión con un mock).
7. **[Menor] Fecha sin convertir en seed muerto de Córdoba Hogar** → `seed_cordoba_hogar()` (`cordoba_hogar/service.py`) ahora convierte `fecha_anuncio` con `date.fromisoformat()` antes de insertar. (Corrección de alcance: el mismo hallazgo para Cordón Cuneta era un falso positivo — ver "Verificado y correcto".)
8. **[Menor] Mutaciones sin `onError` → fallos silenciosos** → agregado a: `updateMut`/`createMut` de estados en `GestionarEstadosModal`/`GestionarParametrosModal` (CC y CH), `deleteMutation` de pedidos en ambos `DetailPanel` (CC y CH), `deleteMutation` de gestiones en `GestionesListPage.tsx`. Cada uno ahora muestra un mensaje de error inline en vez de fallar en silencio.
9. **[Menor] `BeneficiarioFormPage.tsx`** → `fecha_nacimiento` vacía ahora se envía como `undefined` (no `''`); el mensaje de error genérico fue reemplazado por una extracción real de `err.response.data.detail` (string, `{message}` o array de errores Pydantic).
10. **[Menor] `GestionDetalleDrawer.tsx` sin guard de rol en "Modificar estado"** → el componente ahora recibe una prop `canModify: boolean` (calculada en `GestionesListPage.tsx`, la misma lógica que ya usaba la tabla) y solo renderiza el botón si es `true`.
11. **[Menor] `DashboardPage.tsx` sin manejo de `isError` de `usePortalUser`** → agregado un bloque de error visible ("No pudimos cargar tu perfil de acceso...") en vez de dejar la pantalla completamente en blanco cuando `/api/v1/portal/me` falla.
12. **[Menor] `programas/router.py` sin `require_roles`** → los 3 endpoints (`listar_programas`, `get_programa`, `estadisticas_programa`) ahora usan `Depends(require_roles(*ROLES_LECTURA))` en vez de `Depends(get_current_user)` — un usuario con rol `invitado` (no registrado en `portal_usuarios`) ya no puede leer el catálogo. El test `test_lectura_requiere_autenticacion` (que documentaba el bug como comportamiento esperado) se renombró a `test_lectura_denegada_a_invitado` y ahora afirma el 403 correcto; se agregó `test_lectura_permitida_a_consulta` para no perder cobertura del camino positivo.
13. **[Menor] `onEditExisting` no limpiaba `editError` en `CordobaHogarPage.tsx`** → agregado `setEditError(null)` (paridad con CC).

### Tests nuevos agregados junto con estos fixes
- `test_reordenar_estado_recomputa_estado_general_de_municipios` / `..._localidades` (CC y CH)
- `test_crear_municipio` / `test_crear_municipio_duplicado_devuelve_409` (antes no existía NINGÚN test de creación para CC)
- `test_crear_localidad` / `test_crear_localidad_duplicada_devuelve_409` (ídem CH)
- `test_numero_expediente_colision_se_resuelve_con_reintento`
- `test_lectura_denegada_a_invitado` (renombrado) + `test_lectura_permitida_a_consulta` (nuevo)

### Tests nuevos agregados el mismo día para cerrar gaps de cobertura señalados (2026-07-23, a pedido del usuario tras preguntar "¿queda algún test por desarrollar?")
- `test_estado_general_se_recomputa_al_cambiar_dimension` en CH (paridad con CC — antes solo existía para Cordón Cuneta, con fixture nueva `localidad_con_estado`).
- Visibilidad jerárquica de `listar_pedidos` (CC y CH, 4 tests cada uno): `test_listar_pedidos_vivienda_no_ve_infraestructura_ni_supervision`, `..._infraestructura_ve_vivienda_y_propia_no_supervision`, `..._supervision_ve_todo`, `..._admin_ve_todo` — con fixture `pedidos_multi_secretaria` y 2 usuarios de test nuevos en `conftest.py` (`INFRAESTRUCTURA_USER`/`client_infraestructura`, `SUPERVISION_USER`/`client_supervision`).
- Asignación automática de `secretaria` en `crear_pedido` (CC y CH, 3 tests cada uno): prioridad `supervision` > `infraestructura` > primera secretaría del actor.
- Total: 175/175 tests backend verdes (160 de la corrección de bugs + 15 de estos gaps de cobertura).

---

## ❌ Descartado / Won't fix

*(vacío — los 3 hallazgos que el usuario decidió no corregir en esta sesión (#15 asimetría de permisos, #16 xlsx, #18 checklist sin limpieza) se dejaron en "🟡 Pendiente — Menor" en vez de acá, porque no son un "no" definitivo sino "por ahora no, mantenerlo visible" — ver esa sección para el detalle y la razón de cada decisión.)*
