# Billing-engine-config-service

Centralized configuration repository for the **billing-engine-microservices** ecosystem, served by a [Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_spring_cloud_config_server) backed by Git.

Each microservice, on startup, registers with Eureka and fetches its configuration (datasource, AWS queues/topics, JWT secrets, Stripe integration, OTLP metrics, etc.) from this repository, keeping sensitive and environment-specific values out of the application code.

## How it works

- Each service has a `<service-name>-<profile>.yml` file (e.g., `payment-service-dev.yml`), following Spring Cloud Config's `{application}-{profile}.yml` convention.
- Client services point to the Config Server via Eureka discovery (`spring.cloud.config.discovery.enabled: true` + `service-id: config-server`).
- Sensitive values (passwords, API keys, secrets) are not committed to this repository — they are referenced via placeholders (`${VARIABLE}`) and resolved from environment variables at runtime.

## Configured services

| File | Service | Main responsibilities |
|---|---|---|
| `eureka-service-dev.yml` | Service Discovery (Eureka) | Service discovery server, port `8761` |
| `api-gateway-dev.yml` | API Gateway | Routing via Spring Cloud Gateway, JWT validation (OAuth2 Resource Server) |
| `authentication-service-dev.yml` | Authentication Service | Authentication/authorization, dedicated Postgres database, JWT issuance/validation |
| `customers-service-dev.yml` | Customers Service | Customer registration, dedicated Postgres database, Stripe integration |
| `subscription-service-dev.yml` | Subscription Service | Subscription management, SQS queues, SNS topics, Stripe integration |
| `invoice-service-dev.yml` | Invoice Service | Invoice generation, SQS queues (LocalStack and production AWS) |
| `payment-service-dev.yml` | Payment Service | Payment processing via Stripe, SQS/SNS queues, resilience with Resilience4j (retry + circuit breaker) |
| `notification-service-dev.yml` | Notification Service | Notification delivery, SQS queues, S3 bucket, AWS/email integration |

## Common patterns across services

- **Service discovery**: all clients register with Eureka via `eureka.client.service-url.defaultZone`.
- **Database**: services with persistence use PostgreSQL (`spring.datasource`) with Hibernate in `validate` mode (migrations managed outside of JPA).
- **Security**: JWT validation via `spring.security.oauth2.resourceserver.jwt.secret-key`.
- **Messaging**: integration with AWS SQS/SNS, with LocalStack support for local environments and dedicated credentials/region for production.
- **Observability**: all services expose `health`, `info` and `metrics` via Spring Actuator and export metrics in OTLP format (`management.otlp.metrics.export`).

## Environment variables

The configuration files rely on environment variables for sensitive and environment-specific data, including:

- Databases: `*_DB`, `*_USER`, `*_PASSWORD`
- Security: `SECRET_KEY`
- AWS: `AWS_REGION`, `AWS_URI`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`, `AWS_S3_BUCKET` (and `*_PRODUCTION` variants)
- Queues/topics: `*_QUEUE`, `*_TOPIC`
- Stripe: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
- Discovery: `EUREKA_ZONE_URL`
- Observability: `OTLP_METRICS_EXPORT_URL`

These variables must be provided to the Config Server or to the client services themselves, depending on the deployment strategy in use (e.g., Docker Compose, Kubernetes Secrets, etc.).

## Relation to the main repository

This repository is part of the **billing-engine-microservices** ecosystem, which brings together the remaining services (gateway, authentication, customers, subscriptions, invoices, payments, and notifications) that consume these configurations.