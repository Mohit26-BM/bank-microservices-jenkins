# Bank Service Portal (microservices + React)

A React portal for managing bank customers and accounts, backed by Spring Boot microservices.

```text
React (5173) ──► Auth service (9898)            register / sign in, returns a JWT
     │
     └────────► API Gateway (8765) ──► checks the JWT ──► Eureka (8761) finds the service
                                                         ├─► CUSTOMER  (8081)  /api/v1/customers
                                                         └─► ACCOUNTS  (8082)  /api/v1/accounts
```

| Module | Port | What it does |
| --- | --- | --- |
| `eureka-server` | 8761 | Service registry |
| `auth-service` | 9898 | Registers users and issues JWTs (H2 in memory) |
| `customer-service` | 8081 | Customer directory (H2 in memory, registers as `CUSTOMER`) |
| `account-service` | 8082 | Accounts; checks the customer exists first (H2 in memory, registers as `ACCOUNTS`) |
| `api-gateway` | 8765 | One entry point; rejects requests without a valid JWT |

All data is in memory, so it resets when a service restarts. Sample customers 1–3 and four sample accounts are created on start.

## Run it

You need **JDK 17+** (`JAVA_HOME` must point at it) and **Node 20+**. Maven is not needed; the project has its own wrapper.

**1. Backend** (PowerShell, from `backend`):

```powershell
cd backend
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-17'   # only if JAVA_HOME is not already set
.\start-all.ps1
```

This opens five windows (Eureka first, then the rest). The first run downloads dependencies, so give it a few minutes.
When it is ready, <http://localhost:8761> lists `ACCOUNTS`, `CUSTOMER`, `AUTH-SERVICE` and `BANK-GATEWAY`.

Prefer to do it by hand? In five terminals, from `backend`, start them in this order:

```powershell
.\mvnw.cmd -pl eureka-server spring-boot:run      # wait until it is up
.\mvnw.cmd -pl auth-service spring-boot:run
.\mvnw.cmd -pl customer-service spring-boot:run
.\mvnw.cmd -pl account-service spring-boot:run
.\mvnw.cmd -pl api-gateway spring-boot:run
```

Stop everything with `.\stop-all.ps1` (or close the windows).

**2. Front end** (second terminal):

```powershell
cd frontend
npm install        # first time only
npm run dev
```

Open <http://localhost:5173> and sign in with **demo / Demo@123** (or register a new user).

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Unsupported class file major version` or `release version 17 not supported` | Maven is using an old JDK. Set `JAVA_HOME` to JDK 17+ and open a new terminal. |
| Customers or Accounts page says *a backend service is unavailable* (503) | The service is not registered in Eureka yet. Wait a few seconds after starting, and check <http://localhost:8761>. |
| *Network error. Verify services and Gateway CORS* | The gateway (8765) or auth service (9898) is not running, or the front end is not on `localhost:5173` / `127.0.0.1:5173`. |
| *Session expired* right after login | The auth service and gateway must use the same `JWT_SECRET` (both default to the same value). |
| Port already in use | A previous run is still going. Run `.\stop-all.ps1`. |

## What was fixed in this project

- **There was no parent POM.** Both Maven modules pointed at a missing `bank-parent` (the gateway even at the wrong path), so nothing could build. Added `backend/pom.xml` with Spring Boot 4.1.1 and Spring Cloud 2025.1.3, the same versions as the Day 012–013 projects.
- **Spring Boot 4 / Spring Cloud 2025 names:** `spring-boot-starter-web` is now `spring-boot-starter-webmvc`, and `spring-cloud-starter-gateway` is now `spring-cloud-starter-gateway-server-webflux` (its settings live under `spring.cloud.gateway.server.webflux`).
- **The services behind the gateway were missing.** Eureka, `CUSTOMER` and `ACCOUNTS` did not exist in this folder, so the Customers and Accounts pages could not load. They are added now, with the JSON the React app expects.
- **The auth service hid its own errors.** A wrong password, a duplicate username or a validation failure all came back as an empty `403`. Errors now return the right status (401, 409, 400) with a readable message.
- **`npm run build` failed** with `Property 'env' does not exist on type 'ImportMeta'`. Added `src/vite-env.d.ts`.
- **The front end replaced every 404 with "API route not found".** It now shows the service's own message, such as *Customer 999 does not exist*.
