# Multi-Cluster Deployment & State Isolation Guide

This guide explains how to deploy and maintain multiple, completely isolated agent clusters (e.g. `v3`, `v4`, `v5`, `hr-cluster`, `finance-cluster`) within the same Google Cloud Project without resource collisions or accidental overwrites.

---

## 1. Core Architectural Isolation Principle

Each cluster in our SuperApp architecture operates as an independent, decoupled stack:

```
                  Google Cloud Project (e.g. ceo-dev123)
  ┌─────────────────────────────────┐   ┌─────────────────────────────────┐
  │         Cluster V3 Stack        │   │         Cluster V4 Stack        │
  │ • ceoagent-gateway-v3           │   │ • ceoagent-gateway-v4           │
  │ • ceoagent-persistence-worker-v3│   │ • ceoagent-persistence-worker-v4│
  │ • ceoagent-billing-api-v3       │   │ • ceoagent-billing-api-v4       │
  │ • Topic: agent-turn-events-v3   │   │ • Topic: agent-turn-events-v4   │
  │ • Firestore: agent_threads_v3   │   │ • Firestore: agent_threads_v4   │
  │ • SA: ceoagent-gateway-sa-v3    │   │ • SA: ceoagent-gateway-sa-v4    │
  └────────────────┬────────────────┘   └────────────────┬────────────────┘
                   │                                     │
                   ▼                                     ▼
      gs://...-tfstate/ceodev-v3/           gs://...-tfstate/ceodev-v4/
```

---

## 2. Terraform State Isolation (`backend.hcl`)

The foundation of multi-cluster isolation is the **Google Cloud Storage Terraform Backend Prefix**.

Each cluster MUST write to its own unique GCS state path:

| Cluster | Remote State GCS Path |
| :--- | :--- |
| **V3** | `gs://<PROJECT_ID>-tfstate/ceodev-v3/middleware/default.tfstate` |
| **V4** | `gs://<PROJECT_ID>-tfstate/ceodev-v4/middleware/default.tfstate` |
| **V5 (Future)** | `gs://<PROJECT_ID>-tfstate/ceodev-v5/middleware/default.tfstate` |
| **Custom (e.g. HR)** | `gs://<PROJECT_ID>-tfstate/ceodev-hr/middleware/default.tfstate` |

### How It Works Automatically
Our deployment script [`scripts/cloudshell_deploy_middleware.sh`](../scripts/cloudshell_deploy_middleware.sh) automatically generates this file before running Terraform:

```bash
CLUSTER_SUFFIX="${CLUSTER_SUFFIX:-v4}"
TF_STATE_PREFIX="ceodev-${CLUSTER_SUFFIX}/middleware"

cat > backend.hcl <<EOF
bucket = "${PROJECT_ID}-tfstate"
prefix = "${TF_STATE_PREFIX}"
EOF

terraform init -backend-config=backend.hcl -reconfigure
```

---

## 3. Dynamic Resource Naming Checklist

All Terraform resources are fully parameterized to prevent `409 Already Exists` conflicts.

When configuring a new cluster in `infra/terraform/variables.tf`:

1. **Cloud Run Service Names**:
   * `gateway_service_name`: `"ceoagent-gateway-<SUFFIX>"`
   * `worker_service_name`: `"ceoagent-persistence-worker-<SUFFIX>"`
   * `billing_api_service_name`: `"ceoagent-billing-api-<SUFFIX>"`
2. **Pub/Sub Topic**:
   * `pubsub_topic_name`: `"agent-turn-events-<SUFFIX>"`
3. **Service Accounts**:
   * `gateway_service_account_name`: `"ceoagent-gateway-sa-<SUFFIX>"`
   * `worker_service_account_name`: `"ceoagent-worker-sa-<SUFFIX>"`
   * `billing_api_service_account_name`: `"ceoagent-billing-api-sa-<SUFFIX>"`
   * `eventarc_service_account_name`: `"ceoagent-eventarc-sa-<SUFFIX>"`
   * `billing_reconciler_service_account_name`: `"ceoagent-reconciler-sa-<SUFFIX>"`
4. **Firestore Collections**:
   * `firestore_threads_collection`: `"agent_threads_<SUFFIX>"`
   * `firestore_customer_wallets_collection`: `"customer_wallets_<SUFFIX>"`
   * `firestore_billing_reservations_collection`: `"billing_reservations_<SUFFIX>"`
   * `firestore_idempotency_collection`: `"processed_events_<SUFFIX>"`
   * `firestore_billing_ledger_collection`: `"agent_billing_ledger_<SUFFIX>"`
   * `firestore_wallet_transactions_collection`: `"wallet_transactions_<SUFFIX>"`
   * `firestore_customer_billing_periods_collection`: `"customer_billing_periods_<SUFFIX>"`
5. **Monitoring & Logging Metrics** (`monitoring.tf`):
   * Metric: `name = "${var.worker_service_name}_retryable_failures"`
   * Alert Policies: `display_name = "${var.gateway_service_name} 5xx responses"`, etc.

---

## 4. Step-by-Step: Deploying Any Brand-New Cluster

Whenever you create a new cluster repo (e.g. `ceodev-v5`):

### Step A: Local Setup
1. Copy `middleware-template` contents into your new project folder:
   ```powershell
   cd C:\Users\Admin\Desktop\ANTIGRAVITY\v5middleware
   git init -b main
   git remote add origin https://github.com/alneox2026/ceodev-v5.git
   ```
2. In `infra/terraform/variables.tf`, update default names with `v5` (or your cluster name).
3. In `config/agents.prod.yaml`, add your V5 agents.
4. Commit and push:
   ```powershell
   git add .
   git commit -m "feat: initial commit for v5 cluster"
   git push -u origin main
   ```

### Step B: Cloud Shell Deployment
```bash
# 1. Clone your new repo
cd ~
git clone https://github.com/alneox2026/ceodev-v5.git ceodev-v5
cd ceodev-v5

# 2. Build Middleware Images
bash ./scripts/cloudshell_build_middleware.sh

# 3. Deploy Middleware Infrastructure
CLUSTER_SUFFIX=v5 bash ./scripts/cloudshell_deploy_middleware.sh
```

---

## 5. Troubleshooting & FAQs

### Q: Why did an existing cluster disappear when I deployed a new one?
* **Reason**: Both repositories were using the exact same `prefix` in `backend.hcl`. Terraform thought you were renaming the services in that state and replaced them.
* **Fix**: Ensure each repo has its own unique `prefix = "ceodev-<CLUSTER>/middleware"` in `backend.hcl`, run `terraform init -backend-config=backend.hcl -reconfigure`, and redeploy both repos.

### Q: Why did Terraform throw `Error 409: Service account or Service already exists`?
* **Reason**: Resources were created in GCP under a previous state file and were missing from the new state file.
* **Fix**: Delete the orphan GCP resources with `gcloud run services delete ...` and `gcloud iam service-accounts delete ...`, then re-run `cloudshell_deploy_middleware.sh`.
