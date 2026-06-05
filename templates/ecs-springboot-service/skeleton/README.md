# ${{ values.appName }}

${{ values.description }}

## Quick Start

```bash
./gradlew bootRun
```

The service starts on `http://localhost:8080`.

## Health Check

```
GET /actuator/health
```

## Architecture

- **Runtime:** ECS Fargate behind ALB
- **Service Discovery:** ECS Service Connect (namespace: `ecommerce`)
- **Async Events:** Kafka (MSK)
- **Secrets:** AWS Secrets Manager (injected via ECS task definition)
- **CI/CD:** GitLab CI → ECR → ECS deploy
- **Monitoring:** Dynatrace OneAgent + Splunk log forwarding

## Infrastructure

| File | Purpose |
|------|---------|
| `infra/ecs-task-definition.json` | Fargate task definition (CPU, memory, container config) |
| `infra/ecs-service-definition.json` | ECS service with ALB, Service Connect, circuit breaker |
| `Dockerfile` | Multi-stage build for production image |
| `.gitlab-ci.yml` | Build → Test → Publish → Deploy pipeline |

## Deployment

Deployments are automatic on merge to `main`:
1. GitLab CI builds the JAR and Docker image
2. Pushes to ECR (`${{ values.awsAccountId }}.dkr.ecr.us-east-1.amazonaws.com/${{ values.appName }}`)
3. Forces new ECS deployment (blue-green via circuit breaker rollback)

## Owner

Team: **${{ values.team }}**
