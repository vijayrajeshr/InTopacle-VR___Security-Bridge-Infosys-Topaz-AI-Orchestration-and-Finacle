# ADR 0001: Deterministic Entity Resolution Over Probabilistic LLM Parameter Assignment

* **Status:** Accepted
* **Date:** 2026-10-05
* **Deciders:** InTopacle-VR Core Architectural Committee
* **Technical Area:** Intent & Parameter Binding Layer

---

## Context and Problem Statement

When integrating generative AI orchestrators (e.g., Infosys Topaz) with rigid core banking microservices (e.g., Infosys Finacle), Large Language Models (LLMs) frequently suffer from **entity parameter confusion** [32]. LLMs process conversational text probabilistically using semantic tokens [32]. However, core banking endpoints require strictly typed, immutable parameters [32].

Because plain strings of numbers (such as 10-digit identifiers) lack intrinsic semantic meaning to an LLM, models frequently swap or transpose unique identifiers—such as passing an Account ID into an Order ID slot or Beneficiary ID slot [2, 32]. Because banking microservices execute any syntactically valid JSON payload, this leads to **silent mis-binding**, where the core banking system successfully executes an action against the wrong financial entity without throwing a syntax error [2, 32, 33].

---

## Decision Drivers

1. **Zero Execution Error Tolerance:** Core banking systems cannot tolerate silent parameter transposition errors that cause unauthorized account actions or misdirected funds [2, 34].
2. **Strict Type Safety:** Numerical IDs must be validated by mathematical type rules (length, prefix, checksum) before payload construction [5, 23].
3. **Session-Scoped Ownership:** Users/employees must only interact with entities owned by or explicitly authorized within their authenticated session [5, 23].
4. **Auditability and Regulatory Compliance:** Every parameter binding decision must produce an unalterable, replayable decision trace for regulatory requirements (e.g., GDPR) [8, 27].

---

## Considered Alternatives

### Alternative 1: Direct LLM Payload Construction & Tool Calling
* **Description:** Expose Finacle OpenAPI specifications directly to the LLM via tool calling and rely on system prompts and few-shot examples to map user-mentioned IDs to API parameters.
* **Pros:** Simple architecture; no intermediate middleware required.
* **Cons:** Extremely high risk of parameter transposition and API hallucinations [2, 33]. Fails when user phrasing is ambiguous [33]. Exposes thousands of raw endpoints to the LLM context window, increasing latency and cost [4, 19, 33]. Discarded due to catastrophic compliance and operational risk [34].

### Alternative 2: LLM-Based Entity Disambiguation Prompts
* **Description:** Ask the LLM to verify its own parameter choices in a secondary reasoning pass before executing the API call.
* **Pros:** Keeps logic within the AI orchestrator.
* **Cons:** Probabilistic models remain susceptible to systematic context bias and halllucinated confidence [2, 32]. Does not provide deterministic guarantees [2]. Discarded.

---

## Chosen Option: Deterministic Entity Resolver (Layer 3)

We adopt the governing principle: **"The LLM proposes meaning. Deterministic code decides truth"** [2]. 

Under this architectural choice:
1. **Unverified Intent Frame:** The LLM produces only a schema-constrained intent frame containing masked placeholders (`<ID_1>`) and optional `role_hints` (e.g., `account`), which are treated strictly as unverified guesses [2, 4, 15].
2. **ID Type Registry:** An offline, deterministic registry checks candidate IDs against mathematical rules (prefix, string length, Luhn/checksum) to establish branded types (`AccountId`, `OrderId`, `BeneficiaryId`, `TxnRef`) that are mutually non-assignable [5, 23].
3. **Session Ownership Query:** The resolver queries read-only inquiry APIs scoped strictly to the authenticated session's customer [5, 23]. Unowned entities are completely invisible [5, 23].
4. **Deterministic Resolution Outcomes:**
   * **Exactly 1 match:** Binds the verified ID to the payload [5, 23].
   * **Multiple matches:** Stops execution and prompts the user with targeted disambiguation questions [5, 23].
   * **Zero matches:** Safely rejects the request with a non-leaking error message [5, 23].

---

## Consequences

### Positive
* **Elimination of Silent Mis-Binding:** AI parameter guesses never directly reach execution endpoints; only database-verified payloads proceed [2, 5, 8].
* **Strict Session Isolation:** Cross-customer data leaks and unauthorized entity access are prevented by design at the query layer [5, 8, 23].
* **Reduced Model Context & Costs:** Exposing the LLM to 30–50 curated semantic business intents rather than thousands of raw endpoints reduces prompt overhead and API hallucinations [4, 15, 19].

### Negative / Trade-Offs
* **Inquiry API Latency:** Entity resolution requires read-only inquiry queries to Finacle prior to payload execution [5, 11]. *Mitigation:* Implement session-scoped caching with safe Time-To-Live (TTL) bounds [11].
* **Format Ambiguity:** Identifiers sharing identical formats across types require user disambiguation screens [9, 11].

---

## References & Mapping

* **Architecture Layer:** Layer 3 (Deterministic Entity Resolver) [3, 5, 23]
* **Source Alignments:** "FinGuard Reference Architecture" [1, 2, 5], "Vijay's Solution" [2]
