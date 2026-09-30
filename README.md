# GW Services Deployment

This repository deploys the GW services as a Docker Compose stack. It includes wallet, exchange, analytics, and notification services together with their PostgreSQL, ClickHouse, MongoDB, and NATS dependencies.

## Services

| Service | Purpose | Host address | Dependencies |
| --- | --- | --- | --- |
| `gw-wallet` | User accounts and multi-currency wallets; publishes wallet events and integrates with exchange functionality | `http://localhost:8180` | PostgreSQL, NATS |
| `gw-exchanger` | Exchange service | `http://localhost:8181` | PostgreSQL |
| `gw-analytics` | Analytics service | `http://localhost:8182` | ClickHouse, NATS |
| `gw-notification` | Notification service | `http://localhost:8183` | NATS, MongoDB |
| `postgres` | Relational storage for wallet and exchange data | `localhost:5432` | — |
| `clickhouse` | Analytics storage | `localhost:8123` (HTTP), `localhost:9000` (native) | — |
| `mongo` | Notification data storage | `localhost:27017` | — |
| `nats` | Messaging with JetStream enabled | `localhost:4222` (client), `localhost:8222` (monitoring) | — |

All containers join the explicitly named `gw-network` bridge network. Containers resolve one another by Compose service name and connect using container ports (for example, `postgres:5432` or `gw-exchanger:8080`); do not use `localhost` for container-to-container connections. Published host ports are for local development and access. The exact HTTP routes and per-service configuration variables are defined by the individual service images; this deployment repository does not contain those service implementations.

## Requirements

- Docker Engine
- Docker Compose v2 (`docker compose`)

## Configuration

Compose loads environment variables from these files:

- `gw-wallet/config.env`
- `gw-exchanger/config.env`
- `gw-analytics/config.env`
- `gw-notification/config.env`

Populate each file with the settings required by its corresponding service image before starting the stack. The checked-in connection settings use Compose service names (`postgres`, `nats`, `clickhouse`, `mongo`, and `gw-exchanger`) and container ports. Replace the development credentials and secrets before using the stack beyond local development.

PostgreSQL is initialized with the `postgres` user and `postgres` password by the current Compose configuration. Change these credentials and configure matching service connection strings before using the stack beyond local development. The SQL initialization scripts create the `gw_wallet` and `gw_exchange` PostgreSQL databases and the `gw_analytics` ClickHouse database when the respective database containers initialize a new data directory.

## Start the stack

The Compose file uses published `latest` service images; it does not build the application images locally. Pull the current images and start all services:

```sh
docker compose pull
docker compose up -d
```

View service status and logs:

```sh
docker compose ps
docker compose logs -f
```

Stop the services while preserving their data:

```sh
docker compose down
```

The database and broker data is stored in named volumes: `postgres-data`, `clickhouse-data`, `clickhouse-logs`, `mongo-data`, and `nats-data`. To remove the containers and permanently delete these persisted volumes:

```sh
docker compose down -v
```

Database initialization scripts run only when the relevant data directory is first initialized. Removing the volumes with `down -v` causes the databases to be created again on the next startup.

## Wallet API

The wallet HTTP API is available at `http://localhost:8180`. Its routes use the `/api/v1` prefix:

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/v1/register` | Register a user and create a wallet |
| `POST` | `/api/v1/login` | Authenticate and return a JWT |
| `GET` | `/api/v1/balance` | Read the authenticated user's balance |
| `POST` | `/api/v1/wallet/deposit` | Deposit funds |
| `POST` | `/api/v1/wallet/withdraw` | Withdraw funds |
| `GET` | `/api/v1/exchange/rates` | Get exchange rates |
| `POST` | `/api/v1/exchange` | Convert wallet funds |

All endpoints are under `/api/v1`. Protected endpoints require:

```http
Authorization: Bearer <token>
```

### Register

```sh
curl -X POST http://localhost:8180/api/v1/register \
	-H 'Content-Type: application/json' \
	-d '{"username":"alice","password":"secret","email":"alice@example.com"}'
```

### Login

The response contains a JWT valid for five minutes.

```sh
TOKEN=$(curl -s -X POST http://localhost:8180/api/v1/login \
	-H 'Content-Type: application/json' \
	-d '{"username":"alice","password":"secret"}' | jq -r .token)
```

### Check balance

```sh
curl -H "Authorization: Bearer $TOKEN" \
	http://localhost:8180/api/v1/balance
```

### Deposit or withdraw

Amounts are integer units in the selected currency. Supported currency columns are `USD`, `EUR`, and `RUB`.

```sh
curl -X POST http://localhost:8180/api/v1/wallet/deposit \
	-H "Authorization: Bearer $TOKEN" \
	-H 'Content-Type: application/json' \
	-d '{"currency":"USD","amount":100}'

curl -X POST http://localhost:8180/api/v1/wallet/withdraw \
	-H "Authorization: Bearer $TOKEN" \
	-H 'Content-Type: application/json' \
	-d '{"currency":"USD","amount":25}'
```

### Exchange currencies

Rates are provided by the exchange service:

```sh
curl http://localhost:8180/api/v1/exchange/rates

curl -X POST http://localhost:8180/api/v1/exchange \
	-H "Authorization: Bearer $TOKEN" \
	-H 'Content-Type: application/json' \
	-d '{"from_currency":"USD","to_currency":"EUR","amount":10}'
```

## Repository layout

```text
docker-compose.yaml        Compose definitions for services and dependencies
gw-wallet/config.env       Wallet service environment configuration
gw-exchanger/config.env   Exchange service environment configuration
gw-analytics/config.env    Analytics service environment configuration
gw-notification/config.env Notification service environment configuration
postgres/init.sql          PostgreSQL database initialization
clickhouse/init.sql        ClickHouse database initialization
```
