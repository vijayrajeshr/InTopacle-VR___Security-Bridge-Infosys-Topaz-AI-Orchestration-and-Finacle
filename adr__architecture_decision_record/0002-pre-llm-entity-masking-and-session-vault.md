# ADR 0002: Pre-LLM Entity Masking and Session Vault

* **Status:** Accepted
* **Date:** 2026-10-05
* **Deciders:** InTopacle-VR Core Security Architecture Team
* **Technical Area:** Data Privacy & Pre-Processing Gateway

---

## Context and Problem Statement

Sending raw financial numbers (e.g., 10-digit account numbers, credit card numbers, national identifiers) directly to Large Language Models (LLMs) introduces severe data privacy, compliance, and security risks [3, 13, 34]. 

Prompt injection attacks, conversational context leakage across multi-tenant sessions, and model training/logging pipelines can expose raw customer account data [3, 13]. Furthermore, when an LLM processes raw numeric strings in context, it frequently transposes or invents digits during string generation [3, 13].

---

## Decision Drivers

1. **Zero Raw Financial ID Exposure to LLM:** Raw account numbers must never enter the LLM prompt context window [3, 13].
2. **Regulatory Compliance:** Strict compliance with GDPR, PCI-DSS, and regional banking confidentiality regulations [8, 34].
3. **Immutability of Numerical Identifiers:** Prevent the LLM from modifying, hallucinating, or transposing digits during text parsing [3, 13].

---

## Considered Alternatives

### Alternative 1: Post-Generation Anonymization
* **Description:** Allow the LLM to process raw numbers in context, then sanitize the generated output before presenting it to the user.
* **Pros:** Simpler pre-processing pipeline.
* **Cons:** Completely fails to protect the LLM context window. Raw identifiers remain exposed to model provider logs, prompt injection attacks, and hallucination during reasoning passes [3, 13]. Discarded.

### Alternative 2: Irreversible Tokenization / Hashing
* **Description:** Hash identifiers using SHA-256 before sending them to the LLM.
* **Pros:** Cryptographically secure.
* **Cons:** Prevents downstream deterministic core banking execution systems from reconstructing the original payload without complex database lookups for every transient mention [3, 13]. Discarded in favor of session-vault placeholders.

---

## Chosen Option: Pre-LLM Entity Masking & Session-Scoped Vault (Layer 1)

We implement a deterministic pre-processing pipeline (**Layer 1: Pre-LLM Entity Masking**) prior to calling the AI orchestrator [3, 13]:

1. **Regex & Format Scanning:** Every incoming raw user message is scanned by deterministic regular expressions and string format validators for numeric identifiers [3, 13].
2. **Placeholder Substitution:** Identified numerical strings are stripped from the prompt and replaced with session-scoped, sequential placeholders (e.g., `<ID_1>`, `<ID_2>`) [3, 13, 15].
3. **Session-Scoped Vault:** The mapping between raw values and placeholders is stored in an encrypted, ephemeral, session-scoped vault (e.g., Redis with memory encryption) [3, 9, 13].
4. **LLM Blind Spot:** The LLM receives only sanitized text containing placeholders (`"Check loan status for account <ID_1>"`), making it impossible for the model to see, transpose, leak, or hallucinate real account numbers [3, 13, 29].

---

## Consequences

### Positive
* **Complete LLM Blindness to Sensitive Numbers:** Real account numbers never enter the LLM's context window, eliminating context leaks and prompt extraction risks [3, 13].
* **Prevention of Digit Transposition:** Because the LLM handles string tokens like `<ID_1>`, it cannot transpose digits within an account number [3, 13].
* **Privacy Compliance:** Meets GDPR and PCI-DSS requirements regarding PII isolation [8, 34].

### Negative / Trade-Offs
* **Regex Maintenance Overhead:** Requires maintaining robust regex patterns and format validators for bank-specific ID formats [1, 11, 13].
* **Vault Expiration Management:** Session vaults must be tightly coupled to authenticated user session lifetimes and securely wiped upon logout or timeout [3, 13].

---

## References & Mapping

* **Architecture Layer:** Layer 1 (Pre-LLM Entity Masking) [3, 13]
* **Source Alignments:** "FinGuard Reference Architecture" [3], "Layer 1 Detailed Explanation" [13], "Vijay's Solution" [3]
