# LLM Hardening Checklist — Enterprise Template

> **For:** security engineering teams hardening LLM-integrated applications and AI agent deployments.
> **How to use:** duplicate this file per system/tenant. Assign an **Owner** per section, record **Evidence** (ticket, config link, test run) per item, and re-review on any change to model, tools, or retrieval sources. P1 = blocking for production; P2 = required within 90 days; P3 = hardening.
>
> **Status legend:** ☐ Not started · ◐ Partial · ☑ Implemented · N/A
> **Cross-references:** `docs/04_Defensive_Guardrails.md` (architecture), `docs/02_Threat_Modeling_AI.md` (STRIDE grid), `benchmarks/prompt_injection_cases.json` (regression cases).

| Field | Value |
|---|---|
| System / tenant | *(fill in)* |
| Owner (security) | *(fill in)* |
| Owner (platform) | *(fill in)* |
| Review cadence | Quarterly + on-change |
| Last red-team ASR measurement | *(fill in)* |

---

## 1 · Governance & Shadow AI  *(OWASP: cross-cutting · P1)*

> The control whose absence preceded 97% of AI-related breaches, and the +USD 670K Shadow-AI cost premium.[8]

- [ ] **AI acceptable-use policy published** and acknowledged by all staff (approved tools, prohibited data classes, consequences).
- [ ] **Approved-tool registry** maintained: every sanctioned LLM/agent service, owner, data classification allowed, integration points.
- [ ] **Shadow-AI detection** in place: network/egress monitoring for unmanaged AI endpoints, DLP rule hits, CASB/SSE coverage for LLM categories.
- [ ] **AI governance committee** (or accountable role) named: intake for new AI tools, model approval, exception process.
- [ ] **Data-classification matrix for AI** defined: which data classes may enter which AI systems (none / public / internal / confidential), per tool.
- [ ] **Vendor AI terms reviewed**: training-on-inputs retention, data residency, breach-notification clauses, model change notification.
- **Done when:** a new AI tool cannot be used for confidential data without an intake record; quarterly Shadow-AI scan produces an actionable report.

## 2 · Data handling & DLP  *(OWASP LLM02 · P1)*

- [ ] **DLP treats LLM prompts as a data channel**: scan outbound prompt/API traffic to third-party AI for PII, credentials, source code, and classified markers.
- [ ] **Egress controls on AI endpoints**: unmanaged generative-AI categories blocked or proxied with inspection at the network layer (SSE/SWG policy).
- [ ] **Least-privilege retrieval**: RAG/service accounts return only the records the requesting user could read (ACL filtering at retrieval time, not after).
- [ ] **Secrets hygiene**: no API keys, connection strings, or credentials reachable from AI tool contexts; tool secrets stored in a vault, injected just-in-time.
- [ ] **Output DLP**: PII/secret detection and masking on model outputs before rendering to end users or passing to downstream systems.
- **Done when:** a prompt containing a sample secret/PII is blocked or masked end-to-end in a test run, and retrieval cannot return records outside the user's ACL.

## 3 · Input guardrails  *(OWASP LLM01 · P1)*

> Measured effect of the core technique: Spotlighting reduced indirect injection success from over 50% to below 2%.[5]

- [ ] **Prompt Shields enabled at the gateway** for every model and agent: user-prompt attacks (block mode) and document attacks (annotate → block after FP tuning).[16]
- [ ] **Spotlighting/datamarking on every RAG or document-ingestion path**: untrusted content transformed (delimiting → datamarking → encoding, strongest available) before reaching the model.[5]
- [ ] **Invisible-Unicode sanitization upstream of classifiers**: strip zero-width, BiDi controls, Unicode-tag and emoji smuggling (character-injection evasion measured ~72% against production shields).[13][12]
- [ ] **Encoding-attack coverage**: Base64/ROT13/homoglyph payloads in the regression corpus; classifier responses verified.
- [ ] **Multilingual test coverage**: prompts in supported languages plus one unsupported language, to measure evasion quality outside the trained set.[13]
- [ ] **False-positive baseline measured** on real traffic before graduating document scanning from annotate to block.
- **Done when:** all cases marked `direct_injection` / `indirect_injection` in `benchmarks/prompt_injection_cases.json` are blocked or flagged, and the FP rate on legitimate traffic is documented.

