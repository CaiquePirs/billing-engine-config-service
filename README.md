# billing-engine-config-service

Centralized configuration repository for the **billing-engine-microservices** ecosystem, served by a [Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_spring_cloud_config_server) backed by Git.

On startup, each microservice registers with Eureka, locates the Config Server through discovery, and fetches its configuration (datasource, AWS queues/topics, JWT secrets, Stripe integration, OpenTelemetry observability, etc.) from this repository — keeping sensitive and environment-specific values out of the application code.

## How it works

- Each service has a `<service-name>-<profile>.yml` file (e.g., `payment-service-dev.yml`), following Spring Cloud Config's `{application}-{profile}.yml` convention.
- Client services locate the Config Server via Eureka discovery (`spring.cloud.config.discovery.enabled: true` + `service-id: config-server`).
- Sensitive values (passwords, API keys, secrets) are **never committed**. They are referenced through placeholders (`${VARIABLE}`) and resolved from environment variables at runtime.

## Configured services

| File | Service | Main responsibilities |
|---|---|---|
| `eureka-service-dev.yml` | Service Discovery (Eureka) | Service discovery server on port `8761` |
| `api-gateway-dev.yml` | API Gateway | Routing via Spring Cloud Gateway (WebFlux discovery locator), JWT validation (OAuth2 Resource Server), port `8080` |
| `authentication-service-dev.yml` | Authentication Service | Authentication/authorization, dedicated Postgres (`5433`), JWT issuance/validation |
| `customers-service-dev.yml` | Customers Service | Customer registration, dedicated Postgres (`5432`), Stripe integration |
| `subscription-service-dev.yml` | Subscription Service | Subscription management, dedicated Postgres (`5434`), SQS queues, SNS topics, Stripe integration |
| `invoice-service-dev.yml` | Invoice Service | Invoice generation, dedicated Postgres (`5436`), SQS queues (LocalStack and production AWS) |
| `payment-service-dev.yml` | Payment Service | Payment processing via Stripe, dedicated Postgres (`5435`), SQS/SNS, resilience with Resilience4j (retry + circuit breaker) |
| `notification-service-dev.yml` | Notification Service | Notification delivery, SQS queues, S3 bucket, AWS/email integration |

## Common patterns across services

- **Service discovery** — all clients register with Eureka via `eureka.client.service-url.defaultZone`.
- **Database** — services with persistence use PostgreSQL (`spring.datasource`) with Hibernate in `validate` mode (schema migrations managed outside of JPA). Each service owns a dedicated database on its own port.
- **Security** — JWT validation via the OAuth2 Resource Server (`spring.security.oauth2.resourceserver.jwt.secret-key`); services that issue tokens also carry client credentials.
- **Messaging** — AWS SQS/SNS integration, with LocalStack support for local environments and dedicated credentials/region for production.
- **Observability** — all services expose `health`, `info` and `metrics` via Spring Actuator and export **metrics, logs, and traces** in OTLP format (`management.otlp.*`), with distributed tracing sampled at 100% (`management.tracing.sampling.probability: 1.0`).

## Environment variables

The configuration files rely on environment variables for sensitive and environment-specific data, including:

- **Databases** — `*_DB`, `*_USER`, `*_PASSWORD`
- **Security** — `SECRET_KEY`
- **Service credentials** — `AUTHENTICATION_SERVICE_CLIENT_ID`, `AUTHENTICATION_SERVICE_SECRET`, `SUBSCRIPTION_SERVICE_CLIENT_ID`, `SUBSCRIPTION_SERVICE_SECRET`
- **AWS** — `AWS_REGION`, `AWS_URI`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`, `AWS_S3_BUCKET` (and `*_PRODUCTION` variants)
- **Queues/topics** — `*_QUEUE`, `*_TOPIC`
- **Stripe** — `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_WEBHOOK_SIGN`
- **Notifications** — `INTERNAL_SERVICE_EMAIL`
- **Discovery** — `EUREKA_ZONE_URL`
- **Observability (OTLP)** — `OTLP_METRICS_EXPORT_URL`, `OTLP_LOGGING_EXPORT_URL`, `OTLP_TRACING_EXPORT_URL`

These variables must be provided to the Config Server or to the client services themselves, depending on the deployment strategy in use (e.g., Docker Compose, Kubernetes Secrets, etc.).

## Relation to the main repository

This repository is part of the **billing-engine-microservices** ecosystem, which brings together the remaining services (gateway, authentication, customers, subscriptions, invoices, payments, and notifications) that consume these configurations.