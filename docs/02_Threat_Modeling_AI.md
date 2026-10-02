# 02 · Threat Modeling for AI Systems — STRIDE + OWASP

> **For:** security architects, threat-modeling facilitators, red/blue teams.
> **Purpose:** a repeatable method for threat-modeling LLM-integrated systems, combining Microsoft's classic **STRIDE** taxonomy with the **OWASP Top 10 for LLM Applications (2026)** and the **NIST AI 100-2** adversarial-ML lifecycle taxonomy.

---

## TL;DR

Classic threat models assume deterministic components with structured interfaces. LLM systems break both assumptions: their core logic is **probabilistic**, their interface is **natural language**, and their context window **concatenates trusted and untrusted text into a single stream** the model cannot reliably separate.[5] This document provides: (1) a reference architecture for an LLM system, (2) a STRIDE-per-element mapping, (3) an OWASP-2026 cross-map, and (4) three worked threat scenarios.

## 1 · Why LLMs need their own threat-model pass

Three properties of LLM systems invalidate naive reuse of STRIDE-for-webapps:

1. **No intrinsic data/instruction separation.** When a system prompt, a user message, and retrieved documents are concatenated, the LLM has no reliable way to know which bits are instructions and which are data — the root enabler of prompt injection.[5][14]
2. **Probabilistic enforcement.** Guardrails are classifiers; Microsoft's MSRC explicitly calls indirect prompt injection "an inherent risk that arises from the probabilistic language modelling, stochastic generation, and linguistic flexibility of modern LLMs," with deterministic detection still an open research problem.[12]
3. **Agency blurs the trust boundary.** Tool calls, memory, and multi-step planning mean a model that was tricked can *act* — not just speak — which is why the OWASP GenAI project moved Excessive Agency to #3 in 2026 and moved agentic risk to a dedicated Agentic (ASI) Top 10.[9][10]

## 2 · Reference architecture (the elements to threat-model)

```text
                    ┌──────────────────────────────────────────────────┐
                    │                 CORPORATE TRUST ZONE             │
 ┌──────────┐       │  ┌────────────┐   ┌─────────────┐   ┌─────────┐  │
 │  User /  │──────▶│  │ 1. Input   │──▶│ 2. LLM Core │◀──│ 3. RAG / │─┼── external
 │ Client   │       │  │  Guardrail │   │  (engine)   │   │  Tool    │  │  content
 └──────────┘       │  │  (shields, │   │  system     │   │  layer   │  │  (email,
                    │  │  DLP-in)   │   └──────┬──────┘   └────┬─────┘  │  docs, web)
                    │  └────────────┘          │               │        │
                    │                   ┌──────▼──────┐  ┌─────▼─────┐  │
                    │                   │ 4. Output   │  │ 5. Tool   │  │
                    │                   │  Filter     │  │ Execution │  │
                    │                   │  (DLP-out)  │  │ (agent    │  │
                    │                   └──────┬──────┘  │  runtime) │  │
                    │                          │         └─────┬─────┘  │
                    └──────────────────────────┼───────────────┼────────┘
                                               ▼               ▼
                                        ┌──────────┐   ┌────────────────┐
                                        │ 6. User  │   │ 7. Downstream  │
                                        │ Response │   │ systems (DB,   │
                                        └──────────┘   │ email, SaaS)   │
                                                       └────────────────┘
```

Each numbered element gets its own STRIDE row. Elements 3 and 5 are where untrusted content enters the trust zone; element 5 is where the *lethal trifecta* (private-data access + untrusted-content exposure + external communication) completes if not designed against.[13]

## 3 · STRIDE-per-element mapping

