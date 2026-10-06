# InTopacle-VR (FinGuard) Architecture Diagrams

This document contains native **Mermaid.js** source code diagrams for the **InTopacle-VR Zero-Trust Gateway**. These scripts render natively in GitHub, GitLab, and Markdown visualizers.

---

## Diagram 1: Complete 8-Layer Request Processing Sequence

```mermaid
sequenceDiagram
    autonumber
    actor User as Customer / Employee
    participant L1 as Layer 1: Pre-LLM Masking
    participant Vault as Session Vault
    participant Topaz as Infosys Topaz (LLM)
    participant L2 as Layer 2: Intent Extractor
    participant L3 as Layer 3: Entity Resolver
    participant L4 as Layer 4: Policy Engine (OPA)
    participant L5 as Layer 5: Plan Verifier
    participant L6 as Layer 6: Confirmation Gate
    participant L7 as Layer 7: Signed Gateway
    participant Finacle as Infosys Finacle Core
    participant L8 as Layer 8: OpenTelemetry Audit

    User->>L1: "Check loan default for user account 8877665544"
    Note over L1,Vault: Detects raw ID "8877665544" via Regex
    L1->>Vault: Store mapping ("8877665544" -> "<ID_1>")
    L1->>Topaz: Masked text: "Check loan default for user account <ID_1>"
    
    Note over Topaz,L2: Retrieves curated 30-50 MCP business intents
    Topaz->>L2: Outputs JSON Frame {intent: "check_loan_status", slot: "<ID_1>", role_hint: "account"}
    
    L2->>L3: Pass Intent Frame + Placeholders
    L3->>Vault: Fetch raw value for "<ID_1>" ("8877665544")
    L3->>L3: Validate ID Type Registry (Format, Checksum)
    L3->>Finacle: Read-Only Inquiry (Session-Scoped Ownership Query)
    Finacle-->>L3: Confirmed Match (Account #8877665544, John Doe)
    
    L3->>L4: Pass Verified Payload
    Note over L4: Evaluate declarative policy (OPA/Cedar rules, Channel, Risk Tier)
    L4->>L5: Authorized Intent Frame
    
    Note over L5: Dry-run state preconditions & financial invariants
    L5->>L6: Validated Execution Plan
    
    L6->>User: Display database-fact confirmation screen
    User->>L6: User Confirms Action
    Note over L6: Generate Ed25519 Token bound to SHA-256 Payload Hash
    
    L6->>L7: Submit Signed Payload + Cryptographic Token
    Note over L7: Recalculate Payload Hash & Verify Signature
    L7->>Finacle: Execute API Call (Exclusive Credentials)
    Finacle-->>L7: Execution Success (Data Response)
    
    L7->>User: Present Final Verified Result
    
    Note over L8: Append-only replayable decision trace logged across all layers
    L1-->>L8: Log Masking Event
    L3-->>L8: Log Resolution Outcome
    L4-->>L8: Log Policy Decision
    L7-->>L8: Log Execution Result
```

---

## Diagram 2: Layer 3 Deterministic Entity Resolver Decision Logic

```mermaid
flowchart TD
    Start([Intent Frame from Layer 2]) --> Extract[Extract Placeholder e.g. ID_1 & Role Hint]
    Extract --> VaultFetch[Fetch Raw Value from Session Vault]
    
    VaultFetch --> TypeReg{Layer 3a: ID Type Registry Check<br/>Format, Length, Checksum}
    TypeReg -->|Failed Validation| RejectInvalid[Reject: Invalid ID Format]
    TypeReg -->|Passed| CandidateTypes[Assign Branded Types<br/>AccountId, OrderId, BeneficiaryId]
    
    CandidateTypes --> QueryDB[Layer 3b: Ownership Query<br/>Read-Only Inquiry API Scoped to Authenticated Session]
    
    QueryDB --> MatchCount{Evaluate Database Matches}
    
    MatchCount -->|0 Matches| RejectNotFound[Reject: Unowned or Non-Existent Entity]
    MatchCount -->|>1 Matches| Disambiguate[Prompt User: Targeted Disambiguation Screen]
    MatchCount -->|Exactly 1 Match| BindPayload[Bind Verified ID to Payload<br/>Ignore LLM Role Hint]
    
    Disambiguate --> UserSelect[User Selects Specific Entity]
    UserSelect --> BindPayload
    
    BindPayload --> OutputVerified[Output Typed, Verified Payload to Layer 4]
    
    style Start fill:#EBF8FF,stroke:#3182CE,color:#2B6CB0
    style BindPayload fill:#C6F6D5,stroke:#2F855A,color:#1C4527
    style RejectInvalid fill:#FED7D7,stroke:#C53030,color:#742A2A
    style RejectNotFound fill:#FED7D7,stroke:#C53030,color:#742A2A
    style OutputVerified fill:#9AE6B4,stroke:#276749,color:#1C4527
```

---

## Diagram 3: InTopacle-VR Zero-Trust Security Boundary Map

```mermaid
graph TB
    subgraph UntrustedZone["Untrusted / Probabilistic Zone"]
        User["User / Employee Input"]
        Topaz["Infosys Topaz AI Orchestration<br/>(LLM Context Window)"]
    end

    subgraph MiddlewareZone["FinGuard Zero-Trust Gateway Boundary"]
        direction TB
        L1["Layer 1: Pre-LLM Masking Engine"]
        Vault[("Session Vault<br/>(Ephemeral Encrypted Storage)")]
        L2["Layer 2: Constrained Intent Extraction"]
        L3["Layer 3: Deterministic Entity Resolver"]
        IDReg["ID Type Registry<br/>(AccountId, OrderId, etc.)"]
        L4["Layer 4: OPA / Cedar Policy Engine"]
        L5["Layer 5: Plan Verifier & Dry-Run Simulator"]
        L6["Layer 6: Database-Fact Confirmation Gate"]
        KMS["KMS / Vault<br/>(Ed25519 Signing Keys)"]
    end

    subgraph SecureCoreZone["Isolated Core Banking Zone"]
        L7["Layer 7: Signed Execution Gateway<br/>(Exclusive API Credentials)"]
        Finacle[("Infosys Finacle<br/>Core Banking Engine")]
        L8["Layer 8: OpenTelemetry Audit Trail<br/>(Append-Only Replayable Logs)"]
    end

    User -->|Raw Text| L1
    L1 -->|1a. Store Real Numbers| Vault
    L1 -->|1b. Masked Text ID_1| Topaz
    Topaz -->|2. Unverified Intent Frame| L2
    L2 --> L3
    Vault -.->|3a. Read Real Value| L3
    IDReg -.->|3b. Format Rules| L3
    L3 -->|Verified Payload| L4
    L4 --> L5
    L5 --> L6
    KMS -.->|Sign Payload Hash| L6
    L6 -->|Cryptographic Token + Payload| L7
    L7 -->|Verified Microservice API Call| Finacle
    
    L1 -. Decision Trace .-> L8
    L3 -. Decision Trace .-> L8
    L4 -. Decision Trace .-> L8
    L7 -. Decision Trace .-> L8

    style UntrustedZone fill:#FFF5F5,stroke:#FEB2B2,color:#9B2C2C
    style MiddlewareZone fill:#EBF8FF,stroke:#90CDF4,color:#2C5282
    style SecureCoreZone fill:#F0FFF4,stroke:#9AE6B4,color:#22543D
```
