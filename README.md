# Cloud-Native-Customer-Billing-Platform-

A portfolio-grade, event-driven commerce backend that demonstrates how customer, catalog, purchase, billing, and fulfillment workflows can be split across independently deployable services.

Architecture

flowchart LR
    Client[Client / Postman] --> C[Commerce API\nC# / ASP.NET Core]
    C --> SQL1[(Azure SQL\nCommerce DB)]
    C -->|purchase.created| SB1[Azure Service Bus\nPurchase Topic]
    SB1 --> B[Billing Service\nC# / ASP.NET Core]
    B --> SQL2[(Azure SQL\nBilling DB)]
    B -->|billing.completed| SB2[Azure Service Bus\nBilling Topic]
    SB2 --> F[Fulfillment Service\nJava / Spring Boot]
    C -.audit.-> COS[(Cosmos DB)]
    F -.audit.-> COS
    C --> OBS[Prometheus / App Insights-ready metrics]
    B --> OBS
    F --> OBS

What this project demonstrates

C# and Java microservices with REST APIs

Customer, catalog, purchase, billing, and fulfillment workflows

Event-driven architecture with Azure Service Bus topics/subscriptions

Service Bus-triggered Azure Function for purchase audit/event processing

Azure SQL persistence with indexed query paths

Cosmos DB audit/event history

Asynchronous I/O, idempotent message handling, retries, and structured errors

Docker images and Kubernetes/AKS manifests

Liveness/readiness probes, metrics, autoscaling, and resource limits

Unit/integration-test projects and GitHub Actions CI

Terraform for AKS, ACR, Service Bus, Azure SQL, Cosmos DB, and Application Insights

Services

Service

Stack

Port

Main responsibility

Commerce API

C# / ASP.NET Core

8080

customers, catalog, purchases, purchase events

Billing Service

C# / ASP.NET Core

8081

invoices, purchase-event processing, billing events

Fulfillment Service

Java / Spring Boot

8082

fulfillment creation and billing-event processing

Purchase Audit Function

C# / Azure Functions

n/a

purchase-event validation and Cosmos DB audit persistence

Local quick start

Option A — Docker Compose

Copy the environment template:

cp .env.example .env

Start SQL Server and all services:

docker compose up --build

Local mode defaults MESSAGING_ENABLED=false, so each REST service can be exercised without Azure credentials. To test the real event chain, provide an Azure Service Bus connection string and set MESSAGING_ENABLED=true.

Useful endpoints

Commerce:

GET  http://localhost:8080/health
GET  http://localhost:8080/api/catalog
POST http://localhost:8080/api/customers
POST http://localhost:8080/api/purchases

Billing:

GET  http://localhost:8081/health
POST http://localhost:8081/api/invoices
GET  http://localhost:8081/api/invoices/{purchaseId}

Fulfillment:

GET  http://localhost:8082/actuator/health
POST http://localhost:8082/api/fulfillments
GET  http://localhost:8082/api/fulfillments/{purchaseId}

Example flow

Create a customer:

curl -X POST http://localhost:8080/api/customers \
  -H 'Content-Type: application/json' \
  -d '{"email":"demo@example.com","name":"Demo User"}'

Create a catalog item:

curl -X POST http://localhost:8080/api/catalog \
  -H 'Content-Type: application/json' \
  -d '{"sku":"PRO-001","name":"Pro Plan","price":49.99}'

Create a purchase:

curl -X POST http://localhost:8080/api/purchases \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: demo-order-001' \
  -d '{"customerId":1,"catalogItemId":1,"quantity":2}'

When Service Bus is enabled, purchase.created is consumed by Billing, which creates an invoice and emits billing.completed; Fulfillment then creates a fulfillment record.

Performance design

The code includes common optimizations that can be measured with the included benchmark harness:

database indexes for customer email, SKU, idempotency keys, and purchase IDs

async database and messaging calls

pagination-friendly query patterns

idempotent consumers to avoid duplicate work

compact event payloads

service-level metrics for latency and throughput

Run scripts/benchmark.py against a running Commerce API to capture your own before/after numbers. Do not claim the resume metric “35% faster” unless you have measured it in your environment.

Testing

C# services:

cd services/commerce-api/Tests && dotnet test
cd ../../billing-service/Tests && dotnet test

Java service:

cd services/fulfillment-service
mvn test

CI runs these jobs on every pull request and push to main.

Azure deployment

infra/terraform provisions the main Azure dependencies. infra/k8s contains AKS-ready Kubernetes manifests.

Typical flow:

cd infra/terraform
terraform init
terraform plan
terraform apply

Then build/push the three images to ACR and update image names in infra/k8s/platform.yaml before applying:

kubectl apply -f infra/k8s/platform.yaml

Security notes

No secrets are committed.

.env, Terraform state, and local secret files are ignored.

Kubernetes secret values in the sample manifest are placeholders only.

Use Key Vault / workload identity for a real deployment.

Azure SQL should use least-privilege identities and private networking in production.

See SECURITY.md for more detail.

Repository documentation

PROJECT_DESCRIPTION.md — recruiter/interview explanation and resume-safe bullets

ARCHITECTURE.md — technical design and event contracts

GITHUB_SETUP.md — upload instructions

VERIFICATION.md — what was and was not verified in the generation environment
