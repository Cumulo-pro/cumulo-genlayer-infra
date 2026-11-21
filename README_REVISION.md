# Notas de revision (NO subir a GitHub)

Borradores para sustituir los 5 .md del repo `Cumulo-pro/cumulo-genlayer-infra` (mismos nombres que usan las paginas PHP). Fecha: 2026-11-21.

| Borrador | Sustituye a |
|---|---|
| genlayer-validator-installation-cumulo.md | install.php |
| genlayer_staking_guide.md | staking.php |
| genlayer-key-management.md | key.php |
| genlayer_metrics_doc.md | metrics.php |
| faq.md | faq.php |

## Fuentes

Documentacion oficial (docs.genlayer.com, paginas descargadas el 2026-11-21): setup-guide, system-requirements, upgrade (incluye migracion v0.5 a v0.6), monitoring, genvm-configuration, CLI de staking, ops health/balance/metrics. Tu config.yaml y los datos de compose que pegaste. `/health` y `/metrics` reales de vuestro nodo (51.79.78.121:9153).

## URGENTE: el doc antiguo de claves tenia una contrasena en el ejemplo

El `genlayer-key-management.md` actual del repo (publico) incluye en el ejemplo `--source-password "LV@1tRf5EQ^"` y la direccion owner `0xea75b2f766eb89c40D636f11663143F9A618623F`. Si esa contrasena es o fue real, cambiadla y rotad esa clave. El nuevo borrador no contiene ninguna. Ojo: seguira en el historial de git del repo aunque se sustituya el fichero.

## Cosas que no he podido verificar (revisalas antes de subir)

1. Rutas del directorio: pongo `~/genlayer-bradbury-p2/v0.6.0-rc.2` como ejemplo (el nombre historico que me diste). Cambialo si prefieres no publicarlo.
2. Descripcion de los estados Active/Inactive/Quarantined/Banned en staking: la definicion de cuarentena ("por strikes: idle, votos incorrectos") es deduccion mia a partir del explorador y de /health. Confirmarla con la doc de staking de GenLayer.
3. `set-operator` en Phase 2: el registro exige prueba de posesion del operador. No he comprobado si `set-operator` tambien la exige. Lo dejo avisado en el doc. Probadlo en vuestra cuenta antes de publicar el procedimiento tal cual.
4. Mensajes de error de la tabla de key.md: `password is required for validator mode` esta en la doc oficial; `could not decrypt key with given passphrase` y `account not found` son los habituales de geth y del doc anterior, no los he provocado.
5. Opciones exactas de `genlayer account export/import/create/show` en la version rc (v0.40): la doc oficial solo lista los subcomandos, las opciones salen del doc antiguo.
6. Hot-reload del modulo LLM: la doc oficial usa el manager en 127.0.0.1:3999 pero dentro del contenedor; no he probado si el contenedor trae curl. En la FAQ recomiendo recrear el contenedor en vez de hot-reload.
7. Que el tarball trae `docker-compose.yaml`, `.env.example`, `genvm-module-web-docker.yaml` y `alloy-healthcheck.sh` esta tomado de la doc oficial, no de vuestro servidor.
8. Alertas Prometheus de metrics.md: expresiones razonables pero no probadas contra vuestro Prometheus.
9. Los enlaces entre guias son relativos (`staking`, `key`, `install`, `metrics`, `faq`, `index`, `validators`): funcionan en cumulo.pro pero no en la vista de GitHub.
10. Metricas: la lista sale de vuestro `/metrics` real (nombres), las descripciones de las que no tienen HELP propio las he resumido yo.

## Que no se ha tocado

Nada se ha publicado en GitHub. En los 5 PHP solo he cambiado el boton "View in GitHub" (apuntaba al repo de Arcium): ahora enlaza a la vista github.com del .md que cada pagina ya carga.
