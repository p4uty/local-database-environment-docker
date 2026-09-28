---
name: devdb
description: Levanta, consulta y destruye bases de datos locales de prueba (PostgreSQL, MySQL, MongoDB, Redis y la base vectorial Qdrant) con el CLI `devdb`. Úsalo cuando necesites una base de datos real para ejecutar la app, correr tests de integración, probar migraciones o consultas SQL/Mongo/Redis, una base vectorial para RAG, embeddings o memoria de agentes, u obtener una URL de conexión (DATABASE_URL, REDIS_URL, QDRANT_URL, etc.) en desarrollo local.
---

# devdb — bases de datos de prueba locales

`devdb` es un CLI (envoltorio de Docker Compose) que levanta PostgreSQL, MySQL, MongoDB, Redis y Qdrant
(base de datos vectorial) en contenedores, espera a que estén listos y entrega las URLs de conexión.

## Antes de empezar

1. Comprueba que existe: `command -v devdb`. Si no, pide al usuario que clone el repo
   `local-database-environment-docker` y ejecute `./devdb install`. No lo instales por tu cuenta.
2. Revisa qué hay corriendo: `devdb status --json`.

## Reglas

- **Nunca uses el modo interactivo.** Pasa siempre las bases de datos explícitamente: `devdb up postgres`.
- **Usa `--ram` para tests y experimentos** (rápido, desechable). Usa `--persist` solo si el usuario quiere conservar datos.
- **Pasa `--no-ui`** salvo que el usuario pida los paneles web.
- **No ejecutes `devdb down --volumes` ni `devdb reset`** sobre datos persistentes sin confirmación del usuario: borran datos.
- Los datos (URLs, JSON, env) salen por **stdout**; los mensajes por **stderr**. Usa `-q` para silenciar mensajes.
- Códigos de salida: `0` ok, `1` error (Docker caído, puerto ocupado, timeout), `2` uso incorrecto.

## Flujo típico

```sh
devdb up postgres --ram --no-ui          # bloquea hasta que la base acepta conexiones
export DATABASE_URL="$(devdb url postgres)"
# ... migraciones, tests, la app ...
devdb down                                # al terminar, si tú lo levantaste
```

## Referencia rápida

| Comando | Qué hace |
|---|---|
| `devdb up <dbs...> [--ram\|--persist] [--no-ui]` | Levanta y espera healthcheck. dbs: `postgres mysql mongo redis qdrant all` |
| `devdb status --json` | Estado, salud (`healthy`), puertos y URLs de cada base |
| `devdb url <db>` | URL desde el host (`localhost`) |
| `devdb url <db> --docker` | URL desde otro contenedor en la red `devdb-net` (host = nombre del servicio) |
| `devdb env [--all]` | Líneas `POSTGRES_URL=...`, `MYSQL_URL=...`, `MONGO_URL=...`, `REDIS_URL=...`, `QDRANT_URL=...` |
| `devdb shell <db> -- <args>` | Ejecuta el cliente nativo dentro del contenedor |
| `devdb reset <db>` | Borra los datos de esa base y re-ejecuta sus scripts de `init/` |
| `devdb logs <db>` | Últimas líneas del log (útil si `up` falla) |

## Ejecutar consultas

El shell funciona sin TTY y acepta stdin, así que sirve para consultas puntuales y archivos:

```sh
devdb shell postgres -- -tAc "SELECT count(*) FROM users"
devdb shell mysql    -- -Nse "SHOW TABLES"
devdb shell mongo    -- --eval "db.users.countDocuments()"
devdb shell redis    -- GET mi-clave
devdb shell postgres < schema.sql
```

## Base de datos vectorial (Qdrant)

Para RAG, búsqueda semántica o memoria de agentes usa Qdrant (`devdb up qdrant --ram`). No tiene consola:
`devdb shell qdrant` llama a su API REST (requiere `curl` en el host). El tamaño del vector (`size`) debe coincidir
con el modelo de embeddings que uses.

```sh
devdb shell qdrant GET /collections
devdb shell qdrant PUT /collections/docs '{"vectors":{"size":384,"distance":"Cosine"}}'
devdb shell qdrant PUT '/collections/docs/points?wait=true' - < puntos.json   # cuerpo desde stdin
devdb shell qdrant POST /collections/docs/points/query '{"query":[...],"limit":5,"with_payload":true}'
```

En código usa `QDRANT_URL` (`devdb url qdrant`) con el cliente oficial (`qdrant-client`, `@qdrant/js-client-rest`)
o integraciones como `langchain-qdrant`. Panel web: `http://localhost:<puerto>/dashboard`.

## Credenciales por defecto

Usuario `dev`, contraseña `devpass`, base `devdb` (iguales en las 4 bases clásicas; Redis y Qdrant sin autenticación).
Puertos: 5432, 3306, 27017, 6379, 6333 (Qdrant REST) y 6334 (Qdrant gRPC). Se cambian en el `.env` del repo de devdb; **no asumas los valores**:
obtén siempre la URL real con `devdb url <db>`.

## Si algo falla

- `puerto ocupado` / `address already in use` → otro servicio usa el puerto. Indica al usuario que lo cambie en el `.env` del repo de devdb (p. ej. `POSTGRES_PORT=15432`).
- Timeout en `up` → `devdb logs <db>`.
- `No se puede hablar con el daemon de Docker` → Docker no está corriendo; pide al usuario que lo inicie.
