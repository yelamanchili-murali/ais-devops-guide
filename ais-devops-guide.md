# Azure Integration Services - Ops Methodology

## Introduction
This document explains how to design and operationalise Azure Integration Services (Logic Apps, Functions, Service Bus, Event Grid, API Management) in a way that balances governance, developer autonomy, and scalability. By separating infrastructure, application, and configuration concerns, you achieve **clear separation of concerns**—where shared infrastructure is centrally managed, domain teams focus on application logic, and secrets/config remain securely parameterized. This approach ensures **scalable governance, autonomous domain ownership**, and **robust DevOps** practices, paving the way for secure, reliable, and efficient collaboration across multiple teams.

[DevOps deployment for Standard logic apps in single-tenant Azure Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/devops-deployment-single-tenant-azure-logic-apps#separate-concerns)

![Standard Logic App Structure](https://learn.microsoft.com/en-us/azure/logic-apps/media/set-up-devops-deployment-single-tenant-azure-logic-apps/infrastructure-dependencies.png)

## High-level DevOps principles

| **Principle**           | **Description**                                                                                                                                                                                                                     |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Infra-as-code vs Application Code** | Keep **Infrastructure as Code (IaC)** separate from your **application/integration logic**. Use dedicated IaC templates (Bicep/Terraform/ARM) for provisioning resource groups, shared services, networking, and security. Place integration flows (Logic Apps definitions, Function code) in a separate codebase or folder/repo. ![Separation of concerns](https://learn.microsoft.com/en-us/azure/logic-apps/media/devops-deployment-single-tenant/deployment-pipelines-logic-apps.png) |
| **Environment Parity**  | Maintain consistent deployment flows (Dev → SIT → UAT → Prod) with environment-specific parameter files or Key Vault references. Each environment is deployed from the same templates or code, only differing by parameters (names, connection strings, SKUs, etc.). |
| **Automation-First**    | No direct portal deployments for developers. Everything is checked into Git, tested, and **deployed via CI/CD pipelines**. Developers can simulate or run local tests (Logic Apps in VS Code, Functions locally with the Azure Functions Core Tools). |
| **Integration Patterns**| Treat each integration pattern (e.g., request/reply, pub-sub, orchestration) as a logical unit of code or set of templates. Use consistent patterns for each domain (e.g., a common approach to pub-sub with Event Grid + Logic Apps). |

## Organizing Resource Groups Across Multiple Environments

### Two Subscriptions

| **Subscription** | **Description**                                                                                     |
|-------------------|-----------------------------------------------------------------------------------------------------|
| **Non-Prod**      | Hosts Dev, SIT (System Integration Testing), and UAT (User Acceptance Testing). Enables iterative, lower-risk testing of integration apps. |
| **Prod**          | Hosts production resources. Strict RBAC policies and minimal direct developer access.              |

```
Subscriptions
 ├─ Non-Prod Subscription
 │    ├─ Dev Environment
 │    ├─ SIT Environment
 │    └─ UAT Environment
 └─ Prod Subscription
  └─ Production Environment
```

## The Three-Plane Model in a Single Subscription

| **Plane**                  | **Description**                                                                                                                                                                                                                     | **Key Components**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **1. Infrastructure Plane** | Manages **base resources** (e.g., Resource Groups, networking, security, shared services) and sets the **governance framework** (RBAC, Azure Policies).                                                                         | - **Resource Groups**: Shared RG (e.g., `rg-shared-integration`) for cross-cutting assets like API Management, Service Bus, Log Analytics, etc.; Domain RGs for each domain + environment (e.g., `rg-sales-dev`, `rg-sales-prod`).<br> - **Networking & Security**: VNet integration, private endpoints, NSGs, RBAC for domain teams (Contributor access to their RGs, centrally owned shared RG), Azure Policy for naming, tagging, and security baselines.<br> - **IaC**: Use Bicep/Terraform/ARM templates; pipelines for shared RG resources and domain RG scaffolding. |
| **2. Data (Integration/Application) Plane** | Holds **domain-specific workflows, code, and integration logic**.                                                                                                                                                     | - **Logic Apps & Azure Functions**: Deployed into domain RGs (e.g., `rg-sales-dev`); Logic Apps orchestrate workflows, Functions handle custom transformations or compute; deployed via domain-specific pipelines.<br> - **Messaging**: Service Bus (shared in `rg-shared-integration` with domain-specific queues/topics), Event Grid Topics/Subscribers for pub-sub patterns.<br> - **Observability**: Central Log Analytics workspace for logs; domain-specific alerts (e.g., Logic App failures). |
| **3. Configuration Plane** | Manages **environment-specific settings, secrets, and parameters** to avoid hard-coding in code or IaC templates.                                                                                                                | - **Azure Key Vault**: Store secrets, connection strings, certificates; one Key Vault in shared RG or domain-specific Key Vaults per environment (e.g., `kv-sales-dev`, `kv-sales-prod`).<br> - **Azure App Configuration** (optional): Central store for non-secret configs or feature flags; alternatively, use environment parameter files or pipeline variables.<br> - **Parameterization**: IaC templates, Logic App definitions, and Function apps read environment values from Key Vault or parameter files.<br> - **Security & RBAC**: Managed Identities for secure access to Key Vault; automate secret rotation. |

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


| **Category**               | **Description**                                                                                                                                                                                                                     |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Shared Resource Group**  | Holds shared integration assets (API Management instance, Service Bus namespace, central Log Analytics).                                                                                   |
| **Domain Resource Groups** | Each domain (Sales, HR, Finance, Marketing, IT) has three RGs per environment (Dev, SIT, UAT). For example: `rg-sales-dev` hosts only Sales Dev workloads; `rg-sales-sit` for SIT; `rg-sales-uat` for UAT.                        |
| **Tagging & Naming**       | Use consistent tags like `Domain=Sales`, `Environment=SIT`, `CostCenter=XYZ`. Resource naming conventions might include patterns like `sales-func-dev` or `sales-order-sit`.                                                     |

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

| **Aspect**               | **Details**                                                                                                                                                                                                                     |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Infrastructure Plane** | - **Resource Groups**: <br> `rg-shared-integration` (Service Bus: `sb-enterprise`, Log Analytics: `law-enterprise`, API Management: `apim-enterprise`). <br> `rg-sales-integration-dev` (Dev environment). <br> `rg-sales-integration-prod` (Production environment). <br> - **Networking & RBAC**: <br> Contributor access for Sales team on `rg-sales-integration-dev` and `rg-sales-integration-prod`. <br> Reader access on `rg-shared-integration` for shared resources. |
| **Data Plane**           | - **Logic App**: `sales-order-dev` triggers on `sb-enterprise/salesorders` topic. <br> - **Azure Function**: `sales-func-dev` validates order data. <br> - **Logging**: Logs sent to `law-enterprise`. |
| **Configuration Plane**  | - **Azure Key Vault**: <br> `kv-sales-dev` stores dev secrets like “DownstreamSystemApiKey.” <br> - **Parameter Files**: <br> `dev.parameters.json` specifies triggers and connections. <br> - **Pipeline Flow**: <br> 1. Pulls Logic App JSON, references `dev.parameters.json`, retrieves secrets from `kv-sales-dev`. <br> 2. Deploys Logic App and Function into `rg-sales-integration-dev`. <br> 3. Uses `prod.parameters.json` and `kv-sales-prod` for production deployment. |

**Result**: The Sales domain iterates rapidly on workflows while leveraging centralized resources like Service Bus and logging.

---

### Example Scenario 2: HR Employee Onboarding Workflow

| **Aspect**               | **Details**                                                                                                                                                                                                                     |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Infrastructure Plane** | - **Resource Groups**: <br> `rg-shared-integration` (APIM: `apim-enterprise`, optional Service Bus: `sb-enterprise`, Log Analytics: `law-enterprise`). <br> `rg-hr-integration-dev` (Dev environment). <br> `rg-hr-integration-prod` (Production environment). <br> - **Networking**: <br> Private endpoints connect HR Logic App to internal systems. <br> - **RBAC**: <br> HR team deploys Logic Apps and Functions in their RGs but cannot modify shared resources. |
| **Data Plane**           | - **Logic App**: `hr-onboard-dev` triggered by HTTP endpoint in `apim-enterprise`. <br> - **Azure Function**: `hr-notify-dev` sends notifications to Slack or email. <br> - **Monitoring**: Logs flow to `law-enterprise`, with alerts for Logic App failures. |
| **Configuration Plane**  | - **Secrets**: <br> `kv-hr-dev` stores Workday OAuth tokens, Slack Webhook URLs, etc. <br> - **Parameterization**: <br> `dev.parameters.json` points to dev Slack channels, AD endpoints, etc. <br> - **Deployment Flow**: <br> 1. Fetches config from `kv-hr-dev`, merges with `dev.parameters.json`, and deploys Logic App and Function. <br> 2. On success in dev, merges to main trigger deployment with `prod.parameters.json` and `kv-hr-prod`. |

**Result**: The HR domain securely manages sensitive tokens and environment details while using a shared API Management front end for inbound requests.

## Local Development Without Portal Access

| **Tool**                     | **Description**                                                                                                                                                                                                                     |
|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Azure Logic Apps (VS Code Extension)** | Allows developers to create or edit Logic Apps (especially Standard) locally. They can debug or test triggers locally (with some setup).                                                                                 |
| **Azure Functions Core Tools**         | Enables local debugging of Functions. Emulates triggers (HTTP, Service Bus) locally using the local storage emulator or a test Service Bus.                                                                               |
| **Use Git and CI/CD to deploy**        | Commit and push changes to Git repo. CI/CD pipeline packages and deploys the updated workflows to Azure.                                                                                                                  |


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