# Spec: Estados de los paneles CC / CH / ML — un listado por área

**Estado**: approved
**Versión**: 1.0.1
**Servicio**: `svc-vivienda` (módulos `cordon_cuneta`, `cordoba_hogar`, `mi_lugar`) + frontend `vivienda`
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21

### Changelog
- **1.0.1 (2026-10-21)** — §3: el listado General arranca con todos los estados (encontrado al
  probar la migración contra la base local).
- **1.0.0 (2026-10-21)** — aprobado. A1 y A2 según la propuesta; A3 resuelto por el usuario:
  Operador crea y edita estados, Supervisor además los borra (§4.1).
- **0.1.0 (2026-10-21)** — versión inicial, con decisiones abiertas (§1).

---

## 0. Origen

Pedido (2026-10-21): en el administrador de estados de los paneles Cordón Cuneta, Córdoba Hogar
y Mi Lugar, en vez de un único listado, que haya **un listado por área** (técnico, jurídico,
financiero), para que cada área corrija o agregue sus propios estados. El administrador lleva un
selector (switcher) para elegir qué listado editar, y los desplegables de cada estado muestran
solo los estados de su listado.

## 1. Situación actual y decisiones abiertas

Hoy cada programa tiene **un** catálogo (`viv_cc_estados`, `viv_ch_estados`, `viv_ml_estados`).
Cada estado lleva tres flags (`aplica_juridico`, `aplica_tecnico`, `aplica_financiero`) que
filtran los desplegables, y un mismo estado (mismo `id`) puede usarse a la vez como Jurídico y
como Presupuestario. El **Estado General** usa el catálogo entero, sin filtro. Desde
`spec-estado-tecnico-desde-checklist.md` el Técnico ya no sale de este catálogo sino de
`viv_checklist_estado_expediente` (compartido por los tres programas), y el check "Técnico" ya
no se muestra. Administran estados `Admin` y `Supervisor` (`ROLES_TRANSICION`).

| # | Decisión | Propuesta |
|---|---|---|
| A1 | ¿Qué pasa con el **Estado General**? | Un cuarto listado, "General", en el mismo switcher. |
| A2 | ¿Qué muestra la pestaña **Técnico**? | El catálogo del Checklist ("Estado del expediente"). Es **uno solo para los tres programas**: editarlo desde CC cambia también CH y ML. Opción B: mostrarlo de solo lectura con un enlace al Checklist. |
| A3 | ¿Quién edita cada listado? | **Resuelto (usuario, 2026-10-21):** por rol, no por área. `Operador` (y `Supervisor`, `Admin`) crea y edita estados de cualquier listado; `Supervisor` (y `Admin`) además los borra. Ver §4.1. |
| A4 | Estados que hoy se usan en más de un área | Se **duplican**: una copia por área (§3). Después de la migración, corregir el nombre en Jurídico no cambia el de Presupuestario. |

## 2. Modelo

Se agrega a las tres tablas de catálogo:

```
dimension  VARCHAR(16) NOT NULL   -- juridico | financiero | general | tecnico_anterior
CHECK (dimension IN ('juridico','financiero','general','tecnico_anterior'))
```

- Cada estado pertenece a **un solo** listado. `dimension` no se puede cambiar después del alta
  (para mover un estado se crea uno nuevo en el otro listado).
- `orden` pasa a ser el orden **dentro de su listado**.
- `tecnico_anterior` agrupa los estados que solo se usaban como Técnico antes de
  `spec-estado-tecnico-desde-checklist.md`. No aparece en el switcher; se conserva porque la
  columna congelada `etecnico` y el historial viejo los referencian ("Técnico (anterior)").
- Los flags `aplica_*` quedan congelados: no se leen ni se escriben, no se borran (vuelta atrás
  solo de código, mismo criterio que con `etecnico`).
- `area_responsable` (`spec-vivienda-tablero-analitico.md §4.4`) no cambia.

## 3. Migración de datos (una por los tres programas)

Para cada estado existente se calcula en qué listados se usa:

- `juridico` si `aplica_juridico` o si alguna fila (activa o borrada) lo tiene en `ejuridico`, o
  algún historial con `campo='ejuridico'` lo referencia;
- `financiero`, ídem con `aplica_financiero` / `efinanciero`;
- `general` **siempre** (v1.0.1): antes el Estado General podía tomar cualquier estado del
  catálogo; si General arrancara solo con los estados ya usados, el desplegable perdería opciones
  el día del deploy (en la base local quedaba con 1 estado de 16). Cada área depura después;
- excepción: un estado que no va a Jurídico ni a Presupuestario, que nadie usa como General y que
  solo tiene `aplica_tecnico` o lo referencia `etecnico`/historial `etecnico`, va solo a
  `tecnico_anterior`.

Consecuencia aceptada: como casi todos los estados tienen hoy los tres flags en `true` (es el
default), al principio los listados Jurídico, Presupuestario y General son copias del catálogo
actual. La separación real la hace cada área corrigiendo o borrando en su listado.

Si cae en un solo listado, se le pone esa `dimension` y conserva su `id`. Si cae en varios, el
primero (orden: jurídico, financiero, general) conserva el `id` y para cada otro se inserta una
copia con `id` nuevo, mismo label, colores, orden y `area_responsable`; y se repuntan a la copia
las filas de la entidad (`ejuridico` / `efinanciero` / `estado_general`) y del historial
(`estado_anterior_id` / `estado_nuevo_id` según `campo`).

