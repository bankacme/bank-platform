# Diagrama de despliegue — P2 (`docker-compose`)

Lo que levanta `docker compose up -d` en `bank-platform`, con las imágenes `hbcordova10/bank-*:latest` de Docker Hub. Todo corre en una red de Docker
(`bank-platform_default`); desde la máquina se entra por el Gateway (8080). Los puertos de los
servicios también se publican, solo para pruebas directas y para el panel de Eureka.

```mermaid
flowchart LR
    Postman([Postman / cliente HTTP])
    cfgrepo[("GitHub<br/>bankacme/bank-config")]

    subgraph host["Máquina local"]
        subgraph net["Red de Docker: bank-platform_default"]
            gw["api-gateway<br/>:8080"]
            eureka["eureka-server<br/>:8761"]
            config["config-server<br/>:8888"]
            subgraph svc["Servicios (perfil docker, 512 MB c/u)"]
                cust["customer-service<br/>:8081"]
                acc["account-service<br/>:8082"]
                cred["credit-service<br/>:8083"]
                tx["transaction-service<br/>:8084"]
                rep["report-service<br/>:8085"]
            end
            mongo[("mongo:7.0<br/>replica set rs0<br/>:27017")]
            vol[("volumen<br/>mongo-data")]
        end
    end

    Postman -->|HTTP :8080| gw
    gw -->|"lb:// (circuit breaker 2 s)"| svc
    gw -.->|consulta registro| eureka
    svc -.->|registro y latidos| eureka
    svc -.->|configuración al arrancar| config
    eureka -.->|configuración al arrancar| config
    gw -.->|configuración al arrancar| config
    config -->|"HTTPS, rama main"| cfgrepo
    acc -->|REST| cust
    acc -->|REST| cred
    cred -->|REST| cust
    cred -->|REST| tx
    tx -->|REST| acc
    rep -->|REST| acc
    rep -->|REST| cred
    rep -->|REST| tx
    cust --> mongo
    acc --> mongo
    cred --> mongo
    tx --> mongo
    mongo --- vol
```

## Arranque

```mermaid
flowchart LR
    m[mongo healthy] --> s
    c[config-server healthy] --> e[eureka-server healthy] --> s[customer, account, credit,<br/>transaction, report, api-gateway]
```

## Notas

- **Una imagen por servicio**, publicada en Docker Hub por el pipeline de GitHub Actions al hacer merge a `main` (amd64 y arm64). El `Dockerfile` tiene dos etapas (Maven → JRE 17) y
  separa las capas de Spring Boot: cambiar el código no vuelve a copiar las dependencias.
- **Sin estado en los contenedores** salvo Mongo: los datos viven en el volumen `mongo-data`, que
  sobrevive a `docker compose down` (se borra con `down -v`).
- **Una base por servicio** en el mismo Mongo; `report-service` no tiene base en P2.
- **Qué cambia en P3:** se suman Kafka (KRaft) y Redis a esta red, y `auth-service`,
  `debit-service` y `yanki-service`.
