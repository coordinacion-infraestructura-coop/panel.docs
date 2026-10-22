# Spec: Padrón político único — cruce Gobierno ↔ Privada (ADR-027, segunda parte)

**Estado**: implemented
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-22
**Servicios**: `svc-datos-externos`, `svc-privada`
**Antecede**: `spec-datos-externos-fase6-politico.md` §6 (paso siguiente pedido por el usuario)
**ADR**: ADR-027 (cierra el mecanismo: espejo hacia Privada, con Gobierno como autoridad)

## Changelog

- **1.0.1** (2026-10-22): implementado y en producción. Backend `d6c4866` (migraciones
  `svc-datos-externos` 0010 y `svc-privada` 0004 aplicadas antes del push), frontend `7d5582a`.
  `run.invoker` cruzado entre `svc-privada@` y `svc-datos-externos@`; Cloud Scheduler
  `sync-privada-padron-politico` (`0 9 * * *` UTC, con `svc-vivienda@` como la del espejo de
  localidades). Primera corrida: Privada leyó 430 filas; el sync actualizó 429 (504 campos;
  casi todas por grafía: Privada guardaba mayúsculas y Gobierno no), creó 1 localidad y a otra
  que estaba cargada sin `id_geo` le completó el vínculo; quedó 1 fila de Privada sin par en
  Gobierno. Segunda corrida: 0 cambios. Respaldo: 429 filas con el valor original. Cruce contra
  el respaldo — intendente: 208 iguales, 126 en otro orden, 75 parecidos, 17 distintos, 1 solo en
  Gobierno; partido: 371 iguales, 27 parecidos, 28 distintos, 1 solo en Gobierno. La localidad
  creada no tiene respaldo (no había valor original) y el cruce la cuenta como igual: detalle
  menor, sin efecto en los datos.
- **1.0.0** (2026-10-22): aprobado por el usuario con autorización para decidir y desplegar de
  forma autónoma. Criterio dado por el usuario: **por defecto, el dato de la planilla de
  Gobierno; si falta, el de Privada. Los dos padrones no deberían diferir. El valor original de
  Privada se conserva en una tabla de respaldo.** Medido antes en producción (solo lectura,
  conteos): ver §1.

## 1. Situación medida (2026-10-22)

| | Privada (`priv_localidades_info`) | Gobierno (`ext_gob_intendente`) |
|---|---|---|
| Localidades | 430 filas, 429 con `id_geo` (3 `id_geo` en dos filas) | 427, todas con `id_geo` |
| Con intendente / partido | 426 / 426 | 427 / 427 |
| Cruzan por `id_geo` | 425 | 425 |
| Solo en un lado | 1 `id_geo` sin par en Gobierno + 1 fila sin `id_geo` | 2 sin fila en Privada |

En las 425 comunes: intendente igual en 330 (124 solo cambian el orden nombre/apellido), parecido
en 74 (comparten parte del nombre), distinto en 17, falta en Privada en 4. Partido igual en 366,
distinto en 55 (mayormente grafías o alianzas: "VECINAL UCR" / "VECINAL/UNION CIVICA RADICAL"),
falta en Privada en 4.

## 2. Reglas

1. **Valor combinado** por localidad (por `id_geo`) y por campo (intendente, partido): el de
   Gobierno si tiene dato; si no, el de Privada; si ninguno, vacío. Cada valor lleva su `origen`.
2. **Privada adopta el valor combinado.** Un sync diario escribe el combinado en
   `priv_localidades_info.intendente_jefe_comunal` / `partido_politico` cuando difiere. Las filas
   que comparten `id_geo` reciben el mismo valor. Las localidades de Gobierno sin fila en Privada
   se crean con el nombre oficial del espejo `priv_geo_localidades` (si el `id_geo` no está en el
   espejo, se informa y no se crea).
3. **Respaldo**: antes de la primera escritura sobre una fila, se copia su valor original a
   `priv_localidades_info_respaldo` (una vez por fila; nunca se pisa). Es el dato "de Privada"
   del cruce.
4. **Gobierno no se escribe**: la planilla es de solo lectura. El "completar Gobierno con Privada"
   se resuelve en el valor combinado que muestra el panel Político (con su origen).
5. Lo que solo está en Privada y no es un gobierno local del nomenclador (1 `id_geo`, 1 fila sin
   `id_geo`) no se toca; se informa.
6. La edición manual de intendente y partido en Privada (`PUT /localidades-info`) **sigue
   disponible** (contrato byte-compatible, ADR-008), pero el próximo sync la reemplaza si Gobierno
   tiene dato. Donde Gobierno no tiene dato, lo editado en Privada pasa a ser el combinado.
   `habitantes` y `electores` no cambian.

## 3. Mecanismo

```
08:00 UTC  sync-gob-transferencias (svc-datos-externos)
           └─ fuente `intendentes`: planilla → ext_gob_intendente
              └─ lee GET svc-privada /internal/privada/padron-politico → ext_priv_padron_politico
09:00 UTC  sync-privada-padron-politico (Cloud Scheduler → svc-privada)
           └─ POST /internal/privada/padron-politico/sync
              ├─ lee GET svc-datos-externos /internal/datos-externos/padron-politico (combinado)
              ├─ respalda y escribe priv_localidades_info (audit log por fila)
              └─ crea las filas faltantes
```

- **svc-privada**: `GET /internal/privada/padron-politico` (por fila: departamento, localidad,
  `id_geo`, valores actuales y de respaldo) y `POST /internal/privada/padron-politico/sync`
  (`dry_run` opcional). 502 sin escribir nada si `svc-datos-externos` no responde. Migración
  `0004`: `priv_localidades_info_respaldo` (misma clave que la fila, `id_geo`, intendente,
  partido, `respaldado_en`).
- **svc-datos-externos**: `GET /internal/datos-externos/padron-politico` (combinado con origen,
  solo localidades con `id_geo`); tabla `ext_priv_padron_politico` (lo último leído de Privada,
  reemplazo completo; si Privada no responde, queda lo anterior y el log lo dice). Migración
  `0010`.
- Ambos endpoints son IAM-only (no van al gateway). `run.invoker` cruzado entre las cuentas de
  servicio de los dos servicios. Flags apagados en `config.py` y encendidos en `cloudbuild.yaml`
  (`PADRON_POLITICO_SYNC_ENABLED`, `PRIVADA_PADRON_FETCH_ENABLED`), como las federaciones previas.

## 4. Panel Político › Intendentes (solo Admin y Autoridad)

- La tabla muestra el valor combinado; si vino de Privada, lo marca ("dato de Privada").
- Bloque nuevo **"Cruce con Privada"**: por campo, conteo por estado (igual, igual en otro orden,
  parecido, distinto, solo Gobierno, solo Privada) y la lista de las localidades que no son
  iguales, con los dos valores. Compara Gobierno contra el **respaldo** de Privada (el valor
  original), así el cruce sigue visible después de que Privada adopte el de Gobierno.
- Comparación: sin tildes, mayúsculas, sin puntuación; "igual en otro orden" = mismas palabras.

## 5. Privacidad

Nombres y partidos viajan solo entre los dos servicios (IAM) y se ven solo en Político. Los logs
llevan conteos. El celular no sale de `svc-datos-externos`.

## 6. Criterios de aceptación

- Después del primer sync, `priv_localidades_info` coincide con Gobierno en las 425 comunes y
  tiene las 2 que faltaban.
- `priv_localidades_info_respaldo` tiene el valor original de cada fila modificada.
- El bloque "Cruce con Privada" muestra los conteos de §1.
- Si `svc-datos-externos` no responde, Privada no cambia.
