# AGENTS.md

Guía para agentes de IA (Claude Code, Codex, Cursor, Copilot, Gemini CLI, etc.) que **trabajan en este repositorio**.
Para **usar** el entorno desde otro proyecto, consulta el skill en [`skills/devdb/SKILL.md`](skills/devdb/SKILL.md).

## Qué es

Un entorno local de bases de datos de prueba (PostgreSQL con pgvector opcional, MySQL, MongoDB, Redis, Qdrant vectorial y paneles web) sobre Docker Compose,
con un CLI en Bash (`devdb`) que sirve tanto a humanos (asistente interactivo) como a scripts y agentes (flags, `--json`,
códigos de salida). No hay código de aplicación ni build; la documentación de usuario (`README.md`) está en español.

## Estructura

| Archivo | Rol |
|---|---|
| `compose.yml` | Servicios, red `devdb-net`, healthchecks y perfiles. **Sin almacenamiento.** |
| `compose.ram.yml` | Capa temporal: `tmpfs` en el directorio de datos de cada base |
| `compose.persist.yml` | Capa persistente: volúmenes con nombre `devdb_<db>-data` (+ AOF en Redis) |
| `devdb` | CLI. Siempre combina `compose.yml` + una capa: `-f compose.yml -f compose.<ram\|persist>.yml` |
| `.env.example` | Todas las variables configurables con sus valores por defecto |
| `init/<db>/` | Scripts de esquema/semilla montados en `/docker-entrypoint-initdb.d` |
| `skills/devdb/SKILL.md` | Skill (formato Agent Skills) para agentes que *usan* devdb; enlazado en `.claude/skills/` |

## Comandos

```sh
./devdb up postgres redis --ram --no-ui   # levantar (espera healthchecks)
./devdb up postgres --pgvector            # Postgres con la extensión vector
./devdb shell qdrant GET /collections     # API REST de Qdrant
./devdb status --json                     # estado legible por máquina
./devdb down [--volumes]                  # detener
bash -n devdb                             # validar sintaxis del CLI
shellcheck devdb                          # lint (si está instalado)
docker compose -f compose.yml -f compose.ram.yml config -q      # validar compose
docker compose -f compose.yml -f compose.persist.yml config -q
```

No hay suite de tests: verifica los cambios levantando el entorno de verdad. Si los puertos por defecto están ocupados,
exporta otros antes de probar (p. ej. `POSTGRES_PORT=15432 ./devdb up postgres`).

## Invariantes (no romper)

- **Valores por defecto duplicados a propósito:** cada variable tiene su default en `compose.yml` (`${VAR:-x}`), en el
  bloque de configuración de `devdb` y en `.env.example`. Si cambias uno, cambia los tres.
- **Perfiles:** cada base tiene perfil `<db>` y cada panel `<nombre-del-panel>`; todos tienen además `all`. El CLI
  asigna los paneles en `ui_of()` (postgres/mysql → adminer, mongo → mongo-express, redis → redis-commander).
  Qdrant no tiene contenedor de panel: su dashboard va integrado (`qdrant-dashboard` en `status` es virtual).
- **pgvector es una variante de la imagen de Postgres, no un servicio aparte:** `compose.yml` usa
  `${POSTGRES_IMAGE:-postgres:${POSTGRES_VERSION}}`. Con `--pgvector` (guardado como `PGVECTOR` en `.devdb.state`),
  `compose()` exporta `POSTGRES_IMAGE` con `pgvector_image()` (misma versión mayor que `POSTGRES_VERSION`), salvo que
  el usuario la haya definido en `.env`. Tras `up`/`reset`, `enable_pgvector()` ejecuta `CREATE EXTENSION IF NOT EXISTS vector`.
- **Qdrant no tiene cliente de consola:** `devdb shell qdrant` es `qdrant_http()`, un atajo a su API REST vía `curl`
  en el host. Su imagen no trae curl/wget; por eso el healthcheck usa `/dev/tcp` de bash.
- **Nombre de proyecto `devdb`, red `devdb-net`:** son fijos. Otros proyectos se conectan a la red por nombre y
  `devdb reset` asume los volúmenes `devdb_<db>-data`.
- **stdout = datos, stderr = mensajes.** Los comandos `url`, `env` y `status --json` deben seguir produciendo salida
  parseable. Códigos de salida: 0 ok, 1 error, 2 uso incorrecto.
- **Sin prompts cuando no hay TTY** (`interactive()`): los agentes y CI nunca deben quedarse esperando entrada.
- Los puertos se publican en `BIND_ADDRESS` (por defecto `127.0.0.1`), no en todas las interfaces.
- El CLI requiere bash ≥ 4 y solo depende de Docker (`jq` es opcional para el usuario, nunca para el script).

## Al agregar una base de datos nueva

1. Servicio en `compose.yml` con perfiles `[<db>, all]`, healthcheck, puerto con `BIND_ADDRESS` y variables del `.env`.
2. Entradas en `compose.ram.yml` y `compose.persist.yml` (+ volumen).
3. En `devdb`: `ALL_DBS`, `normalize_db`, `port_of`, `internal_port_of`, `url_of`, `ui_of`, `cmd_shell`
   (y el bloque de configuración con su puerto por defecto).
4. `.env.example`, `init/<db>/` si la imagen soporta scripts de init, `README.md` y `skills/devdb/SKILL.md`.
