# Architecture

## Overview

A polyglot reminder management app built as a **learning project** showcasing multiple backend languages, frontend frameworks, and blockchain integration. All components are Docker-first.

## Component Map

```
Browser
  ├── React App (Next.js 14, port 3000)       — primary SPA frontend
  └── .NET MVC App (port 5050)                — server-rendered frontend

Nginx Load Balancer (port 9999)
  ├── .NET API (port 5000, internal 8080)     — layered DDD architecture
  ├── Go API (port 5001, internal 8080)       — gin + direct SQL
  └── C++ API (port 5002, internal 8080)      — crow + libpq

PostgreSQL (port 5432)                        — shared single database "Reminders"
Ganache (port 8545)                           — local Ethereum blockchain

Migrations Runner (exits after run)           — .NET console, runs EF Core migrations
```

## Data Flow

```
React App → fetch → NEXT_PUBLIC_API_BASE_URL (nginx:9999)
                         ↓  round-robin
                    [dotnet|go|cpp]-api
                         ↓
                    PostgreSQL (reminders table)
                         +
                    Ganache (Solidity contract — supplementary, not primary store)
```

The `.NET MVC` app talks to the nginx load balancer (`ApiOptions__BaseUrl`), not directly to any single API.

## Startup Sequence (Docker Compose)

1. `postgres` starts → healthcheck passes
2. `migrations` runs EF Core migrations → exits 0
3. `dotnet-api`, `go-api`, `cpp-api` all start simultaneously (depend on migrations)
4. `nginx` starts (depends on all 3 apis)
5. `mvc` and `react` start (depend on nginx)

`ganache` starts independently (no dependency ordering required).

## Blockchain Integration (.NET API only)

Each CUD operation on a reminder also writes to a Solidity `Reminders` contract deployed on Ganache via Nethereum. The blockchain is purely supplementary logging — the Postgres record is the source of truth. The Go and C++ APIs do **not** integrate blockchain.

## Compose Profiles

| Profile | Services Started |
|---------|-----------------|
| `all`   | Everything |
| `api`   | postgres, migrations, dotnet-api, go-api, cpp-api, nginx, ganache |
| `mvc`   | postgres, migrations, mvc, nginx (api not included by default) |
| `all-old` | mssql only (legacy) |

## External Dependencies

- **PostgreSQL** — primary database (Npgsql for .NET, `lib/pq` for Go, libpq for C++)
- **Ganache** — local Ethereum node (trufflesuite/ganache image)
- **Nethereum** — .NET library for Ethereum contract interaction
- **Nginx** — HTTP load balancer across 3 API backends
- **Entity Framework Core 8** — ORM for .NET API and migrations
- **Polly** — retry policy in migration runner
