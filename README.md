# 🛒 Ecommerce Microservices — CI/CD on GKE

A production-ready microservices project built with **Java 17 + Spring Boot 3**, deployed to **Google Kubernetes Engine** via **Harness CI/CD** with **GitOps**.

---

## 📐 Architecture

```
                        ┌─────────────────────────────────┐
   Client Request  ───► │      API Gateway  :8080         │
                        │  (Spring Cloud Gateway)         │
                        └──────────┬──────────┬───────────┘
                                   │          │
                    ┌──────────────▼──┐  ┌────▼───────────────┐
                    │ Product Service │  │   Order Service     │
                    │    :8081        │  │      :8082          │
                    │  (H2 in-mem DB) │  │  (H2 in-mem DB)    │
                    └─────────────────┘  └────────────────────┘
```

## 🗂️ Project Structure

```
ecommerce-microservices/
├── product-service/         # Product CRUD microservice
├── order-service/           # Order management microservice
├── api-gateway/             # Spring Cloud Gateway
├── k8s/                     # Kubernetes manifests
│   ├── namespace.yaml
│   ├── product-service/
│   ├── order-service/
│   └── api-gateway/
├── terraform/               # GKE cluster infrastructure
├── harness/                 # Harness CD pipeline YAML
├── .github/workflows/       # GitHub Actions CI
└── docker-compose.yml       # Local development
```

---

## 🚀 Quick Start — Local Development

### Prerequisites
- Java 17+, Maven 3.9+
- Docker & Docker Compose

### Run locally with Docker Compose
```bash
docker-compose up --build
```

### Test the APIs
```bash
# Product Service
curl http://localhost:8080/api/products
curl -X POST http://localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"name":"Laptop","description":"Pro laptop","price":999.99,"stock":10,"category":"Electronics"}'

# Order Service
curl http://localhost:8080/api/orders
curl -X POST http://localhost:8080/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":1,"quantity":2,"totalAmount":1999.98,"customerEmail":"test@test.com","customerName":"John Doe"}'

# Health checks
curl http://localhost:8080/actuator/health
curl http://localhost:8081/api/products/health
curl http://localhost:8082/api/orders/health
```

### Run individual services
```bash
# Build
cd product-service && mvn clean package -DskipTests

# Run
java -jar product-service/target/product-service-1.0.0.jar
java -jar order-service/target/order-service-1.0.0.jar
java -jar api-gateway/target/api-gateway-1.0.0.jar
```

---

## ☁️ GCP + GKE Setup

### 1. Set up GCP
```bash
export PROJECT_ID=your-gcp-project-id
gcloud config set project $PROJECT_ID

gcloud services enable container.googleapis.com artifactregistry.googleapis.com \
  compute.googleapis.com iam.googleapis.com

# Create Terraform state bucket
gsutil mb -p $PROJECT_ID -l us-central1 gs://tf-state-ecommerce2
gsutil versioning set on gs://tf-state-ecommerce2
```

### 2. Provision GKE with Terraform
```bash
cd terraform/
# Edit terraform.tfvars with your project ID first
terraform init
terraform plan
terraform apply -auto-approve
```

### 3. Connect kubectl
```bash
gcloud container clusters get-credentials ecommerce-cluster \
  --zone us-central1-a --project $PROJECT_ID
kubectl get nodes
```

### 4. Deploy to GKE
```bash
# Before applying: replace YOUR_PROJECT_ID in k8s/*/deployment.yaml
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/product-service/
kubectl apply -f k8s/order-service/
kubectl apply -f k8s/api-gateway/
kubectl get pods -n ecommerce
```

---

## 🔧 GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `GCP_PROJECT_ID` | Your GCP project ID |
| `GCP_SA_KEY` | Base64-encoded service account JSON key |
| `HARNESS_API_KEY` | Harness API key |
| `HARNESS_ACCOUNT_ID` | Harness account ID |
| `HARNESS_PIPELINE_ID` | Harness CD pipeline identifier |

---

## 🔄 CI/CD Pipeline Flow

```
git push main
    │
    ▼
GitHub Actions (CI)
    ├── Run tests (parallel, all 3 services)
    ├── Build Docker images
    ├── Push to Google Artifact Registry
    ├── Update k8s manifests with new image tag
    └── Commit manifests + Trigger Harness CD
           │
           ▼
       Harness CD
           ├── Deploy Product Service (Rolling)
           ├── Deploy Order Service (Rolling)
           ├── Manual Approval Gate
           └── Deploy API Gateway (Rolling + Health Check)
                  │
                  ▼
              GKE Cluster
              (ecommerce namespace)
```

---

## 📡 API Endpoints

### Product Service (`/api/products`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/products` | Get all products |
| GET | `/api/products/{id}` | Get product by ID |
| GET | `/api/products?name=laptop` | Search by name |
| GET | `/api/products?category=Electronics` | Filter by category |
| GET | `/api/products?inStock=true` | In-stock products |
| POST | `/api/products` | Create product |
| PUT | `/api/products/{id}` | Update product |
| PATCH | `/api/products/{id}/stock?quantity=5` | Adjust stock |
| DELETE | `/api/products/{id}` | Soft-delete product |

### Order Service (`/api/orders`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/orders` | Get all orders |
| GET | `/api/orders/{id}` | Get order by ID |
| GET | `/api/orders?customerEmail=x@y.com` | Orders by customer |
| GET | `/api/orders?status=PENDING` | Orders by status |
| POST | `/api/orders` | Create order |
| PATCH | `/api/orders/{id}/status?status=CONFIRMED` | Update status |
| POST | `/api/orders/{id}/cancel` | Cancel order |

---

## 📊 Monitoring

```bash
# Pod status
kubectl get pods -n ecommerce

# Logs
kubectl logs -f deployment/product-service -n ecommerce
kubectl logs -f deployment/order-service -n ecommerce
kubectl logs -f deployment/api-gateway -n ecommerce

# HPA status
kubectl get hpa -n ecommerce

# Resource usage
kubectl top pods -n ecommerce
kubectl top nodes
```

---

## 🧹 Cleanup

```bash
# Delete K8s resources
kubectl delete namespace ecommerce

# Destroy GKE cluster (stops billing)
cd terraform && terraform destroy -auto-approve
```

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Java 17 |
| Framework | Spring Boot 3.2 |
| Gateway | Spring Cloud Gateway |
| Database | H2 (in-memory, swap for PostgreSQL in prod) |
| Container | Docker (multi-stage builds) |
| Orchestration | Kubernetes / GKE |
| Registry | Google Artifact Registry |
| IaC | Terraform |
| CI | GitHub Actions |
| CD | Harness |
| GitOps | Harness + ArgoCD |
