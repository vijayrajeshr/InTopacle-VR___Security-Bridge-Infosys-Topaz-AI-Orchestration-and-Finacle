# InTopacle-VR: Zero-Trust Gateway for Infosys Topaz & Finacle

**InTopacle-VR** (built on the **FinGuard** reference architecture) is an open-source, orchestrator-agnostic zero-trust gateway designed to safely connect generative AI orchestrators (such as **Infosys Topaz**) with transaction-critical core banking platforms (such as **Infosys Finacle**).

---




## 🚨 The Core Problem: Silent Mis-Binding

Integrating fluid, probabilistic language models with rigid, transaction-critical core banking microservices introduces a critical structural vulnerability: **entity parameter confusion**. 

Large Language Models (LLMs) operate on semantic tokens and conversational context rather than fixed database schemas. When natural language requests are mapped into API payloads, the AI must assign extracted numbers to specific fields. Because plain numeric strings lack intrinsic semantic meaning to an LLM, the model frequently transposes or misallocates unique identifiers—such as mistaking an Order ID for an Account ID or Beneficiary ID.

Because core banking microservices execute any syntactically valid JSON payload without error, parameter transposition causes **silent mis-binding**: the banking engine accepts and executes the command, but applies it to the wrong financial entity (e.g., cancelling a standing order instead of freezing an account). Exposing raw banking APIs directly to generative models creates severe operational, financial, and regulatory liabilities under frameworks like GDPR.

---

## 🛡️ Governing Architectural Principle

> **"The LLM proposes meaning. Deterministic code decides truth."**

InTopacle-VR strictly prevents LLMs from constructing or executing final API payloads. The AI is restricted to producing unverified *intent frames* containing masked placeholders. Every identity resolution, authorization check, state verification, and payload execution is handled exclusively by deterministic, rule-based, and fully auditable code.

---

## 🏗️ The 8-Layer Zero-Trust Gateway Architecture

```text
[User Request]
       │
       ▼
1. Pre-LLM Entity Masking (Session Vault)
       │
       ▼
2. Constrained Intent Extraction (Infosys Topaz / LLM)
       │
       ▼
3. Deterministic Entity Resolver (Finacle Inquiry APIs)
       │
       ▼
4. Policy & Authorization Engine (OPA / Cedar)
       │
       ▼
5. Plan Verifier & Dry-Run Simulator
       │
       ▼
6. Database-Fact Confirmation Gate
       │
       ▼
7. Signed Execution Gateway (Finacle Execution APIs)
       │
       ▼
8. Audit & Telemetry Trace (OpenTelemetry)
```

### Layer Breakdown

* **Layer 1: Pre-LLM Entity Masking:** Deterministic regex and format validators strip raw numeric identifiers from user prompts, store them in a secure session-scoped vault, and insert placeholders (e.g., `<ID_1>`). The LLM never sees raw account numbers, preventing transposition or PII leakage.
* **Layer 2: Constrained Intent Extraction:** Exposes Topaz to a curated set of 30 to 50 semantic business intents generated automatically from Finacle OpenAPI specs via Model Context Protocol (MCP). The LLM outputs a strict JSON frame with intent names and role hints (treated purely as unverified guesses).
* **Layer 3: Deterministic Entity Resolver (Core Innovation):** Mentions are resolved via **database queries, not AI inference**. Validates IDs against an **ID Type Registry** (`AccountId`, `OrderId`, `BeneficiaryId`) and queries read-only inquiry APIs scoped strictly to the authenticated customer session. It deterministically binds exact matches, prompts targeted user disambiguation for multiple matches, or safely rejects zero matches.
* **Layer 4: Policy & Authorization Engine:** Enforces declarative policy-as-code (OPA/Rego or Cedar) to verify channel permissions, customer tiers, limits, velocity, and risk classifications (read, reversible write, irreversible write).
* **Layer 5: Plan Verifier & Dry-Run Simulator:** Checks state preconditions (e.g., verifying active loan status) and enforces financial invariants (debit equals credit, currency matching) using Finacle simulation endpoints.
* **Layer 6: Confirmation Gate:** Renders user confirmation screens **strictly from database facts, never LLM text**. Confirmation generates an Ed25519 token bound to the exact payload hash.
* **Layer 7: Signed Execution Gateway:** The sole component holding write credentials to Finacle. It accepts **only** payloads carrying valid cryptographic verification signatures, adding idempotency keys and managing multi-step sagas.
* **Layer 8: Audit & Telemetry:** Generates append-only, replayable OpenTelemetry decision traces across all layers for complete explainability and regulatory compliance.

---

## 📊 Safety Guarantees

| Security Property | Architectural Mechanism |
| :--- | :--- |
| **Zero Mis-Binding Executed** | Typed ID registry + session database queries + payload-hash signature |
| **Zero Cross-Customer Data Leaks** | Pre-LLM masking + session-scoped ownership resolution |
| **No Hallucinated Endpoints** | Curated 30–50 semantic toolset via MCP adapter |
| **Ambiguity Handled Safely** | Zero or multiple matches trigger targeted clarification or rejection |
| **LLM Compromise Containment** | Gateway trusts cryptographic signatures, never model output |
| **Total Auditability** | Append-only, replayable OpenTelemetry decision traces |
