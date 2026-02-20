# 📄 Copyright Notice

> **Intellectual Property © 2026 -- Eduardo Osorio Venegas**\
> All rights reserved.
>
> Document prepared with the support of **Generative Artificial
> Intelligence tools**, under the technical direction and validation of
> the author.

------------------------------------------------------------------------

# 🏗 Azure AI Foundry vs OpenAI SDK -- Architectural Explanation

This document explains the difference between consuming models such as
**gpt-4.1** using:

-   `azure.ai.projects` (AIProjectClient)
-   `openai` SDK (OpenAI)

Both can access the same deployed model, but they operate at **different
architectural layers**.

------------------------------------------------------------------------

# 🔷 Architecture Overview

                              ┌────────────────────────────┐
                              │        Your Application     │
                              │  (Backend / Notebook / API) │
                              └─────────────┬──────────────┘
                                            │
                     ┌──────────────────────┼──────────────────────┐
                     │                                              │
                     ▼                                              ▼
    ┌──────────────────────────────┐                ┌──────────────────────────────┐
    │   azure.ai.projects SDK      │                │        openai SDK            │
    │      (AIProjectClient)       │                │        (OpenAI)              │
    └──────────────┬───────────────┘                └──────────────┬───────────────┘
                   │                                                │
         🔐 Entra ID / Service Principal                    🔑 API Key
                   │                                                │
                   ▼                                                ▼
         ┌──────────────────────────────┐              ┌──────────────────────────────┐
         │      Azure AI Foundry        │              │   Azure OpenAI Endpoint      │
         │        (Project)             │              │   (openai REST endpoint)     │
         └──────────────┬───────────────┘              └──────────────┬───────────────┘
                        │                                              │
                        ▼                                              ▼
                ┌────────────────────┐                        ┌────────────────────┐
                │  Deployments       │                        │  Deployments       │
                │  (gpt-4.1, etc.)   │                        │  (gpt-4.1, etc.)   │
                └────────────────────┘                        └────────────────────┘

------------------------------------------------------------------------

# 🟦 Layer 1 -- `azure.ai.projects` (AIProjectClient)

## 🔐 Authentication

Currently, **AzureKeyCredential is NOT supported**.

`AIProjectClient` expects a `TokenCredential` implementation because it
internally uses:

    BearerTokenCredentialPolicy

For this reason, the correct authentication methods are:

-   Managed Identity
-   `DefaultAzureCredential`
-   Service Principal (Microsoft Entra ID Application)

------------------------------------------------------------------------

## ✅ Working Implementation (Service Principal)

``` python
from azure.identity import ClientSecretCredential
from azure.ai.projects import AIProjectClient

credential = ClientSecretCredential(
    tenant_id="<TENANT_ID>",
    client_id="<CLIENT_ID>",
    client_secret="<CLIENT_SECRET>"
)

client = AIProjectClient(
    endpoint="https://<resource-name>.openai.azure.com",
    credential=credential
)

print("Successfully connected to AI Project")
```

------------------------------------------------------------------------

## 🔑 Required Permissions

The Service Principal must be assigned the following role:

    Azure AI Project Manager

In:

1.  The Foundry resource
2.  The project inside Foundry

This is configured under:

    Access Control (IAM)

Without this role, authentication may succeed but authorization will be
denied.

------------------------------------------------------------------------

# 🟩 Layer 2 -- `openai` SDK

## 🔐 Authentication

Uses API Key:

``` python
from openai import OpenAI

client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key="<API_KEY>"
)
```

Does not require Entra ID or Service Principal.

------------------------------------------------------------------------

# ⚖ Key Technical Differences

 | Feature                 | AIProjectClient               | OpenAI SDK |
| ------------------------ | ----------------------------- | ---------- |
| Level                    | Project                       | Deployment |
| Authentication           | TokenCredential (Entra ID)    | API Key    |
| AzureKeyCredential       | ❌ Not supported currently    | N/A        |
| RBAC/Access control (IAM)| Yes                           | Limited    |
| Resource Management      | Yes                           | No         |
| Simplicity               | Medium                        | High       |
| Enterprise-Oriented      | Yes                           | Can be     |

> ⚠ **Note (20-02-2026):** At the time of adding this comparison table,
> the AzureKeyCredential limitation remains unresolved.

------------------------------------------------------------------------

# 🚀 Recommendation

### For enterprise production environments:

✔ `AIProjectClient` + Managed Identity or Service Principal\
✔ Proper RBAC configuration

### For direct inference and simplicity:

✔ `openai` SDK + API Key

------------------------------------------------------------------------

© Architectural explanation for Azure AI Foundry
