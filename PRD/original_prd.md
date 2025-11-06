# **Product Requirements Document**

# **Multi-Tenant AI Inference Service ("Inference-as-a-service")**

# **Version: 1.9**

## **1\. Introduction**

### **1.1. Problem Statement**

Our organization has numerous teams building AI-powered features. This has led to a decentralized, inefficient, and costly approach to AI infrastructure.

1. **Inefficient GPU Usage:** Multiple teams are building individual, small-scale inference systems on their own GPU resources. This results in significant idle time and duplicated effort, as GPU clusters are expensive to maintain and are not being shared effectively.  
2. **Model Sprawl & Inconsistency:** Teams are using different models and versions, leading to inconsistent user experiences and difficulty in maintaining a standard level of quality or security.  
3. **No Central Control or Visibility:** We have no central way to manage which models are being used, who is using them, or how much they are costing. This makes it impossible to manage budgets, enforce security, or track our total AI spend.  
4. **Complex Management:** We want to manage a mix of our own self-hosted open-source models (Llama, Mistral) and, eventually, external API-based models, all without forcing teams to learn multiple new APIs.  
5. **Lack of Cost Control:** We have no mechanism to limit or track token consumption by team, leading to unpredictable and uncontrolled costs.

### **1.2. Solution: "Inference-as-a-service"**

We will build a single, centralized, multi-tenant AI inference service. This platform will be the single "front door" for all AI inference in the organization.

This service will abstract away the complexity of the underlying infrastructure, providing all teams with:

* A **single, unified API** (in the OpenAI format) to access a wide varietyS of self-hosted open-source models.  
* A **self-service platform** (via UI, API, and CLI) for managing access, API keys, and budgets.  
* A **centralized dashboard** for viewing usage, cost, and performance analytics.

By centralizing, we will achieve massive economies of scale, significantly improve GPU utilization, enforce security and compliance, and provide a consistent, high-quality service for all development teams.

### **1.3. Technical Context**

This service is a "platform-on-a-platform." The core service (all microservices, databases, and web portals) will run on a **CPU Node Pool** within our central Kubernetes cluster. The actual model inference will run on a dedicated **GPU Node Pool** within the *same* cluster. This PRD defines the architecture of the service we are building *on top* of this managed K8s environment.

## **2\. Platform Requirements & Principles**

### **2.1. Roles & Permissions (RBAC)**

The system **MUST** be built with a Role-Based Access Control (RBAC) system from day one. All actions must be permission-gated.

| Role | Scope | Description |
| :---- | :---- | :---- |
| **System Admin** | Global | A platform administrator. Manages the *entire service*, including which models are available, system health, and all organizations. |
| **Org Admin** | Organization | A team lead or manager. Manages *their own organization's* users, API keys, budgets, and can view their org's usage. |
| **Org User** | Organization | A developer or team member. Can manage *their own* API keys and view their personal and org's usage. |

### **2.2. Permissions Matrix (V1.0)**

| Action | System Admin | Org Admin | Org User |
| :---- | :---- | :---- | :---- |
| **System** |  |  |  |
| View System-wide Usage Dashboard | ✅ | ❌ | ❌ |
| Manage Model Mappings (e.g., add llama3) | ✅ | ❌ | ❌ |
| Create / Manage Organizations | ✅ | ❌ | ❌ |
| **Organization** |  |  |  |
| View Org Usage Dashboard | ✅ (any) | ✅ (own) | ✅ (own) |
| Invite / Remove Users from Org | ✅ (any) | ✅ (own) | ❌ |
| Set / Adjust Org Budgets | ✅ (any) | ✅ (own) | ❌ |
| **API Keys** |  |  |  |
| Create API Key | ❌ | ✅ (own) | ✅ (own) |
| View / Revoke *Own* API Keys | ❌ | ✅ | ✅ |
| View / Revoke *All* Org API Keys | ✅ (any) | ✅ (own) | ❌ |
| **Audit Logs** |  |  |  |
| View Org Audit Logs | ✅ (any) | ✅ (own) | ❌ |
| **API Access** |  |  |  |
| Call Inference API (with valid key) | ❌ | ✅ | ✅ |

