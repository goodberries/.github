# Self-Healing ML Pipeline — Project Overview

This project implements a production-style, self-healing customer care chatbot using microservices, Retrieval-Augmented Generation (RAG), feedback-driven improvement, and Kubernetes-first deployment. It showcases DevOps + MLOps practices: containerization, GitHub Actions CI/CD, secrets management, persistent data stores, and a nightly training loop.

## High-level Architecture
- Frontend Service (Nginx static site)
- API Gateway (FastAPI): Single entrypoint, CORS, forwards requests to downstream services.
- Bot Service (FastAPI): RAG over Pinecone with AWS Bedrock LLM; calls Interactions Service to log chats.
- Interactions Service (FastAPI): Owns the `interactions` table in Postgres, providing a centralized API for all interaction data.
- Training Service (Python CronJob): Self-healing loop that reads from Interactions Service, generates improved answers, and upserts them to Pinecone.
- ~~Feedback Service~~ (DEPRECATED): Functionality merged into API Gateway and Interactions Service.
- Data Stores:
  - Pinecone (vector DB, SDK v3, index `rag-index`)
  - PostgreSQL (stateful, PVC + hostPath PV, `feedback_db`)
- Infrastructure: Kubernetes Deployments/Services/Secrets/ConfigMaps/Jobs/CronJobs; NodePort for gateway and frontend
- CI/CD: Per-service Docker build/push on GitHub Actions; deploy via `kubectl set image` on a self-hosted runner

## Architecture Diagram
```mermaid
flowchart LR
  U[User] --> F[Frontend (Nginx)]
  F --> G[API Gateway (FastAPI)]
  G --> B[Bot Service (FastAPI)]
  G --> I[Interactions Service (FastAPI)]
  B --> P[(Pinecone Index)]
  B --> L[AWS Bedrock LLM]
  I <--> DB[(PostgreSQL)]
  T[Training Service (CronJob)] --> I
  T --> P
  T --> L

  classDef svc fill:#E8F5FF,stroke:#3B82F6,stroke-width:1px;
  classDef store fill:#F6F8FA,stroke:#999,stroke-width:1px;
  class F,G,B,I,T svc;
  class P,DB store;
```

## Services

### 1) Frontend Service
- Simple chat UI with like/dislike feedback buttons.
- Calls API Gateway; shows feedback UI per response using returned `interaction_id`.

### 2) API Gateway
- FastAPI app with permissive CORS for demo.
- Endpoints:
  - POST `/chat?query=...` → forwards to Bot Service.
  - POST `/feedback` → accepts `{ interaction_id, feedback: 'like'|'dislike' }`, maps to a score, and sends a `PATCH` request to the Interactions Service.
- K8s Service: NodePort (external access for demo).

### 3) Bot Service
- FastAPI with RAG pipeline using:
  - Pinecone index: `rag-index` (Pinecone v3 SDK, API key auth only)
  - BedrockEmbeddings for embeddings
  - Bedrock chat model (small, e.g., Claude 3 Haiku) for answers
- Calls the Interactions Service via a `POST` request to log the chat and retrieve an `interaction_id`.
- Schema fields (key): `interaction_id`, `user_query`, `bot_response`, `feedback` (int), `timestamp`, `processed_for_training` (bool).

### 4) Interactions Service
- Centralized FastAPI service that is the sole owner of the `interactions` table in the PostgreSQL database.
- Provides CRUD endpoints for creating, reading, and updating interactions.
- Decouples all other services from the database, enforcing a clean microservice architecture.

### 5) Training Service (Self-Healing)
- Nightly CronJob (schedule configurable) that:
  1) Reads interactions with negative feedback (`feedback = -1` and `processed_for_training = FALSE`).
  2) Threshold gate (env `FEEDBACK_THRESHOLD`, default 5) to avoid tiny, noisy runs.
  3) De-duplication gate before generation:
     - Similarity search on Pinecone (`k=3`).
     - Judge LLM (fast, Claude 3 Haiku) answers strictly yes/no: "Does retrieved context already answer the question?".
     - If yes → mark processed, skip upsert.
  4) If not sufficient, generate improved answer via teacher LLM (Claude 3 Sonnet) with brand persona grounding.
  5) Upsert a Q&A pair to Pinecone with metadata `{ source: 'self-healing-feedback' }`.
  6) Mark the interaction as processed.

