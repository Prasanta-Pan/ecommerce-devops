# ecommerce-devops

Centralised DevOps repository for the E-Commerce Microservices Platform. This repo holds all deployment artefacts — CI/CD pipelines, Dockerfiles, Kubernetes manifests, and Terraform infrastructure — separated from business logic code. Each service's source code lives in its own GitHub repository; pipelines here pull from those repos at build time.

---

## Repository structure

```
ecommerce-devops/
├── .azure-pipelines/        # Azure DevOps pipeline definitions
│   ├── product-service.yml
│   ├── order-service.yml
│   ├── invoice-service.yml
│   ├── ecommerce-ui.yml
│   └── infra.yml
├── docker/                  # Dockerfiles per service
│   ├── product-service/
│   ├── order-service/
│   ├── invoice-service/
│   └── ecommerce-ui/
├── k8s/                     # Kubernetes manifests per service
│   ├── product-service/     # deployment, service, configmap, secret, hpa
│   ├── order-service/       # deployment, service, configmap, secret, hpa
│   ├── invoice-service/     # airflow-values.yaml (Helm), configmap
│   └── ecommerce-ui/        # staticwebapp.config.json
├── infra/                   # Terraform root module + child modules
│   ├── providers.tf
│   ├── backend.tf
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tfvars.example
│   └── modules/
│       ├── existing-infra/  # Data sources only (AKS, ACR, PostgreSQL, VNet)
│       ├── eventhub/        # Azure Event Hub namespace + order-events hub
│       └── apim/            # Azure API Management + JWT/CORS/rate-limit policy
└── environments/            # Per-environment Terraform variable files
    ├── dev.tfvars
    ├── uat.tfvars
    └── prod.tfvars
```

---

## Service source repositories

| Service | Language | GitHub repo |
|---|---|---|
| product-service | .NET 8 | `Prasanta-Pan/product-service` |
| order-service | Java 21 / Spring Boot | `Prasanta-Pan/order-service` |
| invoice-service | Python 3.11 / Airflow | `Prasanta-Pan/invoice-service` |
| ecommerce-ui | React 18 / Vite | `Prasanta-Pan/ecommerce-ui` |

---

## Setting up Azure DevOps pipelines

### 1. Import this repository

Create a new Azure DevOps project and import this repo (or connect it as a GitHub-hosted repo).

### 2. Create service connections

Go to **Project Settings → Service Connections** and add:

| Connection name | Type | Purpose |
|---|---|---|
| `github-connection` | GitHub | Pull source code from service repos |
| `acr-connection` | Docker Registry / Azure Container Registry | Push images |
| `aks-connection` | Azure Resource Manager | Deploy to AKS |
| `ecommerce-azure-connection` | Azure Resource Manager | Terraform state + apply |

### 3. Create variable groups

Go to **Pipelines → Library** and create these variable groups.

**`env-dev-vars`** (for Dev environment):

| Variable | Example value | Secret? |
|---|---|---|
| `ACR_NAME` | `myecommerceacr` | No |
| `AKS_CLUSTER_NAME` | `ecommerce-dev-aks` | No |
| `AKS_RESOURCE_GROUP` | `ecommerce-dev-rg` | No |
| `K8S_NAMESPACE` | `ecommerce-dev` | No |
| `AZURE_TENANT_ID` | `xxxxxxxx-...` | No |
| `AZURE_CLIENT_ID` | `xxxxxxxx-...` | No |
| `DB_HOST` | `postgres.dev.postgres.database.azure.com` | No |
| `DB_USER` | `pgadmin` | Yes |
| `DB_PASSWORD` | `...` | Yes |
| `AZURE_EVENTHUB_CONNECTION_STRING` | `Endpoint=sb://...` | Yes |

**`env-uat-vars`** — same keys, UAT values, `K8S_NAMESPACE=ecommerce-uat`

**`env-prod-vars`** — same keys, Prod values, `K8S_NAMESPACE=ecommerce-prod`

**`ecommerce-global-vars`** (used by infra pipeline):

| Variable | Description |
|---|---|
| `TF_STATE_RG` | Resource group containing the Terraform state storage account |
| `TF_STATE_SA` | Storage account name for Terraform remote state |

The ecommerce-ui variable groups additionally need:

| Variable | Description |
|---|---|
| `VITE_AZURE_CLIENT_ID` | Azure AD client ID for MSAL |
| `VITE_AZURE_TENANT_ID` | Azure AD tenant ID |
| `VITE_PRODUCT_API_URL` | APIM gateway URL for product API |
| `VITE_ORDER_API_URL` | APIM gateway URL for order API |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Deployment token from Azure Static Web Apps |

### 4. Create environments

Go to **Pipelines → Environments** and create:

- `development` — no approval gate (auto-deploy)
- `uat` — add an approval gate (manual approval required)
- `production` — add an approval gate (manual approval required)
- `infrastructure` — add an approval gate (required before `terraform apply`)

### 5. Register pipelines

For each file in `.azure-pipelines/`, create a new pipeline in Azure DevOps pointing to that YAML file. All pipelines have `trigger: none` — they are run manually or via the Azure DevOps UI with an environment parameter.

---

## Running infrastructure changes

The infra pipeline manages the Event Hub namespace and APIM instance. Existing resources (AKS, PostgreSQL, VNet, ACR) are referenced as read-only data sources and are never modified by Terraform.

**Standard workflow — always plan before apply:**

1. Run the infra pipeline with `action=plan` and your target `environment` (dev/uat/prod).
2. Review the plan output in the pipeline logs.
3. If the plan looks correct, re-run with `action=apply`. This stage requires approval from the `infrastructure` environment.

**First-time setup:**

Before running the pipeline for the first time, ensure the Terraform state storage account exists. You can create it manually:

```bash
az group create --name <TF_STATE_RG> --location australiaeast
az storage account create --name <TF_STATE_SA> --resource-group <TF_STATE_RG> --sku Standard_LRS
az storage container create --name tfstate --account-name <TF_STATE_SA>
```

Then fill in your actual resource names in the relevant `environments/*.tfvars` file before triggering the pipeline.

---

## Running service pipelines

Each service pipeline (product-service, order-service, invoice-service, ecommerce-ui) follows the same pattern:

1. **CI stage** — checks out source from GitHub, builds and runs tests.
2. **Docker stage** — builds the container image and pushes it to ACR (not applicable for ecommerce-ui which deploys as a static site).
3. **Deploy stage** — deploys to the selected environment. Dev deploys automatically; UAT and Prod require approval.

Select the target environment when triggering the pipeline manually.
