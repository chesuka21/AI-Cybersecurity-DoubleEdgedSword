# 04 · Defensive Guardrails — Defense-in-Depth Architecture for LLM Systems

> **For:** security architects, SOC engineers, platform teams.
> **Purpose:** an actionable, layered architecture for defending LLM-integrated systems — from input guardrails (Prompt Shields, Spotlighting) through core-engine isolation to output DLP, execution controls, and SOC detection.

---

## TL;DR

No single control defends an LLM system. Prompt Shields is a real, generally available classifier for both direct and indirect injection — but it is **probabilistic**, and Microsoft's own guidance prescribes pairing it with **deterministic structural controls** (least privilege, information-flow control, tool-call gates, human-in-the-loop) designed so that "even if some prompt injections are successful, this will not lead to security impacts."[12][16] This document specifies each layer, its honest failure mode, and the controls that hold when the layer above is bypassed.

## 1 · The architecture: six layers, two classes

```text
  PREVENTION (probabilistic — raise attacker cost)      VERIFICATION
 ┌─────────────────────────────────────────────┐
 │  L1 · INPUT GUARDRAILS                       │
 │      Prompt Shields (user + documents) [16]  │
 │      Spotlighting / datamarking [5]          │
 │      Sanitization (invisible Unicode) [12]   │
 ├─────────────────────────────────────────────┤
 │  L2 · CORE ENGINE ISOLATION                  │
 │      Hardened system prompt (no secrets)[9]  │
 │      Trusted/untrusted context separation[5] │
 │      Quotas & rate limits (LLM06)[9]         │
 ├─────────────────────────────────────────────┤   ┌──────────────────────┐
 │  L3 · OUTPUT GUARDRAILS                      │   │ OBSERVABILITY & SOC  │
 │      DLP / PII detection on outputs[9][16]   │──▶│ L6 · DETECTION       │
 │      Exfil-pattern blocking (deterministic)  │   │   Guardrail alerts   │
 ├─────────────────────────────────────────────┤   │   [12] → XDR queue   │
 │  L4 · EXECUTION CONTROLS                     │   │   Plan-drift, critic │
 │      Least-privilege scoped tools[12][13]    │   │   agents, tool-chain │
 │      HITL approval (high-impact actions)[6]  │   │   analysis           │
 │      Information-flow labels (FIDES-style)   │   └──────────┬───────────┘
 ├─────────────────────────────────────────────┘              │
 │  L5 · CONTINUOUS RED TEAM                                  ▼
 │      PyRIT / adversarial simulation in CI-CD[12][13]   Iteration loop
 └────────────────────────────────────────────────────────── back into L1–L4
```

**Design rule:** every probabilistic layer (L1–L3) is assumed bypassable. For every flow it protects, L4 must hold a *deterministic* control that caps the blast radius even when the classifier misses.[12][13]

## 2 · L1 — Input guardrails (Prompt Shields + Spotlighting)

**Prompt Shields** is the classifier layer: a unified API that analyzes LLM inputs and detects **user prompt attacks** (direct jailbreaks: rule overrides, fake conversation mockups, role-play, encoding attacks) and **document attacks** (hidden instructions in third-party content — documents, emails, web pages). It runs at the user-input and tool-response intervention points and reports `attackDetected` per prompt and per document.[16]

**Spotlighting** attacks the root cause instead of the symptom: the fact that LLMs read system prompts, user messages, and retrieved documents as one concatenated stream with no reliable way to tell instructions from data. It transforms untrusted content so it carries a continuous "this is data, not instructions" signal, in three modes (Microsoft Research, *Defending Against Indirect Prompt Injection Attacks With Spotlighting*):[5]

| Mode | Technique | Honest assessment |
|---|---|---|
| **Delimiting** | Wrap untrusted content in special tokens; instruct the model that everything inside is data | Cheapest, weakest — a single attacker payload landing mid-document defeats it |
| **Datamarking** | Interleave a sentinel character throughout the text (continuous signal) | Significantly more effective; survives mid-document payloads |
| **Encoding** | Base64/ROT13 the untrusted content | Strongest separation; near-0% attack success in the paper's tests |

**Measured effect:** on GPT-family models, Spotlighting reduced indirect prompt-injection attack success rates **from over 50% to below 2%** while maintaining task performance.[5] This is the single highest bang-for-buck control in the stack.

**Deployment notes:**

- Deploy Prompt Shields **in block mode on user inputs, annotate mode on documents**; measure false positives on annotate, then graduate to block.[13]
- Enforce it at the **gateway** (e.g., API Management policy) so coverage is fleet-wide across models, not per-app.[13]
- Know the coverage gaps before trusting defaults: some agent guardrail configurations do not support Spotlighting/Groundedness at all, and the agentic guardrail can *override* the model guardrail — leaving the tool-call path unscanned if intervention points are not configured explicitly.[13]
- Sanitize invisible Unicode (zero-width, BiDi controls, Unicode-tag/emoji smuggling) upstream of every classifier — character-injection evasion measured ~72% against Azure Prompt Shields on injections.[13][12]
- Multilingual prompts are a known evasion route; classifiers cover 8 languages with varying quality elsewhere.[13]

