# Production Cloud Middleware Template (Multi-Agent Cluster)

A decoupled, production-grade cloud middleware stack connecting frontend clients (e.g. FlutterFlow, web, mobile) to Google Cloud ADK Agents hosted on **Agent Platform** (Vertex AI Reasoning Engines) and **Cloud Run**.

---

## Architecture Overview

```
                          ┌───────────────────────────┐
                          │   FlutterFlow / Clients   │
                          └─────────────┬─────────────┘
                                        │ HTTPS / SSE Streams
                                        ▼
                       ┌─────────────────────────────────┐
                       │      agent_gateway service      │
                       │ • Firebase Auth Verification    │
                       │ • $0.05 Wallet Reservation      │
                       │ • Real-time SSE Token Streaming │
                       └───────┬─────────────────┬───────┘
                               │                 │
                Asynchronous   │                 │ Real-Time Streaming
                Pub/Sub Events │                 │ (Vertex AI / Cloud Run)
                               ▼                 ▼
          ┌──────────────────────────┐     ┌─────────────────────────────┐
          │ agent_persistence_worker │     │ Upstream ADK Agents         │
          │ • Atomic Firestore Batch │     │ • Agent Platform (Vertex AI)│
          │ • Non-blocking Ledger    │     │ • Cloud Run Hosted Agents   │
          │ • Wallet Turn Settlement │     └─────────────────────────────┘
          └────────────┬─────────────┘
                       │
                       ▼
          ┌──────────────────────────┐
          │     Google Firestore     │
          └──────────────────────────┘
```

---

## Directory Structure

```
middleware-template/
├── common/                  # Shared Pydantic contracts, ID generators, and security helpers
├── config/                  # Declarative Agent registry and billing model pricing YAMLs
│   ├── agents.dev.yaml      # Development agent registry
│   ├── agents.prod.yaml     # Production agent registry
│   ├── billing.prod.yaml    # Production model rate cards (Gemini 3.7, 3.5, 2.5)
│   └── billing.test.yaml    # Test model rate cards
├── infra/                   # Terraform Infrastructure-as-Code for Cloud Run, Pub/Sub, IAM
│   └── terraform/
├── scripts/                 # Cloud Shell build, rollout, and deployment automation scripts
├── services/                # Core microservices
│   ├── agent_gateway_v3/    # Public streaming proxy, auth, and reservation engine
│   ├── agent_persistence_worker_v3/ # Background Pub/Sub processor and ledger settlement
│   └── billing_api_v3/      # Public Stripe payment and webhook engine
└── tests/                   # 80+ unit tests covering all schemas, billing math, and auth
```

---

## 🛠️ Files to Modify When Deploying a New Project / Cluster

When using this template to create a new cluster (e.g. `v4`, `hr-agents`, `sales-agents`, or a new GCP project), update the following files:

### 1. Project & Cluster Configuration
* **`infra/terraform/variables.tf`** (or `terraform.tfvars`):
  * `project_id`: Your new Google Cloud Project ID (e.g. `my-new-gcp-project`).
  * `region`: Target GCP deployment region (default: `us-central1`).
  * `gateway_service_name`, `worker_service_name`, `billing_api_service_name`: Service names for the new cluster.
  * `pubsub_topic_name`: Pub/Sub topic name for turn events.
  * `firestore_threads_collection`, `firestore_customer_wallets_collection`: Firestore collection names for this cluster.

### 2. Agent Registration
* **`config/agents.prod.yaml`** & **`config/agents.dev.yaml`**:
  * Register the agents for this cluster:
  ```yaml
  my_new_agent:
    agent_id: my_new_agent
    backend: agent_runtime          # or cloud_run_adk
    model: gemini-3.7-flash
    resource_name: projects/YOUR_PROJECT_ID/locations/us-central1/reasoningEngines/YOUR_ENGINE_ID
    region: us-central1
    streaming_enabled: true
    persistence_enabled: true
    auth_policy: firebase
  ```

### 3. Pricing Rate Cards (If Adding Custom Models)
* **`config/billing.prod.yaml`**:
  * Add model token rates if using models outside standard Gemini defaults:
  ```yaml
  models:
    gemini-3.7-flash:
      input_usd_per_million: 0.75
      output_usd_per_million: 3.75
  ```

### 4. Build & Deployment Shell Scripts
* **`scripts/cloudshell_build_middleware.sh`**:
  * Builds ONLY the 3 middleware container images (`agent_gateway`, `agent_persistence_worker`, `billing_api`).
  * Update `PROJECT_ID` and `REGION` at the top of the script.
* **`scripts/cloudshell_deploy_middleware.sh`**:
  * Deploys the middleware infrastructure via Terraform.
  * Update `PROJECT_ID` and `REGION` at the top of the script.

### 5. Deploying ADK Agents (Independent Scripts)
* **`scripts/deploy_cloudrun_streaming_agent.sh`**:
  * Build & deploy any Cloud Run streaming ADK agent with production sizing (2 vCPU, 2 GiB, Concurrency 80):
  ```bash
  bash ./scripts/deploy_cloudrun_streaming_agent.sh <AGENT_NAME> <AGENT_DIRECTORY> <MODEL_NAME>
  ```
* **`scripts/deploy_agentplatform_streaming_agent.py`**:
  * Deploy any local ADK Agent to Vertex AI Agent Platform (Reasoning Engine).

---

## 🚀 Deployment Sequence in Google Cloud Shell

```bash
# 1. Clone your new repo in Cloud Shell
cd ~
git clone <YOUR_NEW_GITHUB_REPO_URL>
cd <REPO_NAME>

# 2. Build Middleware Container Images in Artifact Registry
bash ./scripts/cloudshell_build_middleware.sh

# 3. Deploy Middleware Stack via Terraform / Cloud Run
bash ./scripts/cloudshell_deploy_middleware.sh
```

---

## 🧪 Running Unit Tests Locally

```powershell
python -m pytest tests/unit/ -v
```
All 80 unit tests should pass with 100% success.