## 4 · Core engine & system prompt  *(OWASP LLM08, LLM06, LLM04 · P1–P2)*

- [ ] **No secrets in the system prompt**: guardrail configs, tool definitions, internal rules treated as leakable (LLM08:2026, Hidden Context Exposure).[9]
- [ ] **Trusted/untrusted separation structurally enforced**: system-role content never composed from end-user or tool input; untrusted content labeled by provenance.[5]
- [ ] **Quotas and rate limits per tenant/session**: tokens, requests, and cost ceilings against Unbounded Consumption (LLM06:2026).[9]
- [ ] **Model provenance & approval**: which weights run in the enterprise is recorded; uncensored/abliterated open-weight variants prohibited without exception.[2][4]
- [ ] **Model change monitoring**: vendor model updates / silent model swaps detected; guardrails re-validated on change.
- [ ] **Supply-chain integrity for artifacts**: fine-tuning corpora and model artifacts verified (hashes, signed manifests) against Data & Model Poisoning (LLM05) and Supply Chain (LLM04).
- **Done when:** a system-prompt extraction attempt from the regression corpus yields no secrets, and quota exhaustion test triggers cost ceilings without service degradation beyond the offending tenant.

## 5 · Output guardrails  *(OWASP LLM02, LLM10 · P1)*

- [ ] **Exfiltration-pattern blocking (deterministic)**: reference-style Markdown links/images to untrusted domains blocked or rewritten; inline and reference link filters applied *equally* (EchoLeak-class gap).[14][15]
- [ ] **Strict CSP on rendered model output**: image/connect origins allow-listed; known-open-redirect proxy domains reviewed and removed (Teams-proxy pattern).[14]
- [ ] **Output treated as untrusted input**: completions encoded/validated before downstream sinks (shell, SQL, HTML, email) — Improper Output Handling (LLM10:2026).[9]
- [ ] **PII/secret masking on outputs** (duplicates §2 — verify both sides of the channel).
- [ ] **Markdown/HTML rendering policy**: which markup the chat UI renders, and to which origins, is documented and enforced.
- **Done when:** the `data_exfiltration` cases in the regression corpus produce zero outbound requests from rendering surfaces, and injected `DROP TABLE`-class output is inert at every sink.

## 6 · Agentic execution controls  *(OWASP LLM03 Excessive Agency · P1)*

> The deterministic floor: what holds when every classifier above is bypassed.[12][13]

- [ ] **Least-privilege scoped tools**: each tool's permissions reduced to task minimum; no broad read/write inheritance from tool servers.
- [ ] **Keyless/managed identity for agents**, with short-lived credentials revoked after use — a successful injection cannot reach what the identity cannot reach.
- [ ] **Lethal-trifecta review per agent**: no agent combines private-data access + untrusted-content exposure + external communication; break one leg by design.[13]
- [ ] **Tool-call gates**: tool calls inspected for task adherence, plan drift, scope creep, and risky sequences before execution; risky chains blocked deterministically.
- [ ] **Human-in-the-loop for high-impact actions**: deletions, payments, production changes, outbound communications require explicit user approval.[6]
- [ ] **Information-flow labels (where available)**: trusted/untrusted propagated through tool calls (most-restrictive-wins); sink tools refuse untrusted input.[13]
- [ ] **Budget/loop ceilings on agent runs**: max steps, max tool calls, max spend per run against runaway loops (Excessive Agency × Unbounded Consumption).
- **Done when:** the `excessive_agency` regression cases (destructive action without approval, permission escalation attempt) are blocked at the gate or require HITL, and the agent identity's blast radius is documented per tool.

## 7 · SOC detection & response  *(P2)*