## 3 · L2 — Core engine isolation

1. **Hardened system prompt — and no secrets in it.** The system prompt is a real layer but "can be bypassed or degraded by adversarial prompting"; treat it as a soft control, never put end-user or tool input into a system-role message, and assume any business logic in it will leak (LLM08:2026, Hidden Context Exposure). Guardrail configuration, tool definitions, and internal rules do not belong in it.[9][12]
2. **Context separation / prompt partitioning.** Structurally separate trusted instructions from untrusted content (Spotlighting at minimum; information-flow labels where available) so an injected instruction is *less likely to be obeyed*, and provenance is visible to the model.[5][15]
3. **Quotas and rate limiting.** Cap tokens, requests, and cost per tenant/session against Unbounded Consumption (LLM06:2026) — cost exhaustion is a direct P&L risk and an attacker-forced one.[9]
4. **Model provenance.** Approve *which weights* run in the enterprise; supply-chain trust (LLM04) extends to fine-tuning corpora and model artifacts, and uncensored variants of open-weight models are the failure case (see `docs/03_Adversarial_Vectors.md` §5).

## 4 · L3 — Output guardrails (DLP + deterministic exfil blocking)

1. **DLP / PII detection on model outputs.** Every completion is a data channel: scan for PII, source code, credentials, and classified markers before rendering (LLM02:2026). Shadow-AI incidents showed elevated PII (65%) and IP (40%) compromise rates — the same channel applies to your own app's outputs.[8][9]
2. **Deterministic exfiltration-pattern blocking.** The EchoLeak class of attacks is best stopped by *structure*, not classifiers: block or rewrite reference-style Markdown links/images to untrusted domains, enforce a strict CSP on rendered content, and deny outbound requests from rendering surfaces that are not user-initiated.[14][15]
3. **Treat model output as untrusted input.** LLM10:2026 (Improper Output Handling) — encode, validate, and context-neutralize completions before they reach downstream sinks (shell, SQL, HTML, email). Assistants generate insecure code at scale; the sink is where it lands.[9]

## 5 · L4 — Execution controls (the deterministic floor)

This layer is what "actually holds" when every classifier above is bypassed.[12][13]

1. **Least privilege, scoped tools, keyless identity.** An agent connected through a tool server usually inherits broad read/write permissions it does not need. Scope tool permissions to the task, put the agent on a managed identity with only the RBAC it needs, and use short-lived credentials — so a successful injection cannot reach what the identity cannot reach. This is also the direct mitigation set for Excessive Agency (excessive functionality, permissions, autonomy).[6][12]
2. **Break the "lethal trifecta" by design.** An agent that can read untrusted content, access private data, *and* communicate externally is injectable and leakable. Architect so it holds at most two legs: a reader of untrusted email should not be a sender to arbitrary destinations.[13]
3. **Tool-call gates.** Inspect the tool call itself — not just the prompt — for task adherence and risky sequences (plan drift, scope creep, premature actions) before execution; block risky chains deterministically.[12]
4. **Human-in-the-loop for high-impact actions.** Require user approval for consequential writes: deletions, payments, production changes, outbound communications. The 2025 data point that matters: of AI-related breaches, 97% lacked proper access controls — the deterministic layer was simply missing.[6][8]
5. **Information-flow control (FIDES-style trust labels).** Label every piece of content trusted/untrusted, propagate the label through tool calls (most-restrictive-wins), and let sink tools refuse untrusted input — a `send_email` tool that deterministically refuses data originating from an untrusted document, regardless of what the model decided.[13]

## 6 · L5 — Continuous red teaming (measure, don't assume)

