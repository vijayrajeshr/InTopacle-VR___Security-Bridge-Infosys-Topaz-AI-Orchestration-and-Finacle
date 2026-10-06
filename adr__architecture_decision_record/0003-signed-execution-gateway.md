# ADR 0003: Signed Execution Gateway and Cryptographic Payload Tokens

* **Status:** Accepted
* **Date:** 2026-10-05
* **Deciders:** InTopacle-VR Core Security Architecture Team
* **Technical Area:** Core Execution & Perimeter Security

---

## Context and Problem Statement

If an AI orchestrator (or the LLM middleware layer) is compromised by prompt injection, model jailbreaking, or code bugs, an attacker could attempt to forge API calls directly to core banking microservices (Infosys Finacle) [7, 8, 10]. 

To guarantee core safety, the banking core must be completely isolated from unverified AI outputs [7, 8]. There must be a mathematical guarantee that no write operation reaches Finacle unless explicitly verified by deterministic policy checks and authenticated user confirmation [6, 7].

---

## Decision Drivers

1. **Perimeter Isolation of Core Banking:** The core banking execution gateway must refuse all unauthenticated or unverified payloads [7, 8].
2. **Payload Anti-Tampering:** Once a user approves a transaction, no intermediary process can modify any payload parameter (e.g., amount, recipient, account) [7, 26].
3. **Execution Idempotency & Replay Protection:** Prevent double-execution or token replay attacks [7, 10].

---

## Considered Alternatives

### Alternative 1: Direct Session Token Forwarding
* **Description:** Pass the user's OAuth/JWT session token directly to Finacle execution APIs alongside the AI-generated JSON payload.
* **Pros:** Standard web architecture.
* **Cons:** If the AI layer alters payload parameters after user confirmation, Finacle will still accept the call because the session token is valid [7, 26]. Does not protect against parameter tampering or AI mis-binding [2, 7]. Discarded.

---

## Chosen Option: Confirmation Gate & Signed Execution Gateway (Layers 6 & 7)

We implement a two-stage cryptographic execution checkpoint (**Layers 6 and 7**) [3, 7, 26]:

1. **Database-Fact Confirmation Screen (Layer 6):** For state-changing operations, the user sees a confirmation prompt rendered **strictly from read-only database facts, never LLM text** (e.g., *"Cancel standing instruction #SI-4471 to R. Kumar (A/c ending 5544) for ₹5,000?"*) [7, 26].
2. **Payload Hash Signing:** Upon explicit user confirmation, the system generates a cryptographic token (using Ed25519) signed against the **SHA-256 hash of the exact JSON payload** [7, 9, 26].
3. **Exclusive Credentials in Signed Gateway (Layer 7):**
   * The **Signed Execution Gateway** is the *only* component in the system holding API write credentials to Infosys Finacle [7, 26].
   * It inspects incoming requests and recalculates the payload hash [7, 26].
   * If even a single digit or field in the payload was modified after user confirmation, the signature validation fails and the request is instantly dropped [7, 26].
4. **Idempotency & Replay Control:** The gateway attaches unique idempotency keys, manages saga-style compensation for multi-step flows, and logs append-only OpenTelemetry traces [7, 8, 27].

---

## Consequences

### Positive
* **Complete Compromise Isolation:** Even if the LLM layer is 100% compromised, it cannot execute unauthorized actions on Finacle because it lacks Ed25519 signing keys [7, 8].
* **Tamper-Proof Payloads:** Any parameter modification invalidates the payload hash signature instantly [7, 26].
* **Replayable Audit Trail:** OpenTelemetry traces record the signed token, payload hash, and gateway execution response for audit compliance [8, 27].

### Negative / Trade-Offs
* **Key Management Overhead:** Requires secure Key Management Service (KMS) or HashiCorp Vault infrastructure to protect Ed25519 signing keys [7, 9].
* **Confirmation Friction for Writes:** State-changing writes require explicit user confirmation steps [7, 26].

---

## References & Mapping

* **Architecture Layer:** Layer 6 (Confirmation Gate) & Layer 7 (Signed Execution Gateway) [3, 7, 26]
* **Source Alignments:** "FinGuard Reference Architecture" [7, 9], "Layer 1 Detailed Explanation" [26], "Vijay's Solution" [3, 7]
