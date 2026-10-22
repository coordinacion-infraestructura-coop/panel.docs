# Spec: Botón "Actualizar datos" del Tablero de Gobierno

**Estado**: implemented
**Versión**: 1.0.3
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-22
**Servicios**: `svc-datos-externos` (y, encadenado, `svc-privada`)
**Spec padre**: `spec-tablero-gobierno-panel.md`

## Changelog

- **1.0.3** (2026-10-22): (1) el botón ya no espera 15 minutos fijos: `/meta` informa
  `en_curso` y `terminado_en`, y el pedido termina cuando el nomenclador, las planillas y las
  fuentes oficiales registran una corrida posterior (medido: alrededor de un minuto). Terminado,
  se acepta otro pedido sin esperar. (2) Pedido del usuario: Transparencia y fuentes oficiales
  pasan a correr a diario (`sync-transferencias-transparencia` 08:30 UTC, nuevo endpoint
  `POST /internal/datos-externos/transferencias/sync-recientes`; `sync-fuentes-oficiales` 08:45
  UTC) y el botón también las dispara. (3) Hallazgo: `transparencia.cba.gov.ar` y la API de
  Estadística Córdoba (`smart-geo-data.cba.gov.ar`) responden **403 a las conexiones que salen
  de Google Cloud** (desde una IP argentina responden 200, con cualquier User-Agent): es un
  bloqueo por IP. Las corridas quedan registradas como falla; RENABAP e IDECOR cargan bien.
  Fuentes oficiales no había corrido nunca en producción: primera corrida 2026-10-22 (7 s).
  Backend `66ba835` + `93e09c4`, frontend `0f319cc`.
- **1.0.2** (2026-10-22): corrección tras el primer uso real. El primer pedido disparó los cuatro
  jobs pero tres devolvieron 502 (nomenclador y los dos ATP) y COPA falló con `KeyError`: el
  cliente de Google Sheets era uno solo por proceso y `googleapiclient`/httplib2 no se puede
  compartir entre hilos; con los jobs corriendo juntos, dos syncs en la misma instancia se
  rompían la conexión. Ahora hay un cliente por hilo (`svc-datos-externos` y `svc-gralgob`,
  backend `702ed17`). Verificado disparando los cuatro jobs a la vez: los cuatro en 200 y todas
  las fuentes en `ok`. Además, `/tablero/meta` informa el último pedido (`actualizacion`) para
  que el botón siga una actualización en curso aunque se recargue la página; el 409 ya no se
  muestra como error.
- **1.0.1** (2026-10-22): implementado y en producción. Backend `6ac5900`, gateway
  `ministerio-config-v20261022` (infra `aa09164`), frontend con el botón al lado del indicador de
  frescura. `svc-datos-externos@` con `roles/cloudscheduler.jobRunner` (verificado disparando
  `sync-gob-localidades` con esa cuenta: 200). Encadenado con Privada verificado: el sync de
  `intendentes` devolvió `privada_sincronizada` (0 cambios, Privada ya estaba al día).
- **1.0.0** (2026-10-22): pedido del usuario ("necesito que haya un botón actualizar para que
  actualicemos los datos"), con autorización vigente para decidir y desplegar.

## 1. Qué hace

Un botón **"Actualizar datos"** en la barra del tablero, al lado del indicador de frescura.
Dispara en ese momento los mismos jobs de Cloud Scheduler que corren solos cada día u hora:

| Job | Qué trae |
|---|---|
| `sync-gob-localidades` | Nomenclador y "0. Resumen General - Vista" |
| `sync-gob-transferencias` | Todas las planillas de Gobierno (transferencias, fondos, gestión, intendentes, elecciones) |
| `sync-atp-compromiso-gobernador` | ATP Compromisos (svc-gralgob) |
| `sync-gralgob-atp-interno` | ATP Interno (svc-gralgob) |
| `sync-transferencias-transparencia` | Transferencias publicadas en Transparencia (meses que falten) |
| `sync-fuentes-oficiales` | Estadística Córdoba, RENABAP e IDECOR |

Se dispara el job y no el sync directo porque cada sync tarda minutos y el gateway corta antes;
el job corre con su propio plazo (hasta 10 min) y su propia identidad. La respuesta vuelve en
segundos. La pantalla avisa que la actualización está en curso y recarga los datos cuando cambia
la frescura de las fuentes (consulta cada 30 s, hasta 15 min).

**Encadenado con Privada** (spec-padron-politico-gobierno-privada.md): cuando el sync de
`intendentes` termina, `svc-datos-externos` confirma la transacción y llama a
`POST /internal/privada/padron-politico/sync`, así Privada adopta el padrón nuevo sin esperar al
job de las 09:00 UTC. Vale también para la corrida diaria. Si Privada falla, se anota en el log y
el sync de Gobierno no se ve afectado.

## 2. Reglas

- `POST /api/v1/datos-externos/tablero/actualizar`, solo `Admin` y `Autoridad` (los mismos que
  ven el tablero). Es un refresco de fuentes de solo lectura, como el "Actualizar" de Resumen
  Territorial: no edita ningún dato.
- **Un pedido cada 15 minutos**: si hubo otro en ese lapso, 409 con la hora del anterior. Evita
  dos syncs completos superpuestos.
- Cada pedido queda en `ext_gob_sync_log` (`fuente = "actualizacion_manual"`, quién lo pidió y
  qué jobs se dispararon o fallaron), que sirve de auditoría.
- La cuenta `svc-datos-externos@` necesita `roles/cloudscheduler.jobRunner` en el proyecto.
- Flag `ACTUALIZAR_TABLERO_ENABLED` (apagado en `config.py`, encendido en `cloudbuild.yaml`).

## 3. Criterios de aceptación

- El botón dispara los cuatro jobs y la pantalla muestra el aviso.
- Un segundo clic antes de 15 minutos responde 409 y la pantalla lo explica.
- Un rol distinto de Admin/Autoridad recibe 403.
- Después del sync de intendentes, Privada queda sincronizada sin intervención.
