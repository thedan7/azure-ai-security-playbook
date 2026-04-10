# Identity and Access Patterns for AI Systems

## Managed Identity Assignment Pattern

```mermaid
flowchart TD
    subgraph ManagedIdentities["Managed Identities (User-Assigned)"]
        APIMMI["apim-identity"]
        OrchestratorMI["orchestrator-identity"]
        IngestionMI["ingestion-identity"]
        AgentMI["agent-identity"]
    end

    subgraph AzureOpenAI["Azure OpenAI Service"]
        OAIUserRole["Cognitive Services\nOpenAI User"]
        OAIContribRole["Cognitive Services\nOpenAI Contributor"]
    end

    subgraph AISearch["Azure AI Search"]
        SearchReader["Search Index\nData Reader"]
        SearchContrib["Search Index\nData Contributor"]
    end

    subgraph KeyVault["Azure Key Vault"]
        KVSecretsUser["Key Vault\nSecrets User"]
        KVSecretsOfficer["Key Vault\nSecrets Officer"]
    end

    subgraph Storage["Azure Blob Storage"]
        BlobReader["Storage Blob\nData Reader"]
        BlobContrib["Storage Blob\nData Contributor"]
    end

    APIMMI --> OAIUserRole
    OrchestratorMI --> OAIUserRole
    OrchestratorMI --> SearchReader
    OrchestratorMI --> KVSecretsUser
    IngestionMI --> OAIUserRole
    IngestionMI --> SearchContrib
    IngestionMI --> BlobContrib
    AgentMI --> SearchReader
    AgentMI --> BlobReader
    AgentMI --> KVSecretsUser
```

## User Authentication Flow (Entra ID OAuth 2.0)

```mermaid
sequenceDiagram
    participant User as User / Browser
    participant App as AI Application
    participant EntraID as Microsoft Entra ID
    participant APIM as Azure API Management
    participant Orchestrator as AI Orchestrator

    User->>App: Access AI feature
    App->>EntraID: Redirect to login (MSAL)
    EntraID->>User: MFA challenge
    User->>EntraID: Credentials + MFA
    EntraID-->>App: Authorization code
    App->>EntraID: Exchange code for tokens
    EntraID-->>App: Access token + refresh token
    App->>APIM: API request + Bearer token
    APIM->>EntraID: Validate token (JWKS endpoint)
    EntraID-->>APIM: Token valid; claims
    APIM->>Orchestrator: Forward request + user claims
    Note over Orchestrator: Logs user identity with every LLM call
```

## Privileged Access Management (PIM) Flow

```mermaid
flowchart TD
    Admin["AI Administrator"] --> PIMRequest["Request PIM Activation\n(Azure Portal / PIM API)"]
    PIMRequest --> Justification["Provide Justification\n(ticket number, reason)"]
    Justification --> MFAStep["Re-authenticate with MFA"]
    MFAStep --> Approval["Auto-approved (configured)\nor Manager Approval Required"]
    Approval --> TimeboxedAccess["Time-limited Access Granted\n(e.g., 2 hours)"]
    TimeboxedAccess --> AdminTask["Perform Administrative Task\n(e.g., update Azure OpenAI deployment)"]
    AdminTask --> AccessExpires["Access Automatically Expires"]
    TimeboxedAccess --> AuditLog["All Activations Logged\n(Entra ID Audit Logs + Sentinel)"]

    subgraph ProtectedRoles["Roles Protected by PIM"]
        CogContrib["Cognitive Services Contributor"]
        SearchAdmin["Search Service Contributor"]
        KVAdmin["Key Vault Administrator"]
        MLOwner["Azure ML Owner / Contributor"]
    end
```

## API Key Elimination Strategy

```mermaid
flowchart LR
    subgraph Before["Before: API Key Pattern (Insecure)"]
        AppBefore["Application"] -- "Authorization: api-key abc123" --> OpenAIBefore["Azure OpenAI"]
        note1["API key stored in:\n- App settings\n- Config files\n- Environment vars\n- Code (worst case)"]
    end

    subgraph After["After: Managed Identity Pattern (Secure)"]
        AppAfter["Application\n(Managed Identity)"] -- "Authorization: Bearer <MI token>" --> OpenAIAfter["Azure OpenAI"]
        note2["No credential to manage\nNo credential to rotate\nNo credential to leak\nFull audit trail in Entra ID"]
    end
```
