# Azure Integration Services - Ops Methodology

## Introduction
This document explains how to design and operationalise Azure Integration Services (Logic Apps, Functions, Service Bus, Event Grid, API Management) in a way that balances governance, developer autonomy, and scalability. By separating infrastructure, application, and configuration concerns, you achieve **clear separation of concerns**—where shared infrastructure is centrally managed, domain teams focus on application logic, and secrets/config remain securely parameterized. This approach ensures **scalable governance, autonomous domain ownership**, and **robust DevOps** practices, paving the way for secure, reliable, and efficient collaboration across multiple teams.

[DevOps deployment for Standard logic apps in single-tenant Azure Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/devops-deployment-single-tenant-azure-logic-apps#separate-concerns)

![Standard Logic App Structure](https://learn.microsoft.com/en-us/azure/logic-apps/media/set-up-devops-deployment-single-tenant-azure-logic-apps/infrastructure-dependencies.png)

## High-level DevOps principles

1. **Infra-as-code vs Application Code**
    - Keep **Infrastructure as Code (IaC)** separate from your **application/integration logic**.
    - Use dedicated IaC templates (Bicep/Terraform/ARM) for provisioning resource groups, shared services (APIM, Service Bus), networking, and security.
    - Place integration flows (Logic Apps definitions, Function code) in a separate codebase or at least a separate folder/repo.

    ![Separation of concerns](https://learn.microsoft.com/en-us/azure/logic-apps/media/devops-deployment-single-tenant/deployment-pipelines-logic-apps.png)


2. **Environment Parity**
    - Maintain consistent deployment flows (Dev → SIT → UAT → Prod) with environment-specific parameter files or Key Vault references.
    - Each environment is deployed from the same templates or code, only differing by parameters (names, connection strings, SKUs, etc.).

3. **Automation-First**
    - No direct portal deployments for developers. Everything is checked into Git, tested, and **deployed via CI/CD pipelines**.
    - Developers can simulate or run local tests (Logic Apps in VS Code, Functions locally with the Azure Functions Core Tools).

4. **Integration Patterns**
    - Treat each integration pattern (e.g., request/reply, pub-sub, orchestration) as a logical unit of code or set of templates.
    - Use consistent patterns for each domain (e.g., a common approach to pub-sub with Event Grid + Logic Apps).

## Organizing Resource Groups Across Multiple Environments
### Two Subscriptions

1. Non-Prod Subscription
    - Hosts Dev, SIT (System Integration Testing), and UAT (User Acceptance Testing).
    - Enables iterative, lower-risk testing of integration apps.
2. Prod Subscription
    - Hosts production resources.
    - Strict RBAC policies and minimal direct developer access.

## The Three-Plane Model in a Single Subscription

### 1. Infrastructure Plane
Manages **base resources** (e.g., Resource Groups, networking, security, shared services) and sets the **governance framework** (RBAC, Azure Policies).

- **Resource Groups**:  
  - **Shared RG** (e.g., `rg-shared-integration`) for cross-cutting assets like API Management, Service Bus, Log Analytics, etc.  
  - **Domain RGs** for each domain + environment, such as `rg-sales-integration-dev`, `rg-sales-integration-prod`, `rg-hr-integration-dev`, etc.  
- **Networking & Security**:  
  - VNet integration if needed, private endpoints, NSGs, etc.  
  - **RBAC**: domain teams have contributor access to their RGs, while shared RG is centrally owned.  
  - **Azure Policy** to enforce naming, tagging, and security baselines.  
- **Infrastructure as Code (IaC)**:  
  - Use Bicep/Terraform/ARM templates.  
  - A dedicated **infrastructure pipeline** deploys shared resources (like API Management, Service Bus) and sets up domain RG scaffolding.
  - One pipeline (or set of pipelines) to stand up the shared RG resources (APIM, Service Bus, Log Analytics) in both subscriptions.
  - Another pipeline to create domain RGs with minimal scaffolding (e.g., Key Vault, function plans).

### 2. Data (Integration/Application) Plane
Holds **domain-specific workflows, code, and integration logic**.

- **Logic Apps & Azure Functions**:  
  - Deployed into each domain RG (e.g., `rg-sales-integration-dev`).  
  - Logic Apps orchestrate workflows; Functions handle custom transformations or compute. 
  - Deployed via domain-specific pipelines to `rg-<domain>-<env>`. For instance, the Sales pipeline deploys to `rg-sales-integration-dev`, `rg-sales-integration-sit`, `rg-sales-integration-uat` in non-prod and `rg-sales-integration-prod` in the prod subscription.
- **Messaging** (Service Bus, Event Grid):  
  - Service Bus can be shared in `rg-shared-integration` with domain-specific queues/topics.  
  - Event Grid Topics/Subscribers as needed for pub-sub patterns.  
- **Observability**:  
  - Send all logs (Logic Apps, Functions, APIM) to a central Log Analytics workspace.  
  - Configure domain-specific alerts (e.g., failure of a “sales-order” Logic App).

### 3. Configuration Plane
Manages **all environment-specific settings, secrets, and parameters** so they’re not hard-coded in code or IaC templates.

- **Azure Key Vault**:  
  - Store secrets, connection strings, certificates.  
  - One Key Vault in the shared RG or domain-specific Key Vaults per environment if stricter isolation is required. Example.: `kv-sales-dev` in `rg-sales-integration-dev`, `kv-sales-prod` in `rg-sales-integration-prod`.
- **Azure App Configuration** (optional):  
  - Central store for non-secret configs or feature flags.  
  - Alternatively, use environment parameter files or pipeline variables.  
- **Parameterization**:  
  - IaC templates, Logic App definitions, and Function apps read environment values from Key Vault or parameter files. 
  - Bicep/Terraform parameter files for each environment
  - Logic App definition parameter files (e.g. `dev.parameters.json`, `sit.parameters.json` etc.)
- **Security & RBAC**:  
  - Use Managed Identities to allow Logic Apps and Functions to retrieve secrets from Key Vault securely.  
  - Automate secret rotation and enforce best practices (e.g., no secrets in code).

---

## Putting it All Together: The Operational Flow

1. **Infra Pipeline** (Infrastructure Plane)  
   1. Deploys or updates shared RG resources (e.g., `apim-enterprise`, `sb-enterprise`, `law-enterprise`).  
   2. Creates domain RGs (e.g., `rg-sales-integration-dev`) and applies policies/tags.  
   3. Defines networking (VNets, private endpoints if needed) and sets up RBAC roles.

2. **Domain Pipeline(s)** (Data + Configuration Planes)  
   1. **Pulls integration code** (Logic Apps JSON definitions, Azure Functions) from the domain’s repo.  
   2. **Fetches environment-specific config** from Key Vault or parameter files (connection strings, API URLs).  
   3. **Deploys Logic Apps / Functions** into `rg-sales-integration-dev`.  
   4. Runs **post-deployment tests** (integration tests, smoke tests).  
   5. Promotes to higher environments (e.g., Test or Prod) following approvals and gating criteria.

3. **Monitoring & Governance**  
   - All logs and metrics flow to a centralized **Log Analytics** workspace in the shared RG.  
   - **Alerts** can be domain-specific (e.g., “Sales domain Logic App failures”) or enterprise-wide (e.g., “Service Bus dead-letter queue length > X”).

---

## Resource Groups in the Non-Prod subscription
To keep it compact and consistent, you might use the following structure in your **non-production subscription**:

```
Non-Prod Subscription: Sub-Integration-NonProd
 └─ rg-shared-integration-nonprod
     ├─ Shared API Management (apim-nonprod)
     ├─ Shared Service Bus (sb-nonprod)
     ├─ Log Analytics (law-nonprod)
     ├─ (Optional) Key Vault for cross-domain testing secrets
     └─ ...
 └─ rg-sales-dev
     ├─ Logic Apps for Sales (e.g., sales-order-dev)
     ├─ Azure Functions (sales-func-dev)
     ├─ Domain Key Vault (kv-sales-dev) if needed
 └─ rg-sales-sit
     ├─ Logic Apps (sales-order-sit)
     ├─ ...
 └─ rg-sales-uat
     ├─ Logic Apps (sales-order-uat)
     ├─ ...
 └─ rg-hr-dev
     ├─ Logic Apps (hr-onboard-dev)
     ├─ ...
 └─ rg-hr-sit
 └─ rg-hr-uat
 └─ rg-finance-dev
 └─ rg-finance-sit
 └─ rg-finance-uat
 └─ rg-marketing-dev
 └─ rg-marketing-sit
 └─ rg-marketing-uat
 └─ rg-it-dev
 └─ rg-it-sit
 └─ rg-it-uat
```

- **Shared Resource Group**: Holds shared integration assets (API Management instance, Service Bus namespace, central Log Analytics).
- **Domain Resource Groups per environment (Dev, SIT, UAT)**:
    - Each domain (Sales, HR, Finance, Marketing, IT) has three RGs.
    - E.g., `rg-sales-dev` hosts only Sales Dev workloads; `rg-sales-sit` for SIT; `rg-sales-uat` for UAT.
- **Tagging & Naming**:
    - For example, Domain=Sales, Environment=SIT, CostCenter=XYZ.
    - Resource naming might be `sales-func-dev` or `sales-order-sit`.

## Resource Groups in the Prod subscription
In the **production subscription**, you set up a parallel structure. Production environments need stricter access, typically with RBAC policies limiting deployments to automated pipelines only(no direct dev user access):

```
Prod Subscription: Sub-Integration-Prod
 └─ rg-shared-integration-prod
     ├─ Shared API Management (apim-prod)
     ├─ Shared Service Bus (sb-prod)
     ├─ Log Analytics (law-prod)
     ├─ (Optional) Key Vault for cross-domain production secrets
     └─ ...
 └─ rg-sales-prod
     ├─ Logic Apps (sales-order-prod)
     ├─ Azure Functions (sales-func-prod)
     ├─ Domain Key Vault (kv-sales-prod) if needed
 └─ rg-hr-prod
 └─ rg-finance-prod
 └─ rg-marketing-prod
 └─ rg-it-prod
```

- **Shared Resource Group**: Production API Management, Service Bus, Log Analytics, etc.
- **Domain Resource Groups per environment (Dev, SIT, UAT)**: Each domain has a single RG for production environment. For example, `rg-sales-prod`

## Example Scenarios

Below are two sample scenarios illustrating how teams might implement the three-plane model within a single subscription.

---

### Example Scenario 1: Sales Order Processing

#### Overview
The **Sales** domain needs to process incoming orders, validate them, and route them to downstream systems. They develop a Logic App (“Sales-Order”) that triggers on new messages posted to a Service Bus topic.

1. **Infrastructure Plane**  
   - **Resource Groups**:  
     - `rg-shared-integration`: Hosts a single Service Bus namespace (`sb-enterprise`), plus Log Analytics (`law-enterprise`), and API Management (`apim-enterprise`).  
     - `rg-sales-integration-dev`: Dev environment for Sales domain.  
     - `rg-sales-integration-prod`: Production environment for Sales domain.  
   - **Networking & RBAC**:  
     - The Sales team has Contributor access to both `rg-sales-integration-dev` and `rg-sales-integration-prod`.  
     - They have Reader access on `rg-shared-integration` to reference shared resources (Service Bus).

2. **Data (Integration/App) Plane**  
   - **Logic App**: `sales-order-dev` triggers on `sb-enterprise/salesorders` topic.  
   - **Azure Function**: `sales-func-dev` used for validating order data.  
   - **Logging**: Both Logic App and Function send logs to `law-enterprise`.

3. **Configuration Plane**  
   - **Azure Key Vault**:  
     - The dev environment uses `kv-sales-dev` inside `rg-sales-integration-dev` to store environment-specific secrets like “DownstreamSystemApiKey.”  
   - **Parameter Files**:  
     - The domain pipeline references `dev.parameters.json` for Logic App definitions, specifying triggers and connections.  
   - **Pipeline Flow**:  
     1. Domain pipeline pulls the Logic App JSON, references `dev.parameters.json`, retrieves secrets from `kv-sales-dev`.  
     2. Deploys the Logic App and Function into `rg-sales-integration-dev`.  
     3. After testing, the same pipeline uses `prod.parameters.json` and `kv-sales-prod` for production deployment.

**Result**: The Sales domain can rapidly iterate on its order-processing workflow without impacting any other domain, but still leverage the centralized Service Bus and logging resources.

---

### Example Scenario 2: HR Employee Onboarding Workflow

#### Overview
The **HR** domain automates employee onboarding via a Logic App that calls external SaaS systems (e.g., Workday) and internal systems (e.g., Active Directory) behind an API Management front end.

1. **Infrastructure Plane**  
   - **Resource Groups**:  
     - `rg-shared-integration` with `apim-enterprise` (exposes onboarding APIs), `sb-enterprise` (optional for async messaging), and `law-enterprise`.  
     - `rg-hr-integration-dev` for Logic App and Function resources in dev, and `rg-hr-integration-prod` for production.  
   - **Networking**:  
     - The HR Logic App uses private endpoints to connect to internal HR systems. The VNet and subnets were defined by the infrastructure pipeline.  
   - **RBAC**:  
     - The HR team can deploy Logic Apps and Functions in their RGs but can’t modify the shared APIM or Service Bus.

2. **Data (Integration/App) Plane**  
   - **Logic App**: `hr-onboard-dev`, triggered by an HTTP endpoint in `apim-enterprise`.  
   - **Azure Function**: `hr-notify-dev`, sends notifications to various Slack channels or email.  
   - **Monitoring**: HR logs flow to `law-enterprise`. The HR team sets up an alert for any Logic App run failure to ensure quick response.

3. **Configuration Plane**  
   - **Secrets**: HR domain Key Vault `kv-hr-dev` stores Workday OAuth tokens, Slack Webhook URLs, etc.  
   - **Parameterization**: In the domain’s repo, `dev.parameters.json` points to dev Slack channels, dev AD endpoints, etc.  
   - **Deployment Flow**:  
     - The HR pipeline fetches config from `kv-hr-dev`, merges with `dev.parameters.json`, and deploys the Logic App and Function.  
     - On success in dev, merges to main triggers the same pipeline with `prod.parameters.json` referencing `kv-hr-prod`.

**Result**: The HR domain can safely manage sensitive SaaS tokens and environment details, while a single enterprise API Management front end remains the shared “front door” for inbound requests.

## Local Development Without Portal Access

### VS Code Tooling
- **Azure Logic Apps (VS Code Extension)**
    - Allows developers to create or edit Logic Apps (especially Standard) locally.
    - They can debug or test triggers locally (with some setup).
- **Azure Functions Core Tools**
    - Enable local debugging of Functions.
    - Emulate triggers (HTTP, Service Bus) locally using the local storage emulator or a test Service Bus.
- **Use Git and CI/CD to deploy**
    - Commit and push changes to Git repo
    - CI/CD pipeline packages and deploys the updated workflows to Azure.


## Project Template
Each domain's integration repo might have a folder structure like:

[Github sample](https://github.com/Azure/logicapps/tree/master/github-sample)

```
/ (root)
 ├─ /logicapps
 │    ├─ sales-order
 |          |- workflow.json
 │    ├─ sales-invoice
 |          |- workflow.json
 │    └─ azure.parameters.json
 │    └─ connections.json
 │    └─ host.parameters.json
 │    └─ dev.parameters.json
 ├─ /functions
 │    ├─ order-validation
 │    │    ├─ host.json
 │    │    ├─ local.settings.json
 │    │    └─ *.cs (if .NET)
 │    └─ invoice-generator
 ├─ /tests
 │    ├─ unit
 │    └─ integration
 ├─ pipeline.yaml
 └─ README.md
```
- **Logic Apps** definitions in /logicapps.
- **Azure Functions** code in /functions.
- Environment parameters in `.json` files or references to Key Vault.
- Pipeline definitions in the root (`pipeline.yaml`) or in the repo's pipeline settings.
---