Cada copia y cada repunte deja fila en `viv_audit_log`; el downgrade sale de ahí (mismo patrón
que la migración `0042`). La migración imprime un resumen por programa (estados duplicados,
filas repuntadas) y se corre primero en un dry-run de solo lectura contra producción (con
autorización) para revisar el resultado con el área.

## 4. Backend

- `GET .../estados`: devuelve `dimension`. Sin filtro nuevo (el frontend filtra).
- `POST .../estados`: `dimension` obligatoria (`juridico` | `financiero` | `general`); no se
  aceptan `tecnico_anterior` ni los flags `aplica_*`.
- `PATCH .../estados/{id}`: no acepta `dimension`.
- `DELETE`: sin cambios (sigue bloqueado si el estado está en uso).
- Alta/edición de entidad (CC/CH/ML): `ejuridico` debe ser un estado `juridico`,
  `efinanciero` uno `financiero`, `estado_general` uno `general`; si no, `422`. Hoy no se valida.
- `mi_lugar.listar_proyectos_ml`, filtro `estado_id`: sin cambio de forma.
- Sin endpoints ni paths de gateway nuevos.

### 4.1 Permisos del administrador de estados (A3)
- `POST` / `PATCH .../estados`: `ROLES_ESCRITURA` (Admin, Supervisor, Operador). Antes `ROLES_TRANSICION`.
- `DELETE .../estados/{id}`: `ROLES_TRANSICION` (Admin, Supervisor), sin cambios.
- Frontend: el botón del administrador se muestra a Operador; el de borrar, solo a Supervisor y Admin.
- **Pestaña Técnico:** su catálogo (`viv_checklist_estado_expediente`) se administra con los
  endpoints `/checklist-tecnico/admin/...`, que siguen siendo solo `Admin`
  (`spec-checklist-tecnico-dgv.md §8`). Para el resto de los roles la pestaña es de solo lectura,
  con la leyenda de que el listado se administra desde el Checklist. Abrir ese catálogo a
  Operador cambiaría el spec del Checklist y afecta a los tres programas a la vez: queda para
  confirmar aparte.

## 5. Frontend (`CordonCunetaPage.tsx`, `CordobaHogarPage.tsx`, `MiLugarPage.tsx`)

1. **Administrador de estados**: switcher arriba con **Jurídico · Presupuestario · General ·
   Técnico**. La lista muestra solo los estados del listado elegido, ordenados por su `orden`;
   "Nuevo estado" crea en ese listado. Se quitan los checks J / P.
2. **Pestaña Técnico** (según A2): edita `viv_checklist_estado_expediente` por los endpoints que
   ya existen del Checklist, con un aviso "compartido por Cordón Cuneta, Córdoba Hogar y Mi
   Lugar"; o bien la lista de solo lectura con enlace al Checklist.
3. **Desplegables** (modal de edición, alta, filtros): Jurídico, Presupuestario y General
   muestran solo su listado.
4. **% de avance**: `posJ` y `posF` pasan a medirse dentro de su propio listado
   (`índice / (largo − 1)`), como ya se hace con el Técnico. **El % de todas las filas cambia.**
5. Los tres paneles cambian juntos (mismo esqueleto).

## 6. Efectos visibles que se aceptan

- El % de avance cambia en todas las filas (§5.4).
- Los estados que se usaban en dos áreas aparecen repetidos, uno en cada listado.
- El historial viejo sigue mostrando los labels de siempre (los ids se repuntan, no se pierden).

## 7. Fuera de alcance

- Permiso de edición por área (A3 se resolvió por rol).
- Que el Checklist alimente Jurídico y Presupuestario (`spec-vivienda-estados-desde-checklist.md`).
- Borrar los flags `aplica_*` o los estados `tecnico_anterior`.

## 8. Pruebas

- Migración: estado compartido J+P → dos filas, entidades e historial repuntados; estado sin uso
  → `general`; estado solo técnico → `tecnico_anterior`; downgrade restaura. Verificada contra
  Postgres real (no solo SQLite).
- API: alta sin `dimension` → `422`; `ejuridico` con un estado `financiero` → `422`; `PATCH` con
  `dimension` se ignora o `422`.
- Frontend: `npm run build` + verificación manual de los tres paneles.

## 9. Deploy

Migración Alembic en `db_vivienda` (Cloud Shell) + backend + frontend el mismo día: un frontend
viejo seguiría filtrando por los flags `aplica_*`, que quedan congelados. Sin config de gateway.

## 10. Implementación (2026-10-21)

Rama `estados-por-area` en `panel.backend` y `panel.front`.

- **Migración `0044`**: verificada contra Postgres (caso sintético y copia de la base local: subida,
  bajada y nueva subida). En la base local: CC 16 estados → 48 (16 por listado), CH 15 → 45, ML 45 → 135.
  **Antes de desplegar, correrla en seco contra producción** (con autorización) para revisar los números.
- **Backend**: `app/estados_dimension.py` (`validar_dimensiones`, error `ESTADO_DE_OTRO_LISTADO`);
  columna `dimension` con default `general` en los modelos (para los seeds y tests que no la pasan).
- **Frontend**: un solo editor compartido, `src/modules/vivienda/estadosPorArea.tsx`
  (`EstadosPorAreaEditor`, `estadosDe`, `avanceEnListado`), que reemplaza los tres administradores de
  estados. En CH y ML el modal "Parámetros" se abre también a Operador, pero la pestaña de parámetros
  financieros sigue siendo solo de Supervisor/Admin.
- **Pruebas**: `tests/test_estados_por_area.py` (13) + suite completa; `npm run build` OK.
  Sin verificación visual todavía.