### **2.3. Platform Access Interfaces**

The platform **MUST** be "API-First." All functionality must be exposed via a REST API, and all other interfaces must consume this API.

1. **Core Management API (REST):** The source of truth. All actions (creating users, revoking keys, setting budgets) are endpoints on this API.  
2. **Web Portal (UI):** A web-based UI that consumes the Core Management API.  
3. **Admin CLI (ias-admin):** A command-line tool that consumes the Core Management API for scripting and System Admin tasks. This tool is the primary way for teams to build their *own* CI/CD or GitOps automation (e.g., a GitHub Action that runs ias-admin user add...).

### **2.4. Architecture Principles**

1. **API-First:** All functionality, including user management, is exposed via a RESTful API. The UI and CLI are "dumb" clients of this API.  
2. **Microservices:** The backend **MUST** be composed of independent, containerized microservices (e.g., User Service, API Router, Analytics Service).  
3. **Stateless:** All services **MUST** be stateless. All state (user data, cache, queues) **MUST** be externalized to dedicated services (Postgres, Redis, RabbitMQ).  
4. **Asynchronous:** The critical inference path must be ultra-low latency. Non-essential tasks (like usage logging) **MUST** be done asynchronously via a message queue.  
5. **Security by Default:** The system must be secure from day one. All access is authenticated, all actions are authorized (RBAC), and keys are hashed.  
6. **Declarative:** All infrastructure (K8s, databases) and application deployments **MUST** be managed declaratively via IaC (Terraform) and GitOps (ArgoCD \+ Helm).

## **3\. Core Features (V1.0)**

### **3.1. Feature: Core Inference API**

* **User Story (Developer):** "As an Org User, I want to send an OpenAI-compatible API request to api.ias.mycompany.com using my API key, specify a model name like llama3, and receive a streaming text response, so that I can build my AI application."  
* **Requirements:**  
  1. The API Router Service must expose a /v1/chat/completions endpoint that is 100% compatible with the OpenAI spec.  
  2. The service must authenticate the request by validating the Bearer Token (API Key).  
  3. The service must authorize the request by checking the organization's status and budget.  
  4. The service must route the request to the correct backend Inference Engine (vLLM) based on the model parameter.  
  5. The service must stream the response back to the client in real-time.

### **3.2. Feature: Organization & User Management**

* **User Story (Admin):** "As a System Admin, I want to use the ias-admin CLI or Web UI to create a new organization, 'Data-Science-Team', and invite a new Org Admin for that team via their email."  
* **User Story (Admin):** "As an Org Admin, I want to use the Web UI to invite a new Org User (developer) to my organization."  
* **User Story (User):** "As an invited user, I want to click a link in my email, set my password, and be successfully logged into the Web Portal as an Org User."

### **3.3. Feature: API Key Management**

* **User Story (Developer):** "As an Org User, I want to log into the Web Portal, navigate to a 'Keys' page, create a new API key for my 'production-app', and see the key *once* so I can copy it to my secrets manager."  
* **User Story (Developer):** "As an Org User, I want to see a list of my existing keys (e.g., 'production-app', 'test-script'), see their last-used date, and be able to revoke them."  
* **User Story (Admin):** "As an Org Admin, I want to see *all* API keys for my organization and be able to revoke *any* of them, in case a developer leaves the team or a key is leaked."

### **3.4. Feature: Usage Dashboard & Budgeting**

* **User Story (Admin):** "As an Org Admin, I want to set a monthly token budget (e.g., 500 million tokens) for my entire organization, so that I can control my costs and prevent runaway usage."  
* **Requirements (Budgeting):**  
  1. The API Router Service **MUST** check an organization's remaining budget before processing an inference request.  
  2. If the budget is exceeded, the API **MUST** return an HTTP 429 Too Many Requests error.  