| Element | S — Spoofing | T — Tampering | R — Repudiation | I — Information Disclosure | D — Denial of Service | E — Elevation of Privilege |
|---|---|---|---|---|---|---|
| **1 · Input guardrail** | User impersonates another tenant/role in prompt framing | Attempted bypass/modification of shield config via prompt | Unsigned/untokened requests can be denied by sender | Guardrail logs leak prompt content to logs/SIEM with sensitive data | Guardrail becomes throughput bottleneck (attacker-forced re-scans) | N/A (pre-model) |
| **2 · LLM Core** | Model plays a spoofed persona (role-play jailbreaks, "you are DAN") | Context tampering: injected instructions alter model behavior; conversation mockups fake prior turns[16] | No attribution of model decisions (model output has no provenance) | System-prompt / hidden-context leakage (LLM08:2026)[9]; training-data memorization | Unbounded consumption: token/cost exhaustion (LLM06:2026)[9] | Model instructed to violate its operating constraints (jailbreak) |
| **3 · RAG / tool context** | Attacker spoofs a trusted source (plants email/doc that retrieval trusts) | Indirect prompt injection: hidden instructions in retrieved content[14][16] | Retrieved content cannot be traced to its origin without provenance labels | Cross-context leakage: one source's data referenced in another's answer (EchoLeak's "LLM Scope Violation")[15] | Attacker floods the corpus to poison retrieval or exhaust embedding budget | Retrieved instructions effectively become system instructions |
| **4 · Output filter** | Model output spoofed as system/UI message | Malicious content smuggled past filters (ASCII/Unicode-tag smuggling, homoglyphs, reference-style Markdown)[12][14] | Filtered/blocked outputs not logged consistently | PII/IP exfiltration via model output (LLM02)[9] | Output filter forced into expensive re-classification | Exfiltration channels in "allowed" output (Markdown image auto-fetch)[15] |
| **5 · Tool execution** | Agent spoofed into calling the wrong tool (task adherence drift)[12] | Tool arguments tampered via injected instructions | Actions taken by agents lack human sign-off | Over-broad tool reads expose data beyond task scope | Runaway agent loops exhaust quota/budget (LLM06)[6][9] | Excessive agency: excessive functionality, permissions, or autonomy[6] |
| **6 · User response** | Phishing-grade content in model answers | Rendered Markdown executes (image auto-fetch, link tricks)[15] | N/A | Sensitive data rendered to unauthorized viewers | N/A | N/A |
| **7 · Downstream systems** | Agent's credentials used by attacker after injection | Writes to production from injected instructions | Unapproved production changes attributed to "the AI" | Database/SaaS records exposed via over-privileged identities[6] | Agent-triggered mass operations exhaust systems | Blast radius = whatever the agent's identity can reach |

## 4 · OWASP Top 10 (2026) cross-map to STRIDE elements

| OWASP entry (2026) | Primary STRIDE elements | Architecture element(s) | One-line mitigation anchor |
|---|---|---|---|
| LLM01 Prompt Injection | T, E, I | 1, 2, 3, 4 | Prompt Shields + Spotlighting + deterministic exfil blocking[5][16][12] |
| LLM02 Sensitive Information Disclosure | I | 2, 4, 5 | DLP on outputs; least-privilege retrieval; PII detection[9] |
| LLM03 Excessive Agency | E, T, D | 5, 7 | Least privilege, scoped tools, HITL approval for high-impact actions[6] |
| LLM04 Supply Chain | T, S | 2, 3 | Verify model/artifact provenance (model identity assurance) |
| LLM05 Data & Model Poisoning | T | 2, 3 | Data lineage, fine-tuning governance, content filtering of training corpora |
| LLM06 Unbounded Consumption | D | 2, 5 | Rate limiting, per-tenant quotas, cost ceilings[9] |
| LLM07 Misinformation | T, S | 2, 6 | Grounding, retrieval citations, confidence signaling[9] |
| LLM08 Hidden Context Exposure | I | 2 | Minimize system-prompt secrets; treat prompts as non-confidential[9] |
| LLM09 Vector & Embedding Weaknesses | T, I | 3 | Access control on embeddings, multi-tenant isolation, chunk ACLs |
| LLM10 Improper Output Handling | T, E | 4, 7 | Treat model output as untrusted input: encode/validate before sinks[9] |

Framework cross-mappings condensed from the OWASP 2026 edition's own appendix (which maps to MITRE ATLAS, ATT&CK, CWE, NIST AI RMF/600-1, and the OWASP Agentic ASI Top 10) and the 2025 entry documentation.[9][10][7]

## 5 · The NIST adversarial-ML lifecycle dimension

NIST AI 100-2 organizes the AML landscape by **lifecycle stage** (training vs. deployment/inference), **attacker goals** (availability, integrity, privacy), and **attacker capabilities/knowledge** (black-box vs. white-box, poisoning vs. evasion vs. extraction).[11] Use it as a second axis over the STRIDE grid:

- **Deployment/inference attacks** (evasion, prompt injection, jailbreaks) dominate LLM operational risk — this is where docs 03 and 04 concentrate.[11]
- **Training-stage attacks** (data/model poisoning, backdoors) are rarer but harder to remediate; supply-chain trust applies to fine-tuning corpora and model artifacts.[11]
- **Privacy attacks** (membership inference, memorization) bound what can be shipped in fine-tuned models — pair with LLM02.

## 6 · Worked threat scenarios

### Scenario A — Shadow AI insider exfiltration (element 3 → 6, no agent)

1. Employee pastes proprietary source code / customer PII into an unmanaged third-party LLM to "save time." (20% of organizations report a Shadow-AI breach; the data at risk is multi-environment.[8])
2. The third-party service may retain, train on, or expose the content — confidentiality is lost at paste time, outside all corporate telemetry.
3. Detection failure mode: no Shadow-AI policy or detection exists (the case in 63% of organizations[8]), so step 1 repeats until a breach surfaces with the +USD 670K Shadow-AI premium.[8]
4. Controls that break the chain: egress policy on AI endpoints, DLP treating prompts as a data channel, approved-tool registry, training. (`templates/llm_hardening_checklist.md` §1–2.)

### Scenario B — Zero-click agentic exfiltration (the EchoLeak pattern: elements 3 → 2 → 4 → 6)

1. Attacker emails a benign, human-worded message to a victim whose copilot will later retrieve it — phrased for humans, so AI-directed classifiers don't fire.[14][15]
2. Victim asks an unrelated question; retrieval (element 3) pulls the attacker's email into context. Hidden instructions coerce a cross-context data reference (element 2).15
3. Output filter misses the exfil channel: reference-style Markdown links/images were unfiltered where inline links were redacted (element 4).[14]
4. The chat UI auto-fetches the Markdown image; the URL routes through an allow-listed Microsoft Teams proxy domain to the attacker's server (element 6). Zero clicks; CVE-2025-32711, CVSS 9.3.[14][15]
5. Controls that break the chain: deterministic blocking of known exfil patterns (reference-style links, untrusted outbound requests), strict CSP, provenance-based access control on retrieval, DLP on outputs.[15][12]

### Scenario C — Multi-turn jailbreak of a guarded assistant (elements 1 → 2, repeated)

1. Attacker opens with a benign question adjacent to the target task; each turn references the model's own prior answers to escalate ("Crescendo").[1]
2. Single-turn guardrails (element 1) scan each prompt in isolation; representation-level research shows safety-aligned models represent Crescendo responses as *benign* as turns accumulate, and single-turn defenses like circuit breakers generalize poorly to multi-turn attacks.[3]
3. Model produces policy-violating content in small, individually-innocuous steps — each step may pass a naive per-turn filter.[1]
4. Controls that break the chain: conversation-history-aware filtering (pass entire turns to the detector), behavioral-override detection (Skeleton Key pattern), output-side policy classifiers, abuse-velocity monitoring.[1][12]

## 7 · How to run this threat model in your organization

1. **Inventory first:** every LLM integration (approved or not), its elements 1–7, and its data flows. Include Shadow AI: ask which teams use which tools.
2. **STRIDE the grid:** one session per system, one row per element; rate each cell (Mitigated / Partial / Gap) with evidence.
3. **Cross-check OWASP:** for every "Gap," name the OWASP entry and pull its mitigation list.[9]
4. **Define the deterministic floor:** for every flow where injection or jailbreak is plausible, decide what *structural* control holds when the classifier misses (least privilege, HITL, egress rules, exfil-pattern blocking).[12][13]
5. **Red-team to evidence:** test with the cases in `benchmarks/prompt_injection_cases.json`; measure ASR before and after control rollouts.
6. **Re-run on change:** any new tool, model, or retrieval source re-opens elements 3 and 5.

## Sources

[1] https://arxiv.org/abs/2404.01833 — The Crescendo Multi-Turn LLM Jailbreak Attack
[3] https://arxiv.org/abs/2507.02956 — A Representation Engineering Perspective on the Effectiveness of Multi-Turn Jailbreaks
[5] https://aka.ms/spotlighting — Microsoft: Introducing Spotlighting in Azure AI Foundry
[6] https://owasp.org/www-project-top-10-for-large-language-model-applications/2_0_vulns/LLM06_ExcessiveAgency — OWASP LLM06:2025 Excessive Agency
[7] https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025 — OWASP Top 10 for LLM Applications 2025
[8] https://newsroom.ibm.com/2025-07-30-ibm-report-13-of-organizations-reported-breaches-of-ai-models-or-applications,-97-of-which-reported-lacking-proper-ai-access-controls — IBM Cost of a Data Breach Report 2025 (press release)
[9] https://genai.owasp.org/download/56857 — OWASP Top 10 for LLM Applications 2026 (PDF)
[10] https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/blob/main/2026/README.md — OWASP LLM Top 10 2026 - canonical source (GitHub)
[11] https://csrc.nist.gov/pubs/ai/100/2/e2023/final — NIST AI 100-2 E2023: Adversarial Machine Learning Taxonomy
[12] https://arnav.au/2026/06/04/microsofts-ai-prompt-defense-stack-what-actually-works — Microsoft's AI Prompt Defense Stack: What Actually Works
[13] https://nitinksingh.com/posts/prompt-shields-spotlighting-defense-in-depth — Defense-in-Depth for AI Agents on Azure
[14] https://simonwillison.net/2025/Jun/11/echoleak — Breaking down EchoLeak - the First Zero-Click AI Vulnerability Enabling Data Exfiltration from Microsoft 365 Copilot (Simon Willison)
[15] https://arxiv.org/abs/2509.10540 — EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System
[16] https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection — Prompt Shields in Azure AI Content Safety - Microsoft Learn
