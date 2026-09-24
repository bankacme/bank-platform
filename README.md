# bank-platform

Infraestructura local del sistema bancario.

## Comandos
- Levantar: `docker compose up -d`
- Estado: `docker compose ps`
- Apagar (conserva datos): `docker compose down`
- Apagar y borrar datos: `docker compose down -v`

## Conexión a Mongo desde las aplicaciones
`mongodb://localhost:27017/<base>?directConnection=true`
(dentro de Docker: `mongodb://mongo:27017/<base>?directConnection=true`)