* **User Story (Admin):** "As an Org Admin, I want to log into the Web Portal and see a dashboard (Feature 3.2) that shows my organization's total token consumption over the last 30 days, broken down by model and by user."  
* **User Story (Developer):** "As an Org User, I want to view the same dashboard to see my *own* token usage."  
* **User Story (System Admin):** "As a System Admin, I want to see a global dashboard showing total token consumption across *all* organizations."

### **3.5. Feature: Audit Logging**

* **User Story (System Admin):** "As a System Admin, I want a read-only log of all major management events (e.g., user\_invited, key\_created, key\_revoked, budget\_changed) so I can see who did what, and when."  
* **User Story (Org Admin):** "As an Org Admin, I want to see the same audit log, but scoped *only* to my organization's events."  
* **Requirements:**  
  1. All Core Management API endpoints that perform a write operation (POST, PUT, DELETE) **MUST** write an entry to an audit\_log table in the State Database (PostgreSQL).  
  2. The entry must contain: timestamp, org\_id, user\_id\_performing\_action, action\_type, and a JSON blob of the request payload.

## **4\. Asynchronous Usage Logging (Critical Flow)**

This is the core flow for tracking usage without impacting API latency.

1. An Org User sends an inference request to the API Router Service.  
2. The API Router Service validates the key and routes the request to the vLLM engine.  
3. The vLLM engine streams the response *back through* the API Router Service.  
4. As the API Router Service streams the response to the user, it counts the input and output tokens.  
5. When the stream is \[DONE\], the API Router Service **immediately** returns the final response to the user.  
6. **Asynchronously**, the API Router Service publishes a "usage log" message (e.g., { "org\_id": "...", "user\_id": "...", "model": "llama3", "input\_tokens": 50, "output\_tokens": 2000 }) to the Usage Logging Queue.  
7. The Usage & Analytics Service (a separate service) consumes this message from the queue and writes it to the Analytics Database (ClickHouse).

This ensures the user's API call is never delayed by a slow database write.

## **5\. Out of Scope (For V1.0)**

* **SSO / SAML / OIDC:** User authentication is userid/password only for V1.  
* **External API Routing:** V1 will *only* route to self-hosted models. The architecture to route to external APIs (OpenAI, Anthropic) is a V2 feature.  
* **Custom Role Creation:** The three roles (System Admin, Org Admin, Org User) are fixed. Orgs cannot create their own custom roles.  
* **Automated Billing / Invoicing:** The platform tracks usage (tokens, cost), but does not generate invoices or handle payments.  
* **Model Fine-Tuning:** The platform is for *inference only*. There is no UI or API for training or fine-tuning models.  
* **GitOps-based Tenant Management:** V1.0 provides an imperative, API-first model (UI/CLI/API) for management. A declarative, GitOps-driven model (where tenants manage their resources via a config file in a Git repo) is a V2 feature. The V1.0 API and CLI are the *enablers* for teams to build their own GitOps workflows if desired.

## **6\. High-Level Architecture & Components**

This defines the concrete microservices and technologies required.

### **6.1. Core Services (Golang Microservices on CPU Nodes)**

1. **User & Org Service:** (Golang)  
   * **Purpose:** Manages all users, organizations, permissions (RBAC), and audit logs.  
   * **API:** POST /org, POST /org/{id}/user, GET /org/{id}/audit\_log  
   * **Database:** Talks *only* to the State Database (PostgreSQL).  
2. **API Router Service:** (Golang)  
   * **Purpose:** The main, high-throughput "front door" for all inference.  
   * **API:** /v1/chat/completions  
   * **Logic:**  
     1. Validates key (hashes and checks PostgreSQL).  
     2. Fetches org budget (from Redis Cache).  
     3. Routes request to the correct Inference Engine.  
     4. Streams response back to the user.  
     5. Publishes usage log to Usage Logging Queue.  
   * **Database:** PostgreSQL (read-only for keys/budgets), Redis (read/write for cache), Usage Logging Queue (write-only).  
