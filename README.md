# devdb — Entorno local de bases de datos con Docker

Levanta **PostgreSQL, MySQL, MongoDB, Redis y Qdrant** (base de datos vectorial) con sus paneles web en segundos, elige **cuáles** quieres y si los datos
deben ser **temporales (en RAM)** o **persistentes (en un volumen)**. Pensado para desarrollo local, pruebas de
integración, CI, prototipos y **agentes de IA**.

```sh
./devdb up postgres redis        # levanta, espera a que estén listas y te da las URLs
✔ postgres listo → postgresql://dev:devpass@localhost:5432/devdb
✔ redis listo → redis://localhost:6379/0
```

---

## Índice

- [Características](#características)
- [Requisitos](#requisitos)
- [Instalación](#instalación)
- [Uso rápido](#uso-rápido)
- [Referencia del CLI](#referencia-del-cli)
- [Temporal vs. persistente](#temporal-vs-persistente)
- [Servicios, puertos y credenciales](#servicios-puertos-y-credenciales)
- [Configuración (.env)](#configuración-env)
- [Esquemas y datos semilla](#esquemas-y-datos-semilla)
- [Conectar tu aplicación](#conectar-tu-aplicación)
- [Paneles web](#paneles-web)
- [Scripts y CI](#scripts-y-ci)
- [Integración con agentes de IA](#integración-con-agentes-de-ia)
- [Uso sin el CLI (Docker Compose puro)](#uso-sin-el-cli-docker-compose-puro)
- [Solución de problemas](#solución-de-problemas)
- [Estructura del repositorio](#estructura-del-repositorio)

---

## Características

- **Elige tus bases de datos:** levanta solo las que necesitas (`postgres`, `mysql`, `mongo`, `redis`, `qdrant` o `all`).
- **Base de datos vectorial incluida:** [Qdrant](https://qdrant.tech) para RAG, búsqueda semántica y memoria de agentes.
- **Temporal o persistente:** datos en RAM (se borran al recrear) o en volúmenes de Docker (sobreviven reinicios).
- **Asistente interactivo** para humanos y **flags + salida JSON** para scripts y agentes.
- **Espera real a que estén listas:** healthchecks en cada base; `up` no termina hasta que aceptan conexiones.
- **URLs de conexión al instante:** `devdb url`, `devdb env` y `devdb status --json`.
- **Consola integrada:** `devdb shell` abre `psql`, `mysql`, `mongosh` o `redis-cli` sin instalarlos en tu máquina
  (y para Qdrant, un atajo a su API REST).
- **Esquemas y semillas:** coloca `.sql` / `.js` en `init/` y se aplican al crear la base; `devdb reset` los reaplica.
- **Configurable con `.env`:** puertos, credenciales y versiones de cada imagen.
- **Seguro por defecto:** los puertos solo se publican en `127.0.0.1`, no en tu red local.
- **Listo para IA:** incluye `AGENTS.md` y un *skill* (`skills/devdb/SKILL.md`) para Claude Code y otros agentes.
- **Sin dependencias extra:** solo Docker y Bash.

---

## Requisitos

- **Docker** con el plugin **Compose v2** (`docker compose version`). Docker Desktop ya lo incluye.
- **Bash 4 o superior** (Linux y WSL ya lo tienen; en macOS: `brew install bash`).
- Opcional: `jq` para formatear la salida JSON.

---

## Instalación

```sh
git clone <url-de-este-repo> devdb
cd devdb
./devdb install            # crea el comando `devdb` en ~/.local/bin
./devdb install --skill    # además, instala el skill para Claude Code (~/.claude/skills/devdb)
```

Después de instalarlo puedes usar `devdb` desde **cualquier directorio**. Si `~/.local/bin` no está en tu `PATH`,
el instalador te dice qué línea agregar a tu shell. Para desinstalar: `devdb uninstall`.

> ¿No quieres instalar nada? Ejecuta `./devdb ...` directamente desde la carpeta del repo.

---

## Uso rápido

### Modo interactivo

Ejecuta `devdb` sin argumentos y responde las preguntas:

```
$ devdb

devdb — ¿qué quieres levantar?

Bases de datos:
  postgres [S/n] s
  mysql [s/N] n
  mongo [s/N] n
  redis [s/N] s

Almacenamiento:
  1) Temporal (RAM) — rápido, se borra al recrear el contenedor
  2) Persistente (volumen) — los datos sobreviven reinicios
  Opción [1] 1

Paneles web:
  ¿Incluir paneles web (Adminer, Mongo Express, Redis Commander)? [S/n] s
```

Tu elección se recuerda: la próxima vez propone los mismos valores, y `devdb up` sin argumentos (fuera de una terminal
interactiva) repite la última selección.

### Modo directo

```sh
devdb up postgres                   # PostgreSQL en RAM + Adminer
devdb up mysql mongo --persist      # MySQL y MongoDB con datos persistentes
devdb up all --no-ui                # las 4 bases, sin paneles web
devdb status                        # qué está corriendo y cómo conectarse
devdb shell postgres                # consola psql
devdb down                          # detener todo
```

---

## Referencia del CLI

| Comando | Descripción |
|---|---|
| `devdb` | Asistente interactivo (igual que `devdb up` sin argumentos). |
| `devdb up [dbs...] [--ram\|--persist] [--ui\|--no-ui]` | Levanta las bases indicadas y **espera** a que estén listas. Por defecto: RAM y con paneles. |
| `devdb down [--volumes]` | Detiene y elimina los contenedores. `--volumes` también **borra los datos persistentes**. |
| `devdb status [--json]` | Estado, salud y URL de cada servicio. |
| `devdb url <db> [--docker]` | Imprime la URL de conexión. `--docker` usa el host interno de la red Docker. |
| `devdb env [--all] [--docker]` | Imprime `POSTGRES_URL=...`, `MYSQL_URL=...`, etc. de las bases que están corriendo (`--all`: todas). |
| `devdb shell <db> [-- args...]` | Abre el cliente nativo. Los argumentos después de `--` se pasan al cliente. |
| `devdb shell qdrant [MÉTODO] /ruta ['json' \| -]` | Llama a la API REST de Qdrant (requiere `curl`). `-` lee el cuerpo desde stdin. |
| `devdb logs [db] [--follow]` | Muestra los logs (últimas 100 líneas; cámbialo con `DEVDB_LOG_LINES`). |
| `devdb reset <db>` | Borra los datos de esa base y vuelve a ejecutar sus scripts de `init/<db>/`. |
| `devdb install [--skill]` / `devdb uninstall` | Instala o quita el comando global y el skill de Claude Code. |
| `devdb help` / `devdb --version` | Ayuda y versión. |

**Opciones globales:** `-q` / `--quiet` oculta los mensajes informativos.
**Alias aceptados:** `pg`, `postgresql` → `postgres`; `mongodb` → `mongo`; `vector`, `vectordb` → `qdrant`.
**Variables de entorno:** `DEVDB_TIMEOUT` (segundos de espera en `up`, por defecto 180), `DEVDB_BIN_DIR`
(destino de `install`), `NO_COLOR` (desactiva colores).

**Convenciones pensadas para automatizar:**

- Los **datos** (URLs, JSON, variables) salen por **stdout**; los **mensajes** por **stderr**.
- **Códigos de salida:** `0` éxito · `1` error (Docker no disponible, puerto ocupado, timeout…) · `2` uso incorrecto.
- **Nunca pregunta nada** si no hay una terminal interactiva (o si `CI` está definida).
- Es **idempotente**: ejecutar `up` dos veces no rompe nada.

---

## Temporal vs. persistente

| | `--ram` (por defecto) | `--persist` |
|---|---|---|
| Dónde viven los datos | Memoria (`tmpfs`) | Volúmenes Docker `devdb_<db>-data` |
| Sobreviven a `devdb down` | ❌ | ✅ |
| Sobreviven a reiniciar el equipo | ❌ | ✅ |
| Velocidad | Máxima | Normal |
| Se borran con | `down`, `reset` o recrear el contenedor | `devdb down --volumes` o `devdb reset <db>` |
| Ideal para | Tests, CI, agentes de IA, experimentos | Desarrollo diario con datos que quieres conservar |

Cambiar de modo (`devdb up postgres --persist` después de haberlo usado en RAM) recrea el contenedor con el nuevo
almacenamiento. Los datos del modo anterior no se migran.

---

## Servicios, puertos y credenciales

Valores por defecto (todos configurables en `.env`):

| Servicio | Puerto local | Host en la red Docker | Usuario | Contraseña | Base de datos |
|---|---|---|---|---|---|
| PostgreSQL 15 | `5432` | `postgres` | `dev` | `devpass` | `devdb` |
| MySQL 8.0 | `3306` | `mysql` | `dev` (o `root`) | `devpass` | `devdb` |
| MongoDB 7.0 | `27017` | `mongo` | `dev` (en `admin`) | `devpass` | `devdb` |
| Redis 7 | `6379` | `redis` | — | — (sin auth) | `0` |
| Qdrant 1.19 (vectorial) | `6333` (REST), `6334` (gRPC) | `qdrant` | — | — (sin auth) | colecciones |
| Adminer | [`8080`](http://localhost:8080) | `adminer` | | | |
| Mongo Express | [`8081`](http://localhost:8081) | `mongo-express` | sin login | | |
| Redis Commander | [`8082`](http://localhost:8082) | `redis-commander` | sin login | | |
| Panel de Qdrant | [`6333/dashboard`](http://localhost:6333/dashboard) | integrado en `qdrant` | sin login | | |

> ⚠️ Son credenciales de **desarrollo**. No expongas este entorno a internet.

---

## Configuración (.env)

```sh
cp .env.example .env
```

Todas las variables son opcionales. Las más útiles:

| Variable | Por defecto | Para qué |
|---|---|---|
| `DB_USER`, `DB_PASSWORD`, `DB_NAME` | `dev`, `devpass`, `devdb` | Credenciales comunes a las 4 bases |
| `POSTGRES_PORT`, `MYSQL_PORT`, `MONGO_PORT`, `REDIS_PORT` | `5432`, `3306`, `27017`, `6379` | Cambia el puerto si ya lo usa otro servicio |
| `QDRANT_PORT`, `QDRANT_GRPC_PORT` | `6333`, `6334` | Puertos REST y gRPC de Qdrant |
| `ADMINER_PORT`, `MONGO_EXPRESS_PORT`, `REDIS_COMMANDER_PORT` | `8080`, `8081`, `8082` | Puertos de los paneles |
| `POSTGRES_VERSION`, `MYSQL_VERSION`, `MONGO_VERSION`, `REDIS_VERSION`, `QDRANT_VERSION` | `15-alpine`, `8.0-oracle`, `7.0-jammy`, `7-alpine`, `v1.19.1` | Tag de cada imagen (p. ej. `POSTGRES_VERSION=17-alpine`) |
| `BIND_ADDRESS` | `127.0.0.1` | `0.0.0.0` para acceder desde otros equipos de tu red |

Las credenciales solo se aplican cuando la base se **crea**. Si las cambias en modo persistente, ejecuta
`devdb reset <db>` (borra los datos).

---

## Esquemas y datos semilla

Coloca tus archivos en `init/<db>/` y se ejecutarán automáticamente cuando la base se cree vacía, en orden alfabético:

```
init/
├── postgres/   01-schema.sql, 02-seed.sql, *.sql.gz, *.sh
├── mysql/      01-schema.sql, 02-seed.sql, *.sql.gz, *.sh
└── mongo/      01-seed.js, *.sh
```

- En modo **RAM** se aplican en cada arranque desde cero.
- En modo **persistente** solo la primera vez. Para reaplicarlos: `devdb reset <db>`.

Detalles en [`init/README.md`](init/README.md).

---

## Conectar tu aplicación

### Desde tu máquina (lo más común)

```sh
devdb env >> .env          # agrega POSTGRES_URL=..., REDIS_URL=... al .env de tu proyecto
eval "$(devdb env)"        # o expórtalas en la sesión actual
export DATABASE_URL="$(devdb url postgres)"
```

URLs por defecto:

```
postgresql://dev:devpass@localhost:5432/devdb
mysql://dev:devpass@localhost:3306/devdb
mongodb://dev:devpass@localhost:27017/devdb?authSource=admin
redis://localhost:6379/0
http://localhost:6333            # Qdrant (QDRANT_URL)
```

> En MongoDB el usuario se crea en la base `admin`, por eso la URL necesita `?authSource=admin`.

### Desde otro contenedor Docker

Todas las bases están en la red **`devdb-net`**. Si tu aplicación corre en su propio `docker-compose`, únela a esa red
y usa el **nombre del servicio** como host (`devdb url postgres --docker`):

```yaml
# docker-compose.yml de TU proyecto
services:
  api:
    build: .
    environment:
      DATABASE_URL: postgresql://dev:devpass@postgres:5432/devdb
      REDIS_URL: redis://redis:6379/0
    networks: [devdb-net]

networks:
  devdb-net:
    external: true
```

### Ejemplos por framework

<details>
<summary><b>NestJS</b> (TypeORM, Mongoose, ioredis)</summary>

```typescript
// PostgreSQL o MySQL con TypeORM
TypeOrmModule.forRoot({
  type: 'postgres',             // o 'mysql'
  url: process.env.POSTGRES_URL, // o process.env.MYSQL_URL
  autoLoadEntities: true,
  synchronize: true,             // solo en desarrollo
});

// MongoDB con Mongoose
MongooseModule.forRoot(process.env.MONGO_URL);

// Redis con ioredis
const redis = new Redis(process.env.REDIS_URL);
```
</details>

<details>
<summary><b>Python</b> (SQLAlchemy, PyMongo, redis-py)</summary>

```python
import os
from sqlalchemy import create_engine
from pymongo import MongoClient
import redis

pg = create_engine(os.environ["POSTGRES_URL"])  # driver psycopg2
my = create_engine(os.environ["MYSQL_URL"].replace("mysql://", "mysql+pymysql://"))
mongo = MongoClient(os.environ["MONGO_URL"])
r = redis.Redis.from_url(os.environ["REDIS_URL"])
```
</details>

<details>
<summary><b>Qdrant</b> (Python, TypeScript, LangChain)</summary>

```python
# pip install qdrant-client
import os
from qdrant_client import QdrantClient, models

client = QdrantClient(url=os.environ["QDRANT_URL"])
client.create_collection("docs", vectors_config=models.VectorParams(size=384, distance=models.Distance.COSINE))
client.upsert("docs", points=[models.PointStruct(id=1, vector=[0.1] * 384, payload={"texto": "hola"})])
hits = client.query_points("docs", query=[0.1] * 384, limit=3).points
```

```typescript
// npm install @qdrant/js-client-rest
import { QdrantClient } from '@qdrant/js-client-rest';
const qdrant = new QdrantClient({ url: process.env.QDRANT_URL });
```

```python
# LangChain: pip install langchain-qdrant
from langchain_qdrant import QdrantVectorStore
store = QdrantVectorStore.from_existing_collection(
    embedding=mis_embeddings, collection_name="docs", url=os.environ["QDRANT_URL"])
```
</details>

<details>
<summary><b>Node.js</b> (pg, mysql2, mongodb)</summary>

```javascript
import pg from 'pg';
import mysql from 'mysql2/promise';
import { MongoClient } from 'mongodb';

const pgPool = new pg.Pool({ connectionString: process.env.POSTGRES_URL });
const myConn = await mysql.createConnection(process.env.MYSQL_URL);
const mongo = await MongoClient.connect(process.env.MONGO_URL);
```
</details>

---

## Paneles web

Se incluyen por defecto (desactívalos con `--no-ui`). Solo se levantan los paneles de las bases que elegiste.

| Panel | URL | Para |
|---|---|---|
| Adminer | http://localhost:8080 | PostgreSQL y MySQL |
| Mongo Express | http://localhost:8081 | MongoDB (sin login) |
| Redis Commander | http://localhost:8082 | Redis (sin login) |
| Panel de Qdrant | http://localhost:6333/dashboard | Qdrant (integrado, siempre disponible) |

> ⚠️ **En los paneles no uses `localhost` como servidor.** Los paneles corren en su propio contenedor, y para ellos
> `localhost` es el propio panel. Usa el **nombre del servicio**: `postgres`, `mysql`, `mongo` o `redis`.

**Ejemplo en Adminer:** Sistema `PostgreSQL` · Servidor `postgres` · Usuario `dev` · Contraseña `devpass` · Base `devdb`.

---

## Scripts y CI

Ejemplo de tests de integración con una base desechable:

```sh
#!/usr/bin/env bash
set -euo pipefail
devdb -q up postgres --ram --no-ui
trap 'devdb -q down' EXIT
export DATABASE_URL="$(devdb url postgres)"
npm run migrate && npm test
```

Consultas y archivos sin abrir una consola interactiva:

```sh
devdb shell postgres -- -tAc "SELECT count(*) FROM users"
devdb shell mysql    -- -Nse "SHOW TABLES"
devdb shell mongo    -- --eval "db.users.countDocuments()"
devdb shell redis    -- KEYS '*'
devdb shell postgres < backup.sql
devdb shell qdrant GET /collections
devdb shell qdrant PUT /collections/docs '{"vectors":{"size":384,"distance":"Cosine"}}'
devdb shell qdrant PUT '/collections/docs/points?wait=true' - < puntos.json
```

Estado para máquinas:

```sh
devdb status --json | jq '.databases[] | select(.state=="running") | {name, health, url}'
```

```json
{
  "mode": "ram",
  "databases": [
    { "name": "postgres", "state": "running", "health": "healthy", "port": 5432,
      "url": "postgresql://dev:devpass@localhost:5432/devdb",
      "docker_url": "postgresql://dev:devpass@postgres:5432/devdb" }
  ],
  "ui": [ { "name": "adminer", "state": "running", "url": "http://localhost:8080" } ]
}
```

---

## Integración con agentes de IA

El repo incluye todo lo necesario para que un agente (Claude Code, Codex, Cursor, Copilot, Gemini CLI…) use el entorno
de forma segura y predecible:

| Archivo | Para qué |
|---|---|
| [`skills/devdb/SKILL.md`](skills/devdb/SKILL.md) | **Skill** (formato abierto *Agent Skills*) que le enseña a un agente a **usar** devdb desde cualquier proyecto: cuándo usar RAM, cómo obtener URLs, cómo consultar, qué no borrar sin permiso. |
| [`AGENTS.md`](AGENTS.md) | Instrucciones estándar para agentes que **modifican** este repo (arquitectura e invariantes). |
| [`CLAUDE.md`](CLAUDE.md) | Instrucciones para Claude Code (importa `AGENTS.md`). |

### Claude Code

```sh
./devdb install --skill      # enlaza el skill en ~/.claude/skills/devdb
```

Desde ese momento, en cualquier proyecto puedes pedir cosas como:

> *"Levanta un Postgres temporal, aplica las migraciones y corre los tests de integración."*
> *"Crea las tablas de `schema.sql` en una base MySQL de prueba y muéstrame cuántas filas quedaron."*
> *"Levanta Qdrant, indexa los archivos de `docs/` y prueba la búsqueda semántica del agente."*

El agente usará `devdb up postgres --ram --no-ui`, `devdb url postgres`, etc.

### Otros agentes

- Agentes compatibles con el estándar **Agent Skills**: copia o enlaza `skills/devdb/` en la carpeta de skills que
  indique su documentación.
- Agentes sin skills: agrega a las instrucciones de **tu** proyecto (`AGENTS.md`, `.cursorrules`,
  `.github/copilot-instructions.md`…) algo como:

  ```markdown
  ## Base de datos local
  Usa el CLI `devdb` para bases de datos de prueba. Nunca uses el modo interactivo.
  - Levantar: `devdb up postgres --ram --no-ui` (espera a que esté lista)
  - URL: `devdb url postgres` · Estado: `devdb status --json`
  - Consultar: `devdb shell postgres -- -tAc "SQL"`
  - No ejecutes `devdb down --volumes` ni `devdb reset` sin preguntar.
  ```

### Servidores MCP

Si usas un servidor MCP de bases de datos, apúntalo a la URL de `devdb url <db>`: el agente podrá inspeccionar
esquemas y consultar directamente las bases de este entorno.

---

## Uso sin el CLI (Docker Compose puro)

El CLI es opcional. Con Docker Compose combinas el archivo base, **una** capa de almacenamiento y los perfiles:

```sh
# PostgreSQL + Adminer en RAM
docker compose -f compose.yml -f compose.ram.yml --profile postgres --profile adminer up -d --wait

# Todo, persistente
docker compose -f compose.yml -f compose.persist.yml --profile all up -d --wait

# Detener
docker compose -f compose.yml -f compose.ram.yml --profile all down
```

Perfiles disponibles: `postgres`, `mysql`, `mongo`, `redis`, `qdrant`, `adminer`, `mongo-express`, `redis-commander`, `all`.

---

## Solución de problemas

| Síntoma | Solución |
|---|---|
| `port is already allocated` / `address already in use` | Otro servicio usa ese puerto. Cámbialo en `.env` (p. ej. `POSTGRES_PORT=15432`). |
| `No se puede hablar con el daemon de Docker` | Inicia Docker. En Linux, agrega tu usuario al grupo `docker` o usa Docker rootless. |
| `up` tarda mucho la primera vez | Está descargando las imágenes. Las siguientes veces arranca en segundos. |
| Timeout en `up` | Revisa `devdb logs <db>`. Puedes ampliar la espera con `DEVDB_TIMEOUT=300`. |
| Mis scripts de `init/` no se ejecutaron | Solo corren con la base vacía. Usa `devdb reset <db>`. |
| Los datos desaparecieron | Estabas en modo RAM. Usa `--persist`. |
| `devdb requiere bash >= 4` (macOS) | `brew install bash`. |
| El panel web no conecta | Usa el nombre del servicio (`postgres`, `mysql`…) como servidor, no `localhost`. |

---

## Estructura del repositorio

```
.
├── devdb                  # CLI (Bash)
├── compose.yml            # servicios, red, healthchecks y perfiles
├── compose.ram.yml        # capa de almacenamiento temporal (tmpfs)
├── compose.persist.yml    # capa de almacenamiento persistente (volúmenes)
├── .env.example           # variables configurables
├── init/                  # esquemas y datos semilla por base de datos
├── skills/devdb/SKILL.md  # skill para agentes de IA
├── .claude/skills/devdb   # → enlace al skill (Claude Code dentro de este repo)
├── AGENTS.md              # guía para agentes que modifican el repo
└── CLAUDE.md              # guía para Claude Code
```

---

## Licencia

[MIT](LICENSE)
