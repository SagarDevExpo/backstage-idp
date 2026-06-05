# Backstage Internal Developer Platform (IDP)

A production-grade [Backstage.io](https://backstage.io) Internal Developer Platform that automates microservice onboarding with self-service templates. Originally built for a payments enterprise running 150+ ECS Fargate services — this project demonstrates the full "golden path" workflow from developer request to running infrastructure.

## Architecture

```mermaid
graph TB
    subgraph Developer Experience
        DEV[Developer] -->|"Create Component"| BACKSTAGE[Backstage UI<br/>:3000]
        SLACK[Slack Bot<br/>/create-repo] -->|Alternative Interface| API[Backstage Backend<br/>:7007]
        BACKSTAGE --> API
    end

    subgraph Backstage Platform
        API --> SCAFFOLDER[Scaffolder Engine]
        API --> CATALOG[Software Catalog]
        API --> AUTH[Auth Provider<br/>Okta SSO / Guest]
        SCAFFOLDER -->|fetch:template| TEMPLATES[Software Templates]
    end

    subgraph Template Execution Pipeline
        TEMPLATES --> STEP1[1. Generate Code<br/>from Skeleton]
        STEP1 --> STEP2[2. Create GitLab Repo]
        STEP2 --> STEP3[3. Push to Pipeline<br/>Generator Repo]
        STEP3 --> STEP4[4. Register in<br/>Backstage Catalog]
    end

    subgraph CI/CD Automation
        STEP3 -->|"Triggers"| CODEPIPELINE[AWS CodePipeline<br/>pipeline-generator]
        CODEPIPELINE -->|"Creates"| NEW_PIPELINE[New App Pipeline]
        NEW_PIPELINE --> ECR[Amazon ECR]
        ECR --> ECS[ECS Fargate Service]
    end

    subgraph Runtime Infrastructure
        ECS --> ALB[Application Load Balancer]
        ECS --> CW[CloudWatch Logs]
        ECS --> SM[Secrets Manager]
        ALB -->|HTTPS| USERS[End Users]
    end

    subgraph Skeleton Output
        STEP1 -->|Generates| FILES[Spring Boot App<br/>+ Dockerfile<br/>+ ECS Task Def<br/>+ GitLab CI<br/>+ catalog-info.yaml]
    end
```

## How It Works

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant BS as Backstage UI
    participant Scaff as Scaffolder
    participant GL as GitLab
    participant PG as Pipeline Generator
    participant CP as CodePipeline
    participant ECS as ECS Fargate

    Dev->>BS: Fill template form (app name, team, env)
    BS->>Scaff: Execute template steps
    Scaff->>Scaff: Render skeleton with Nunjucks
    Scaff->>GL: Create repo (svc-order-processor)
    Scaff->>GL: Push code + Dockerfile + infra/
    Scaff->>PG: PR to pipeline-generator<br/>(environments/dev-123456789012.json)
    PG-->>CP: Merge triggers pipeline creation
    CP->>CP: Create new CodePipeline for app
    CP->>ECS: Deploy to Fargate
    Scaff->>BS: Register entity in catalog
    BS-->>Dev: Links: repo, catalog, pipeline PR
```

## Project Structure

```
backstage-idp/
├── app-config.yaml              # Main Backstage configuration
├── app-config.production.yaml   # Production overrides
├── packages/
│   ├── app/                     # Frontend (React)
│   └── backend/                 # Backend (Node.js + Express)
├── plugins/                     # Custom Backstage plugins
├── templates/
│   └── ecs-springboot-service/  # Custom software template
│       ├── template.yaml        # Template definition (parameters + steps)
│       └── skeleton/            # Code scaffold
│           ├── src/             # Spring Boot application
│           ├── infra/           # ECS task + service definitions
│           ├── build.gradle     # Gradle build (Spring Boot 3.3)
│           ├── Dockerfile       # Multi-stage container build
│           ├── .gitlab-ci.yml   # CI/CD pipeline
│           └── catalog-info.yaml # Backstage entity registration
└── examples/                    # Default Backstage examples
```

## Template: ECS Spring Boot Microservice

The custom template (`templates/ecs-springboot-service/`) provisions:

| Component | Technology | Details |
|-----------|-----------|---------|
| Application | Spring Boot 3.3 + WebFlux | Reactive, non-blocking |
| Messaging | Apache Kafka | Event-driven architecture |
| Container | Eclipse Temurin JRE Alpine | Minimal attack surface |
| Compute | ECS Fargate | Serverless containers |
| Load Balancer | ALB | Path-based routing, health checks |
| CI/CD | GitLab CI → CodePipeline | Push-to-deploy |
| Observability | CloudWatch + Actuator | Structured logging, metrics |
| Secrets | AWS Secrets Manager | Injected at runtime |

### Template Parameters

- **appName** — Must match `svc-*` pattern (e.g., `svc-order-processor`)
- **team** — payments / ecommerce / fulfillment / platform
- **environment** — dev / cert / prod
- **awsAccountId** — 12-digit AWS account number
- **javaVersion** — 17 or 21

## Quick Start

### Prerequisites

- Node.js 22+ 
- Yarn (Berry/v4)

### Run Locally

```bash
# Install dependencies
yarn install

# Start development server (frontend :3000 + backend :7007)
yarn start
```

Open http://localhost:3000 → Click **"Create"** → Select **"ECS Microservice (Spring Boot + Fargate + ALB)"**

### Production Deployment (ECS Fargate)

```bash
# Build the backend Docker image
yarn build:backend
yarn build-image

# Push to ECR and deploy
docker tag backstage:latest <account>.dkr.ecr.<region>.amazonaws.com/backstage:latest
docker push <account>.dkr.ecr.<region>.amazonaws.com/backstage:latest
```

## Configuration

Key settings in `app-config.yaml`:

| Setting | Value | Purpose |
|---------|-------|---------|
| `app.title` | Platform Engineering IDP | Branding |
| `backend.database` | better-sqlite3 (dev) / PostgreSQL (prod) | Persistence |
| `auth.providers` | guest (dev) / Okta (prod) | Authentication |
| `catalog.locations` | file-based templates | Service discovery |

## Tech Stack

- **Backstage** — CNCF Internal Developer Platform framework
- **React** — Frontend UI
- **Node.js + Express** — Backend API
- **SQLite** (dev) / **PostgreSQL** (prod) — Catalog database
- **Nunjucks** — Template rendering engine
- **Rspack** — Frontend bundler

## License

Internal use — Platform Engineering team.