- **Run adversarial testing in CI/CD**, not as an annual audit: PyRIT or equivalent frameworks against your own attack corpus; Azure AI Foundry's adversarial simulator for lighter-weight runs.[12][13]
- **Measure ASR (attack success rate) over time** against the cases in `benchmarks/prompt_injection_cases.json`, multi-turn included. An ASR that drifts upward is an incident-in-waiting; an unmeasured ASR is an unknown.
- **Test the bypass classes explicitly:** character injection, encoding tricks, multi-turn escalation, and retrieval-borne injection — the classes measured against production guardrails (EchoLeak's production XPIA bypass; ~72% evasion under character injection)."[14][13]
- **Re-run on change:** any new tool, retrieval source, model, or guardrail config re-opens the attack surface.

## 7 · L6 — SOC detection & response

1. **Route guardrail signals to the SOC as alerts, not logs.** Microsoft's Defender for Cloud AI threat protection turns Prompt Shields detections plus threat intelligence into XDR alerts — with MITRE ATT&CK tactic tags (Privilege Escalation, Defense Evasion, Reconnaissance, Collection, Exfiltration) — covering the model API regardless of how the request arrived.[12]
2. **Alert classes worth tuning:** jailbreak detected/blocked, ASCII smuggling, sensitive-data exposure in AI models, credential theft via AI, system-prompt extraction attempts, suspicious user-agent/Tor-originated access, malware in uploaded models.[12]
3. **Critic agents and tool-chain analysis:** audit inputs/outputs in real time; block risky sequences of tool calls before they execute; detect multi-step agents veering off task (plan drift).[12]
4. **Response playbook:** on detection, (a) block the session, (b) preserve the prompt/context chain as evidence, (c) check what the agent's identity *could* have reached (least privilege makes this short), (d) rotate credentials if tool secrets were in context.

## 8 · Where each OWASP 2026 risk is defended

| OWASP entry (2026) | Defended by |
|---|---|
| LLM01 Prompt Injection | L1 (Shields + Spotlighting) + L4 (tool gates, exfil blocking) + L3 (output DLP)[16][5][12] |
| LLM02 Sensitive Information Disclosure | L3 (DLP/PII on outputs) + L4 (least-privilege retrieval)[9] |
| LLM03 Excessive Agency | L4 (scoped tools, HITL, short-lived credentials)[6] |
| LLM04 Supply Chain | L2 (model provenance) + procurement controls |
| LLM05 Data & Model Poisoning | L2 (corpus governance, lineage) + L5 (adversarial testing) |
| LLM06 Unbounded Consumption | L2 (quotas, rate limits, cost ceilings)[9] |
| LLM07 Misinformation | L2 (grounding/citations) + L6 (drift monitoring)[9] |
| LLM08 Hidden Context Exposure | L2 (no secrets in system prompt) + L5 (extraction attempts in red-team corpus)[9] |
| LLM09 Vector & Embedding Weaknesses | L2 (embedding ACLs, tenant isolation) |
| LLM10 Improper Output Handling | L3 (output validation/encoding before sinks)[9] |

## 9 · Anti-patterns (what NOT to do)

- ❌ **"Our model is filtered, so our agent is filtered."** The agentic guardrail overrides the model guardrail; unconfigured tool intervention points leave the injection path unscanned.[13]
- ❌ **Prompt engineering as the only control.** System prompts "can be bypassed or degraded by adversarial prompting" — they are a soft layer, never the boundary.[12]
- ❌ **Secrets in the system prompt.** Assume it leaks (LLM08); guardrail configs and tool definitions are reconnaissance material.[9]
- ❌ **Single-turn benchmarks as proof of safety.** Representation-level evidence shows the multi-turn generalization gap directly.[3]
- ❌ **Classifier as the security boundary.** EchoLeak bypassed a production classifier; character-injection research measured high evasion rates. Pair probabilistic filters with deterministic controls.[14][13][12]
- ❌ **Annual-only red teaming.** The attack surface changes with every model, tool, and retrieval source; measure ASR continuously.[12][13]

## 10 · Implementation order (if you do only five things)

1. **Spotlighting/datamarking on every RAG or document-ingestion path** — biggest measured effect (50%→<2% attack success), cheapest to implement.[5]
2. **Prompt Shields at the gateway** — block on user inputs, annotate-then-block on documents.[16][13]
3. **Deterministic exfil blocking + strict CSP on rendered model output** — the control that stops the EchoLeak class structurally.[15]
4. **Least-privilege scoped tools + HITL on high-impact actions** — the deterministic floor that caps blast radius; the control missing in 97% of AI-related breaches.[6][8]
5. **Continuous ASR measurement with PyRIT in CI/CD**, routing guardrail alerts to the SOC.[12]

Operational checklist: `templates/llm_hardening_checklist.md`.

## Sources

[3] https://arxiv.org/abs/2507.02956 — A Representation Engineering Perspective on the Effectiveness of Multi-Turn Jailbreaks
[5] https://aka.ms/spotlighting — Microsoft: Introducing Spotlighting in Azure AI Foundry
[6] https://owasp.org/www-project-top-10-for-large-language-model-applications/2_0_vulns/LLM06_ExcessiveAgency — OWASP LLM06:2025 Excessive Agency
[8] https://newsroom.ibm.com/2025-07-30-ibm-report-13-of-organizations-reported-breaches-of-ai-models-or-applications,-97-of-which-reported-lacking-proper-ai-access-controls — IBM Cost of a Data Breach Report 2025 (press release)
[9] https://genai.owasp.org/download/56857 — OWASP Top 10 for LLM Applications 2026 (PDF)
[12] https://arnav.au/2026/06/04/microsofts-ai-prompt-defense-stack-what-actually-works — Microsoft's AI Prompt Defense Stack: What Actually Works
[13] https://nitinksingh.com/posts/prompt-shields-spotlighting-defense-in-depth — Defense-in-Depth for AI Agents on Azure
[14] https://simonwillison.net/2025/Jun/11/echoleak — Breaking down EchoLeak - the First Zero-Click AI Vulnerability Enabling Data Exfiltration from Microsoft 365 Copilot (Simon Willison)
[15] https://arxiv.org/abs/2509.10540 — EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System
[16] https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection — Prompt Shields in Azure AI Content Safety - Microsoft Learn