## Data and Infrastructure
- PostgreSQL
  - Deployed in-cluster with PVC and hostPath PV at `/mnt/data/postgres` (no default StorageClass scenario).
  - `postgres-config` ConfigMap provides `pg_hba.conf` to allow pod network access.
  - `feedback_db` ensured by re-initializing data volume on first boot.
- Pinecone
  - SDK v3 (global control plane). Auth is via API key only.
  - Index name: `rag-index`. Managed outside pods via scripts/job.
- Secrets and Env
  - Kubernetes Secrets for: `PINECONE_API_KEY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `POSTGRES_PASSWORD`, `DATABASE_URL`.
  - Pods map specific env names (no `envFrom` drift); region via `AWS_REGION`.

## CI/CD
- GitHub Actions per service:
  - On push to `main`: build Docker image, tag with git SHA, push to registry.
  - Deploy step (self-hosted runner): `kubectl set image` to update the running workloads.
- Images are immutable via SHA tags; simple, fast rollouts.

## End-to-End Flow
1) User sends a message from the Frontend.
2) API Gateway calls Bot Service `/chat`. The Bot Service gets a response from the RAG chain and then calls the Interactions Service to create a new interaction record, returning the `interaction_id`.
3) Frontend shows feedback buttons. When a user clicks, the API Gateway receives the feedback and sends a `PATCH` request to the Interactions Service to update the record.
4) The Training Service periodically polls the Interactions Service for interactions with negative feedback, generates improved answers, and updates Pinecone. It then calls the Interactions Service again to mark the records as processed.

## Important Design Choices
- **Database Decoupling**: Introduced a dedicated `Interactions Service` to act as the sole owner of the `interactions` table. All other services now communicate with it via a REST API instead of directly accessing the database. This enforces a true microservice architecture.
- **Frontend Portability**: The frontend now dynamically determines the API endpoint from the browser's URL (`window.location`) instead of using a hardcoded IP address.

## Current Status (Phases)
- Phase 1 (MVP): Bot + RAG + API Gateway + Frontend on Kubernetes → complete.
- Phase 2 (Feedback + DB): Postgres with PV, feedback-service, UI buttons, DB logging → complete.
- Phase 3 (Self-Healing): Training-service CronJob, persona prompt, de-dup judge, Pinecone upserts → baseline complete.

## Configuration (Key Environment Variables)
- `DATABASE_URL` → Postgres connection string used by services.
- `PINECONE_API_KEY` → Pinecone auth.
- `AWS_REGION` → Bedrock region (e.g., `us-east-1`).
- `FEEDBACK_THRESHOLD` → Min count of negatives to process per training run.
- `PINECONE_INDEX_NAME` → Defaults to `rag-index`.

## Known Improvements (Roadmap)
- Circuit breaker + retries/timeouts for gateway calls; health/readiness probes for external deps.
- Alembic migrations and Repository/Unit-of-Work patterns for DB access.
- OpenTelemetry tracing, structured JSON logs, correlation IDs; Prometheus/Grafana/Loki stack.
- Event-driven training (outbox → SQS/Kafka) with idempotent consumer and review UI.
- Hybrid search (BM25 + vector), reranker, and response cache to boost quality and latency.
- Rate limiting at the gateway; Secrets Manager CSI; IAM Roles for Service Accounts.

## Repository Structure (key folders)
- `interactions-service/` — Owns and manages all interaction data via a REST API.
- `bot-service/` — RAG QA; calls Interactions Service to log chats.
- ~~`feedback-service/`~~ — DEPRECATED and removed.
- `frontend-service/` — Static chat UI (Nginx).
- `training-service/` — Self-healing (cron) pipeline; reads from and updates Interactions Service.
- `infra/` — Postgres PV/PVC/ConfigMap/Service + related manifests
- `data/`, `scripts/`, `app/`, `database/` — utilities and assets used during setup

---
This document summarizes the system as implemented to date and highlights the next steps to strengthen reliability, observability, and retrieval quality for a robust DevOps + MLOps demo.
