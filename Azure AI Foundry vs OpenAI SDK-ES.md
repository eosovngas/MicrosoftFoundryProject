# 📄 Derechos de Autor

> **Propiedad Intelectual © 2026 -- Eduardo Osorio Venegas**\
> Todos los derechos reservados.
>
> Documento elaborado con apoyo de herramientas de **Inteligencia
> Artificial Generativa**, bajo dirección y validación técnica del
> autor.

------------------------------------------------------------------------

# 🏗 Azure AI Foundry vs OpenAI SDK -- Arquitectura Explicativa

Este documento explica la diferencia entre consumir modelos como
**gpt-4.1** usando:

-   `azure.ai.projects` (AIProjectClient)
-   `openai` SDK (OpenAI)

Ambos pueden acceder al mismo modelo desplegado, pero operan en **capas
diferentes de la arquitectura**.

------------------------------------------------------------------------

# 🔷 Visión General de Arquitectura

                              ┌────────────────────────────┐
                              │        Tu Aplicación       │
                              │  (Backend / Notebook / API)│
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
         │        (Proyecto)            │              │   (openai endpoint REST)     │
         └──────────────┬───────────────┘              └──────────────┬───────────────┘
                        │                                              │
                        ▼                                              ▼
                ┌────────────────────┐                        ┌────────────────────┐
                │  Deployments       │                        │  Deployments       │
                │  (gpt-4.1, etc.)   │                        │  (gpt-4.1, etc.)   │
                └────────────────────┘                        └────────────────────┘

------------------------------------------------------------------------

# 🟦 Capa 1 -- `azure.ai.projects` (AIProjectClient)

## 🔐 Autenticación

Actualmente **NO funciona con AzureKeyCredential**.

`AIProjectClient` espera una implementación de `TokenCredential` porque
internamente utiliza:

    BearerTokenCredentialPolicy

Por esta razón, la autenticación correcta es mediante:

-   Managed Identity
-   `DefaultAzureCredential`
-   Service Principal (Microsoft Entra ID App)

------------------------------------------------------------------------

## ✅ Implementación Funcional (Service Principal)

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

print("Conexión exitosa al AI Project")
```

------------------------------------------------------------------------

## 🔑 Permisos Requeridos

El Service Principal debe tener asignado el rol:

    Azure AI Project Manager

En:

1.  El recurso Foundry
2.  El proyecto dentro de Foundry

Esto se configura en:

    Access Control (IAM)

Sin este rol, la autenticación será válida pero el acceso será denegado.

------------------------------------------------------------------------

# 🟩 Capa 2 -- `openai` SDK

## 🔐 Autenticación

Usa API Key:

``` python
from openai import OpenAI

client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key="<API_KEY>"
)
```

No requiere Entra ID ni Service Principal.

------------------------------------------------------------------------

# ⚖ Diferencia Técnica Clave

  Característica               AIProjectClient               OpenAI SDK
  ---------------------------- ----------------------------- ------------
  Nivel                        Proyecto                      Deployment
  Autenticación                TokenCredential (Entra ID)    API Key
  AzureKeyCredential           ❌ No soportado actualmente   N/A
  RBAC                         Sí                            Limitado
  Administración de recursos   Sí                            No
  Simplicidad                  Media                         Alta
  Orientado a Enterprise       Sí                            Puede ser

> ⚠ **Nota (20-02-2026):** Al momento de agregar este cuadro, sigue el conflicto de AzureKeyCredential

------------------------------------------------------------------------

# 🚀 Recomendación

### Para producción enterprise:

✔ `AIProjectClient` + Managed Identity o Service Principal\
✔ RBAC configurado correctamente

### Para inferencia directa y simplicidad:

✔ `openai` SDK + API Key

------------------------------------------------------------------------

© Arquitectura explicativa para Azure AI Foundry
