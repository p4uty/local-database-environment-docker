# Scripts de inicialización (esquemas y datos semilla)

Los archivos de estas carpetas se ejecutan **una sola vez, cuando la base de datos se crea vacía**,
en orden alfabético (usa prefijos como `01-schema.sql`, `02-seed.sql`).

| Carpeta          | Archivos soportados              | Se ejecuta con            |
|------------------|----------------------------------|---------------------------|
| `init/postgres/` | `.sql`, `.sql.gz`, `.sh`         | usuario `DB_USER` en `DB_NAME` |
| `init/mysql/`    | `.sql`, `.sql.gz`, `.sh`         | `root` en `DB_NAME`       |
| `init/mongo/`    | `.js`, `.sh`                     | `root` en `DB_NAME`       |

Redis y Qdrant no tienen scripts de inicialización.

- En modo **RAM** se vuelven a ejecutar cada vez que el contenedor se recrea.
- En modo **persistente** solo se ejecutan la primera vez. Para repetirlos: `./devdb reset <db>`.