3. **Usage & Analytics Service:** (Golang)  
   * **Purpose:** The backend for the dashboards. Consumes usage logs and prepares analytics.  
   * **API:** GET /analytics/org/{id}, GET /analytics/system  
   * **Logic:**  
     1. **(Async)** Consumes from Usage Logging Queue.  
     2. Writes to Analytics Database (ClickHouse).  
     3. **(/analytics API):** Runs fast, aggregated queries against ClickHouse.  
   * **Database:** Usage Logging Queue (read-only), Analytics Database (ClickHouse) (read/write).

### **6.2. Frontend (on CPU Nodes)**

1. **Web Portal:** (React / Vue / Svelte)  
   * **Purpose:** The UI for Org Admins and Users to manage their accounts, keys, and view dashboards.  
   * **API:** A "dumb" client that consumes the Core Management API (all services above).  
   * **Hosting:** Served as a static site from an Nginx container.

### **6.3. Supporting Infrastructure (on CPU Nodes)**

1. **State Database: PostgreSQL**  
   * **Purpose:** Stores persistent, relational data: users, orgs, hashed API keys, budgets, audit logs.  
2. **Usage Logging Queue: RabbitMQ / NATS**  
   * **Purpose:** A message broker to buffer usage logs for asynchronous processing.  
3. **Analytics Database: ClickHouse / TimescaleDB**  
   * **Purpose:** A time-series database optimized for fast, large-scale analytics on usage data.  
4. **Cache: Redis**  
   * **Purpose:** Caches "hot" data for the API Router Service (e.g., org budgets) to minimize latency.

### **6.4. Inference Infrastructure (on GPU Nodes)**

1. **Orchestrator: Kubernetes (K8s)**  
   * **Purpose:** Manages the entire cluster, including CPU (for services) and GPU (for models) node pools.  
2. **Inference Engines: vLLM / TGI**  
   * **Purpose:** The high-performance server that *actually runs* the models.  
   * **Deployment:** Each model (e.g., Llama 3 8B, Llama 3 70B) is deployed as a separate K8s Service (e.g., llama-3-8b.inference.svc.cluster.local).  
   * **API:** Exposes an OpenAI-compatible API *internally* for the API Router Service to call.

## **7\. Non-Functional & Operational Requirements**

### **7.1. Security Requirements**

1. **Authentication:** All access to the Web UI and Core Management API will be authenticated. For V1.0, this will be a userid/password system. SSO (SAML/OIDC) is out of scope for V1.  
2. **API Key Security:** API keys stored in the State Database (6.3.1) **MUST** be hashed (e.g., using SHA-256). The plain-text key is only ever shown to the user once upon creation. The *API Router Service* validates requests by hashing the provided key and comparing it to the stored hash.  
3. **Network Isolation:** The K8s cluster (6.4.1) **MUST** use Network Policies to enforce isolation. The *Inference Engines* (6.4.2) must only accept traffic from the *API Router Service* (6.1.2). The databases (6.3.1, 6.3.3) must only accept traffic from the specific services that need them.

### **7.2. Testing & Quality Requirements**

Our testing philosophy emphasizes high-fidelity, end-to-end (E2E) testing over-reliance on mocks. Tests should be run against real, ephemeral instances of dependencies (like databases) wherever possible.