- [ ] **Guardrail signals routed to the SOC as alerts** (not logs): jailbreak detected/blocked, injection attempts, data exposure, credential theft, system-prompt extraction.[12]
- [ ] **AI threat alerts carry ATT&CK tactic tags** and join the XDR queue alongside endpoint/identity/cloud signals.[12]
- [ ] **Critic/audit agents or tool-chain analysis** for multi-step agents: plan-drift detection, risky tool-chain blocking.
- [ ] **Response playbook for AI incidents**: block session → preserve prompt/context chain as evidence → check identity blast radius → rotate exposed tool secrets.
- [ ] **Abuse-velocity monitoring**: per-user/per-tenant rates of guardrail hits; anomalous escalation (many blocked attempts, then a bypass) alerts.
- **Done when:** a table-top exercise of the response playbook completes with evidence, and the alert classes above are tuned (no silent flood).

## 8 · Continuous validation  *(P1 for P1 items · recurring)*

- [ ] **Red-team ASR measured with own corpus** (PyRIT or equivalent) — single-turn AND multi-turn adaptive attacks; tracked over time, not vendor benchmarks.[12][13][3]
- [ ] **Regression suite in CI/CD**: `benchmarks/prompt_injection_cases.json` executed against every guardrail config change; ASR must not drift upward.
- [ ] **Re-validation triggers defined**: new tool, new retrieval source, model update, guardrail config change → re-run suite.
- [ ] **Multi-turn generalization tested**: circuit-breaker/single-turn defenses explicitly tested against multi-turn escalation cases.[3]
- [ ] **Findings triaged to sections 1–7** with owners and deadlines; exceptions time-boxed and approved.
- **Done when:** last ASR measurement is dated, documented, and within threshold; every guardrail change since is covered by a regression run.

## 9 · Sign-off

| Section | Status | Owner | Evidence | Date |
|---|---|---|---|---|
| 1 · Governance & Shadow AI | | | | |
| 2 · Data handling & DLP | | | | |
| 3 · Input guardrails | | | | |
| 4 · Core engine & system prompt | | | | |
| 5 · Output guardrails | | | | |
| 6 · Agentic execution controls | | | | |
| 7 · SOC detection & response | | | | |
| 8 · Continuous validation | | | | |

**Production gate:** sections 1–6 must reach ☑ (or documented, time-boxed exception) before an LLM/agent system handles confidential data.

## Sources

[2] https://arxiv.org/abs/2406.11717 — Refusal in Language Models Is Mediated by a Single Direction
[3] https://arxiv.org/abs/2507.02956 — A Representation Engineering Perspective on the Effectiveness of Multi-Turn Jailbreaks
[4] https://proceedings.neurips.cc/paper_files/paper/2024/file/f545448535dfde4f9786555403ab7c49-Paper-Conference.pdf — Refusal Direction (NeurIPS 2024)
[5] https://aka.ms/spotlighting — Microsoft: Introducing Spotlighting in Azure AI Foundry
[6] https://owasp.org/www-project-top-10-for-large-language-model-applications/2_0_vulns/LLM06_ExcessiveAgency — OWASP LLM06:2025 Excessive Agency
[8] https://newsroom.ibm.com/2025-07-30-ibm-report-13-of-organizations-reported-breaches-of-ai-models-or-applications,-97-of-which-reported-lacking-proper-ai-access-controls — IBM Cost of a Data Breach Report 2025 (press release)
[9] https://genai.owasp.org/download/56857 — OWASP Top 10 for LLM Applications 2026 (PDF)
[12] https://arnav.au/2026/06/04/microsofts-ai-prompt-defense-stack-what-actually-works — Microsoft's AI Prompt Defense Stack: What Actually Works
[13] https://nitinksingh.com/posts/prompt-shields-spotlighting-defense-in-depth — Defense-in-Depth for AI Agents on Azure
[14] https://simonwillison.net/2025/Jun/11/echoleak — Breaking down EchoLeak - the First Zero-Click AI Vulnerability Enabling Data Exfiltration from Microsoft 365 Copilot (Simon Willison)
[15] https://arxiv.org/abs/2509.10540 — EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System
[16] https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection — Prompt Shields in Azure AI Content Safety - Microsoft Learn
