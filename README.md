# Alert Hub

Alert Hub notifies project managers about changes in their task-management tools (**GitHub, Jira, ClickUp**). Managers define **metrics** (e.g. "10 `bug` tasks in the last 12 hours") and **actions** (when the metric condition is met, send an **SMS** or **Email**). A scheduler checks actions every 30 minutes and the system sends the notifications through Kafka.

> Team project for the **MST × Hasoub Labs** Microservices & DevOps course.

---

## Architecture

```mermaid
flowchart LR
    subgraph Providers
        GH[GitHub files]
        JI[Jira files]
        CU[ClickUp files]
    end

    GH & JI & CU --> LOADER[loader-service]
    CLIENT([Client / Postman]) <--> USER[user-service]
    USER <--> SECURITY[security-service]
    USER --> EVAL[evaluation-service]
    USER --> METRIC[metric-service]
    USER --> ACTION[action-service<br/>+ job scheduler]

    ACTION -- Kafka: jobs --> PROC[processor-service]
    PROC -. Feign .-> LOADER
    PROC -. Feign .-> METRIC
    EVAL -. Feign .-> LOADER
    ACTION -. Feign .-> METRIC

    PROC -- Kafka: email / sms --> EMAIL[email-sender-service]
    PROC -- Kafka: email / sms --> SMS[sms-sender-service]
    EVAL -- Kafka: email --> EMAIL

    PROC & EMAIL & SMS --> LOGGER[logger-service]
```

Solid arrows = REST or Kafka · dotted arrows = OpenFeign calls.

---

## Services

| # | Service | Responsibility | Database | Owner |
|---|---------|----------------|----------|-------|
| 1 | `loader-service` | Reads the latest provider file (hourly + manual trigger), maps it to one schema, saves to `platformInformation` | MySQL | _TBD_ |
| 2 | `security-service` | Users, roles and permissions (`user`, `role`, `user_role`) | MySQL | _TBD_ |
| 3 | `user-service` | Entry point for clients; checks permissions and routes requests | MySQL | _TBD_ |
| 4 | `metric-service` | CRUD for metrics (label, threshold, time frame) | MySQL | _TBD_ |
| 5 | `action-service` | CRUD for actions + **job scheduler** (every 30 min, pushes due actions to Kafka) | MySQL | _TBD_ |
| 6 | `processor-service` | Consumes jobs, evaluates the metric conditions, produces `email` / `sms` messages | — | _TBD_ |
| 7 | `evaluation-service` | Developer insights (most-label, label aggregate, task amount), result sent by email | — | _TBD_ |
| 8 | `email-sender-service` | Consumes `email` topic and sends emails | — | _TBD_ |
| 9 | `sms-sender-service` | Consumes `sms` topic and sends SMS | — | _TBD_ |
| 10 | `logger-service` | Stores logs from processor / senders | MongoDB (`logs`) | _TBD_ |

Every service (except logger) must have: **service-layer tests**, **exception handling**, and **Swagger docs**.

---

## Tech stack

Java · Spring Boot · Spring Data JPA · MySQL · MongoDB · Apache Kafka · OpenFeign · springdoc (Swagger) · Lombok · SLF4J · JUnit/Mockito · Docker · Minikube (Kubernetes)

---

## Repository structure

```
alert-hub/
├── services/            # one independent Spring Boot project per microservice
├── data/                # provider files: github/  jira/  clickup/
├── infra/
│   ├── docker-compose.yml   # MySQL, MongoDB, Kafka for local development
│   └── k8s/                 # Minikube manifests
├── .github/
│   ├── workflows/           # one CI workflow per service (path-filtered)
│   └── CODEOWNERS
└── README.md
```

**Rule:** services never import each other's code. They talk only over REST (Feign) or Kafka. Each service has its own `pom.xml`, `Dockerfile` and database.

---

## Getting started

**Prerequisites:** JDK (team version: _TBD_), Maven, Docker Desktop, Git.

```bash
git clone https://github.com/<org>/alert-hub.git
cd alert-hub

# 1. start infrastructure (MySQL, MongoDB, Kafka)
cp infra/.env.example infra/.env      # fill in local passwords — never commit .env
docker compose -f infra/docker-compose.yml up -d

# 2. run a service
cd services/loader-service
./mvnw spring-boot:run
```

Swagger UI for each service: `http://localhost:<port>/swagger-ui.html`

### Ports (proposal)

| Service | Port | Service | Port |
|---|---|---|---|
| user-service | 8081 | processor-service | 8086 |
| security-service | 8082 | evaluation-service | 8087 |
| loader-service | 8083 | email-sender-service | 8088 |
| metric-service | 8084 | sms-sender-service | 8089 |
| action-service | 8085 | logger-service | 8090 |

### Kafka topics (proposal)

| Topic | Producer | Consumer | Content |
|---|---|---|---|
| `jobs` | action-service | processor-service | Due action to check |
| `email` | processor, evaluation | email-sender-service | Recipient email + message |
| `sms` | processor-service | sms-sender-service | Recipient phone + message |

---

## Provider data files

Files live in `data/<provider>/` and follow `{provider}_yyyy_MM_ddTHH_mm_ss`, e.g. `jira_2024_08_22T13_30_00`. The loader always reads the latest file it hasn't scanned yet. Empty numeric values are treated as `0`.

---

## How we work

- **Branches:** `feature/<service>-<short-name>`, `fix/<service>-<short-name>` (e.g. `feature/loader-hourly-scan`)
- **Commits:** `<type>(<service>): <message>`, e.g. `feat(loader): map jira fields to schema`
- **Pull requests:** no direct pushes to `main`; 1 approval + green CI required
- **Tasks:** tracked on the GitHub Project board, one issue per feature
- **Secrets:** never commit passwords or real personal data; use `.env` files (in `.gitignore`) and fake seed data

---

## Deployment (Part Two)

Each service gets a `Dockerfile` → image pushed to Docker Hub → deployed to **Minikube** with manifests in `infra/k8s/` (`Deployment` + `Service`, `Ingress` if needed).

---

## Team

| Name | GitHub | Services |
|---|---|---|
| _TBD_ | @… | … |
| _TBD_ | @… | … |
| _TBD_ | @… | … |
| _TBD_ | @… | … |
