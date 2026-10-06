# bank-platform

Arranque del sistema bancario con `docker-compose` y guía del pipeline de los microservicios
(P2, paso 2.7). Diagrama: `docs/deployment.md`.

## Flujo de trabajo y pipeline

```
push a main        ──▶ verify ──▶ publish: hbcordova10/bank-<servicio>:latest (y sha-xxxxxxx)
etiqueta <svc>-vN  ──▶ verify ──▶ publish: además la etiqueta vN
```

Cada servicio tiene su propio pipeline completo en `.github/workflows/pipeline.yml` (sin
dependencias entre repositorios: cada uno se prueba y publica solo). Todos siguen el mismo esquema:

| Evento | `verify` (pruebas, Checkstyle, cobertura ≥ 80 % de líneas) | `publish` (imagen amd64 + arm64) |
|---|---|---|
| PR hacia `main` (si se usa) | Sí | No |
| Push a `main` | Sí | Sí: `latest` y `sha-<commit>` (solo si `verify` pasa) |
| Etiqueta `customer-v2`, `gateway-v1`, … | Sí | Sí: `v2`, `v1`, … |

- El resumen del run muestra la cobertura; el reporte HTML de Jacoco queda como artifact (14 días).
- Los servicios con Mongo levantan un `mongo:7.0` con replica set en el runner.
- Las pruebas que leen `../bank-config` funcionan porque el pipeline lo descarga al lado del servicio.
- La cobertura mínima se controla en el `pom.xml` (`jacoco.minimum.line`), así `./mvnw verify` falla
  igual en la máquina que en GitHub.

### Configuración (una sola vez)

1. **Docker Hub:** Account settings → Personal access tokens → token con permiso *Read & Write*.
2. **GitHub, organización `bankacme`** → Settings → Secrets and variables → Actions:
   - Secretos: `DOCKERHUB_USERNAME` (tu usuario) y `DOCKERHUB_TOKEN` (el token).
   - Variable: `DOCKERHUB_NAMESPACE` = `hbcordova10` (el usuario donde viven las imágenes).

## docker-compose

| Modo | Comando | Para qué |
|---|---|---|
| Solo Mongo | `docker compose up -d mongo` | Desarrollar: los servicios corren desde el IDE con `localhost` |
| Sistema completo | `docker compose pull && docker compose up -d` | Demo y pruebas de punta a punta con las imágenes `hbcordova10/bank-*:latest`, perfil `docker` |

Requisitos:
- Docker Desktop con unos **6 GB de memoria** (8 servicios Java de 512 MB + Mongo).
- Los cambios de `bank-config` **subidos a GitHub** (`main`): el Config Server lee la configuración
  directamente de `github.com/bankacme/bank-config`, igual que desde el IDE.

No mezclar modos: con el sistema completo arriba, los puertos 8080–8085, 8761 y 8888 están en uso y
un servicio arrancado desde el IDE no levanta. Para volver al IDE:
`docker compose stop config-server eureka-server customer-service account-service credit-service transaction-service report-service api-gateway`.

### Orden de arranque

`docker compose` espera a que cada dependencia esté sana (`healthcheck` sobre `/actuator/health`):

1. `mongo` (replica set `rs0`) y `config-server` (8888).
2. `eureka-server` (8761).
3. `customer-service` (8081), `account-service` (8082), `credit-service` (8083),
   `transaction-service` (8084), `report-service` (8085) y `api-gateway` (8080).

Todo listo cuando `docker compose ps` muestra `healthy` en los nueve contenedores y el panel
`http://localhost:8761` lista los seis servicios.

### Perfil `docker`

Cada servicio arranca con `SPRING_PROFILES_ACTIVE=docker` y `CONFIG_SERVER_URL=http://config-server:8888`.
El Config Server le agrega `bank-config/application-docker.yml`, que cambia los hosts a los nombres
de la red de Docker:

| Propiedad | IDE | Docker |
|---|---|---|
| `bank.mongo.host` (en la URI de Mongo de cada servicio) | `localhost` | `mongo` |
| `eureka.client.service-url.defaultZone` | `http://localhost:8761/eureka` | `http://eureka-server:8761/eureka` |

Las llamadas entre servicios no cambian: ya van por nombre de Eureka (`http://account-service/api/v1`).
Mongo usa `directConnection=true`, así que el `localhost:27017` del replica set no estorba dentro de Docker.

### Comandos

- Estado: `docker compose ps`
- Logs de un servicio: `docker compose logs -f account-service`
- Actualizar a lo último publicado en `main`: `docker compose pull && docker compose up -d`
- Tras un cambio en `bank-config` (con push): `docker compose restart config-server` y luego el servicio
- Apagar (conserva datos): `docker compose down`
- Apagar y borrar datos: `docker compose down -v`

### Conexión a Mongo

Desde la máquina: `mongodb://localhost:27017/<base>?directConnection=true`.
Dentro de Docker: `mongodb://mongo:27017/<base>?directConnection=true`.
