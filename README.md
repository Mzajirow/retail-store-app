# Retail Store Sample App — Project Bedrock Deployment

AWS EKS deployment of the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app) with managed AWS data services.

## Architecture

The application consists of the following microservices:

| Service | Language | Database |
|---|---|---|
| UI | Java | — |
| Catalog | Go | RDS MySQL |
| Orders | Java | RDS PostgreSQL |
| Carts | Java | DynamoDB |
| Checkout | Node.js | Redis (in-cluster) |
| Assets | Nginx | — |

## Prerequisites

- kubectl configured for `project-bedrock-cluster`
- AWS CLI with appropriate credentials
- Infrastructure deployed from [bedrock-project-infrastructure](https://github.com/Mzajirow/bedrock-project-infrastructure)

## Repository Structure
src/app/
└── k8s/
├── base/
│   └── retail-store.yaml     # Base Kubernetes manifests
├── overlays/
│   └── aws/
│       ├── kustomization.yaml        # Kustomize overlay
│       ├── catalog-patch.yaml        # MySQL RDS endpoint override
│       └── orders-patch.yaml         # PostgreSQL RDS endpoint override
├── ingress.yaml              # ALB Ingress resource
└── rbac.yaml                 # Developer RBAC binding

## Deployment

### 1. Get RDS credentials from AWS

```bash
MYSQL_HOST=$(aws rds describe-db-instances \
  --db-instance-identifier bedrock-mysql \
  --region us-east-1 \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

POSTGRES_HOST=$(aws rds describe-db-instances \
  --db-instance-identifier bedrock-postgres \
  --region us-east-1 \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

MYSQL_PASS=$(aws secretsmanager get-secret-value \
  --secret-id project-bedrock/mysql \
  --region us-east-1 \
  --query SecretString \
  --output text | grep -o '"password":"[^"]*"' | cut -d'"' -f4)

POSTGRES_PASS=$(aws secretsmanager get-secret-value \
  --secret-id project-bedrock/postgres \
  --region us-east-1 \
  --query SecretString \
  --output text | grep -o '"password":"[^"]*"' | cut -d'"' -f4)
```

### 2. Create namespace

```bash
kubectl create namespace retail-app
```

### 3. Create patch files with RDS endpoints

```bash
cat > src/app/k8s/overlays/aws/catalog-patch.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog
  namespace: retail-app
data:
  RETAIL_CATALOG_PERSISTENCE_ENDPOINT: "${MYSQL_HOST}:3306"
EOF

cat > src/app/k8s/overlays/aws/orders-patch.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders
  namespace: retail-app
data:
  RETAIL_ORDERS_PERSISTENCE_ENDPOINT: "${POSTGRES_HOST}:5432"
EOF
```

### 4. Deploy with Kustomize

```bash
kubectl apply -k src/app/k8s/overlays/aws/
```

### 5. Patch database credentials

```bash
kubectl patch secret catalog-db -n retail-app \
  --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/data/RETAIL_CATALOG_PERSISTENCE_PASSWORD\", \"value\": \"$(echo -n $MYSQL_PASS | base64)\"}]"

kubectl patch secret orders-db -n retail-app \
  --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/data/RETAIL_ORDERS_PERSISTENCE_PASSWORD\", \"value\": \"$(echo -n $POSTGRES_PASS | base64)\"}]"
```

### 6. Apply Ingress and RBAC

```bash
kubectl apply -f src/app/k8s/ingress.yaml
kubectl apply -f src/app/k8s/rbac.yaml
```

### 7. Verify pods are running

```bash
kubectl get pods -n retail-app
```

All pods should show `1/1 Running`.

### 8. Get the application URL

```bash
kubectl get ingress -n retail-app
```

Copy the ADDRESS value and open `http://<ADDRESS>` in your browser.

## Data Layer

In-cluster databases have been replaced with managed AWS services:

| In-cluster | Managed AWS Service |
|---|---|
| MySQL StatefulSet | Amazon RDS MySQL 8.0 |
| PostgreSQL StatefulSet | Amazon RDS PostgreSQL 16.3 |
| MongoDB | Amazon DynamoDB |

RabbitMQ and Redis remain in-cluster as pods.

## Verifying Developer Access

```bash
# Should succeed
kubectl get pods -n retail-app --context bedrock-dev-view

# Should fail with Forbidden
kubectl delete pod -n retail-app --context bedrock-dev-view <pod-name>