1. **End-to-End (E2E) Test Harness (Primary Focus):**  
   * A dedicated test harness **MUST** be built (e.g., in Python or Go). This harness is the primary measure of system correctness and will run against a fully deployed environment (e.g., a staging or preview environment).  
   * **No Mocks:** The E2E harness **MUST** interact with the system as a real user would: by calling the public-facing API endpoints (Management API and Inference API) and querying the real databases (PostgreSQL and ClickHouse) to validate results.  
   * **Test Case: "Happy Path" (CI/CD Pipeline):** A quick-running E2E test that **MUST** run in the CI/CD pipeline before any deployment. It validates the core flow:  
     1. Create Org & Org User (via Management API).  
     2. Generate API Key (via Management API).  
     3. Call Inference API with the key (e.g., model="test-model").  
     4. Verify a 200 OK response.  
     5. Query the Analytics Database to confirm the token usage was logged correctly within 10 seconds.  
     6. Clean up all created resources.  
   * **Test Case: "Full Regression" (Nightly):** A comprehensive test suite that runs nightly, covering edge cases:  
     1. All "Happy Path" tests.  
     2. **Auth Failure:** Verify that calls with an invalid/revoked key return 401 Unauthorized.  
     3. **Budget Failure:** Set an Org budget, exceed it with calls, and verify that subsequent calls return 429 Too Many Requests.  
     4. **RBAC Failure:** Verify an Org User *cannot* delete an API key created by another Org User in their org.  
     5. **Audit Log:** Verify that key\_created, key\_revoked, and budget\_changed events appear in the Audit Log (in PostgreSQL).  
2. **Component-Level Tests (No Mocks):**  
   * Instead of traditional integration tests with mocks, services **SHOULD** be tested at their "component" level against real dependencies.  
   * **Example:** The User & Org Service unit tests should run against a real PostgreSQL database spun up in a container (e.g., via Testcontainers). This validates that the SQL queries and data models work correctly against the real database engine, not a mock.  
3. **Unit Tests (Business Logic Only):**  
   * All Golang microservices **MUST** have unit tests, but these should be focused on pure business logic (e.g., parsing, data transformation, validation logic) that does not require I/O.  
   * A high test coverage target (\>80%) is required for this pure-logic code. I/O-heavy code should be covered by Component and E2E tests.

### **7.3. Deployment & Observability (DevOps)**

1. **CI/CD (Continuous Integration & Deployment):**  
   * **CI:** **GitHub Actions** **MUST** be used for all CI pipelines. On every pull request, the pipeline must automatically run unit tests and the "Happy Path" E2E tests.  
   * **Artifacts:** On merge to main, the **GitHub Actions** CI pipeline **MUST** build, tag, and push the service's Docker image to a container registry (e.g., GitHub Container Registry, AWS ECR, or Harbor).  
   * **CD (GitOps):** We **MUST** use a GitOps-based deployment strategy. **ArgoCD** is the recommended open-source tool for this. The GitHub Actions pipeline's final step will be to update a K8s manifest (e.g., a Helm chart's values.yaml) in a separate "config" repository, which ArgoCD will then automatically observe and sync to the Kubernetes cluster.  
2. **Infrastructure as Code (IaC):**  
   * **Cloud Infrastructure:** All core cloud resources (the K8s cluster itself, node pools, managed databases like PostgreSQL) **MUST** be provisioned and managed using **Terraform**.  
   * **Application Infrastructure:** The in-cluster services (e.g., Prometheus, Grafana, our own microservices) **MUST** be managed and deployed using **Helm** charts. These Helm charts will be the "application manifests" that ArgoCD consumes.  
3. **Observability (The "LGTM Stack"):**  
   * The open-source "LGTM" stack (Loki, Grafana, Tempo, Mimir/Prometheus) is the recommended target. All services must be instrumented to support it.  
   * **Logging (L):** All services **MUST** log structured JSON to stdout/stderr. These logs will be collected cluster-wide by **Grafana Loki** (using an agent like Promtail).  
   * **Metrics (M/P):** All services **MUST** expose a /metrics endpoint for **Prometheus** to scrape. Long-term, scalable storage for metrics should use **Grafana Mimir**. All dashboards will be built in **Grafana**.  
   * **Tracing (T):** All services **MUST** support distributed tracing via **OpenTelemetry (OTel)**. Traces will be sent to **Grafana Tempo** for storage and visualization in Grafana.