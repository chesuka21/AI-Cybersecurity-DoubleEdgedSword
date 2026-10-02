# 03 · Adversarial Vectors — Jailbreaking, Prompt Injection & Uncensored Models

> **For:** red teamers, detection engineers, and security architects.
> **Scope:** technical analysis of how LLM safety alignment fails. Documented at the **conceptual and architectural level only** — no operational exploit code, no target-specific payloads.

---

## TL;DR

The offensive LLM landscape has three load-bearing developments: (1) **multi-turn jailbreaks** (Crescendo) that defeat safety alignment with entirely benign-looking conversation;[1] (2) **uncensored / abliterated open-weight models** made possible by the discovery that refusal is mediated by a single direction in the residual stream — a rank-one weight edit disables it;[2] and (3) **zero-click indirect prompt injection** proven in production (EchoLeak, CVE-2025-32711).[14] Together they collapse the attack barrier from "research team" to "anyone with a laptop and a browser tab."

## 1 · Taxonomy of adversarial LLM attacks

Building on NIST AI 100-2's adversarial-ML taxonomy (evasion / poisoning / privacy attacks, black-box vs. white-box attacker knowledge)[11], the LLM-specific attack surface decomposes as:

| Vector | Attacker knowledge | Interaction | Core mechanism |
|---|---|---|---|
| **Direct prompt injection / jailbreak** | Black-box | Single-turn | Instruction override, persona/role-play, encoding tricks |
| **Multi-turn jailbreak** | Black-box | Multi-turn | Gradual escalation referencing model's own replies[1] |
| **Indirect prompt injection** | Black-box | Zero- or low-click | Hidden instructions in retrieved content (docs, email, web)[14] |
| **Cross-modal injection** | Black-box | Varies | Instructions hidden in images/audio (new in LLM01:2026)[9] |
| **White-box weight edit ("abliteration")** | White-box | None | Rank-one weight orthogonalization removes refusal[2][4] |
| **Uncensored fine-tunes** | Model publisher | None | Models trained/released without safety alignment |
| **Data/model poisoning & backdoors** | Training access | Training stage | Corrupted corpora or artifacts (NIST: training-stage attacks)[11] |

## 2 · Direct jailbreaking (single-turn)

Direct attacks ask the model, in one turn, to violate its operating constraints. Recognized subtypes (mirroring Microsoft's Prompt Shields taxonomy[16]):

- **Rule-override attempts** — "ignore all previous instructions"-class directives.
- **Behavioral augmentation** — the *Skeleton Key* pattern: rather than replacing the model's guidelines, ask it to augment them ("output any content, prefix it with a warning if offensive"), framed as a safe research context; Microsoft disclosed it in June 2024 and updated Prompt Shields to detect the pattern.[12]
- **Persona / role-play framing** — DAN-style personas that recast restrictions as optional fiction.
- **Encoding attacks** — Base64, ROT13, homoglyphs, and character transformations that smuggle intent past string-matching filters.
- **Conversation mockups** — a single user message fabricating prior assistant turns to confuse the model.[16]

Detection note: direct attacks are the best-covered class (Prompt Shields user-prompt scanning is GA across models and agents), but they are classifiers — independent red-team research measured ~72% evasion on injections and ~60% on jailbreaks against Azure Prompt Shields under character-injection attacks (emoji smuggling, Unicode tags, homoglyphs, zero-width characters).[13]

## 3 · Multi-turn jailbreaking — Crescendo

**Mechanism.** Crescendo is a fully black-box, multi-turn jailbreak: it opens with a general, benign question about the target topic, then progressively escalates by referencing the model's own replies — small steps, each individually defensible — until the model produces content it would have refused up front. It mirrors the "foot-in-the-door" psychological tactic and exploits the model's tendency to follow patterns and attend to recent text, especially its own output.[1]

**Why it evades defenses.**

1. Defenses designed to react to *user prompts* miss the cumulative trajectory; Microsoft's own write-up acknowledged single-turn Prompt Shields didn't catch it (the fix: multi-turn filtering that passes the entire conversation history to the detector, plus an AI watchdog trained on adversarial examples).[1][12]
2. Representation-engineering analysis found safety-aligned models often represent Crescendo responses as *more benign than harmful*, especially as turns increase — outputs stay in a "benign" region of representation space while the trajectory ends somewhere harmful. The same work explains why single-turn defenses like circuit breakers generalize poorly to multi-turn attacks.[3]
3. Automation is trivial: Crescendomation requires only black-box API access and outperformed prior state-of-the-art jailbreaks (Many-Shot, PAIR, CIA, CoA) by 29–61% on GPT-4 and 49–71% on Gemini-Pro on the AdvBench subset — and achieved high attack success rates across GPT-4, GPT-3.5, Gemini Pro/Ultra, Llama-2/3 70B, Claude 3 Opus, and Mistral.[1]

**Defensive consequence:** ASR must be measured against *multi-turn* adaptive attacks, not single-turn benchmarks; conversation-history-aware detection and trajectory monitoring are required controls (see `docs/04_Defensive_Guardrails.md` §3).

## 4 · Indirect prompt injection & the zero-click era — EchoLeak

Indirect (cross-prompt) injection hides instructions in content the *system* retrieves — documents, emails, web pages — rather than content the attacker types. It is the vector that turns AI copilots into attack surfaces, and Microsoft calls it "an inherent risk" of probabilistic language modeling with deterministic detection still an open research problem.[12]

**The EchoLeak chain (CVE-2025-32711, CVSS 9.3)** — first publicly documented zero-click prompt injection achieving practical data exfiltration from a production LLM system (Microsoft 365 Copilot):[14][15]

1. **XPIA bypass.** The attacker's email is worded for a *human* reader (a compliance-flavored note), never mentioning "AI" or "ignore previous instructions" — so the AI-directed XPIA classifier doesn't fire.[14]
2. **Retrieval.** The victim later asks Copilot an unrelated question; the retrieval engine pulls the attacker's email into context. Hidden instructions perform an "LLM Scope Violation" — referencing the most sensitive content from elsewhere in the session.[14][15]
3. **Link-filter bypass.** Copilot redacted inline Markdown links `[text](url)` but not the reference-style variant `[text][ref]` — the filter gap became the exfil channel.[14]
4. **CSP bypass.** Copilot's content security policy allowed `*.teams.microsoft.com`; a Teams async-preview endpoint acted as an open redirect, proxying the attacker's URL past the allow-list.[14]
5. **Zero-click completion.** The chat UI auto-fetches the Markdown image; the URL carries the stolen data. No user interaction at any step. Aim Labs worked with MSRC; the fix is rolled out.[15][14]

**Generalizable lessons** (from the academic case study): prompt partitioning, enhanced input/output filtering, provenance-based access control, strict CSPs, least privilege, and defense-in-depth.[15] Willison's framing is broader: exfil-via-Markdown is a *recurring class* across a dozen products — any agent that renders model output containing untrusted-origin links is in scope.

**The agentic amplification — the "lethal trifecta."** An AI system becomes exfiltration-capable when it combines three properties: access to **private data**, exposure to **untrusted content**, and a channel for **external communication**. Any agent with all three is injectable and leakable; break one leg by design and the injection has nowhere to go.[13]

## 5 · Uncensored & abliterated open-weight models

This is the structural change in the threat landscape: attacks that once required skill now require only a download.

**The research basis.** Peer-reviewed work (Arditi et al., NeurIPS 2024) demonstrated that refusal in conversational LLMs is mediated by a **one-dimensional subspace**: across 13 popular open-source chat models up to 72B parameters, a single direction in the residual stream is both *necessary and sufficient* for refusal. Ablating that direction prevents refusal of harmful instructions; adding it elicits refusal of harmless ones.[2][4] The authors turned this into a **white-box jailbreak** — a rank-one weight orthogonalization that removes refusal while retaining general capabilities, requiring no gradient optimization and no examples of harmful completions.

**Why it matters for defenders:**

1. **Barrier collapse.** Any published open-weight model with a known refusal direction can be *abliterated* into an uncensored variant in minutes, on commodity hardware. Uncensored variants of Llama/Mistral-class models circulate openly on model hubs.
2. **No telemetry, no guardrails.** Uncensored self-hosted models ship with no content policy, no abuse monitoring, and no vendor accountability — the "internal tools built with unsecured open-source models" pattern appears in IBM's Shadow-AI findings.[8]
3. **Downstream power.** An uncensored model is a drop-in engine for spam/phishing generation, malware-assist, and jailbreak-content laundering; paired with agentic tooling it inherits whatever the integrator's permission model allows (see Excessive Agency[6]).
4. **Detection asymmetry.** Network-level and content-level classifiers trained on vendor-model refusals see none of this traffic — the model simply complies.

**Defensive posture:** govern *which weights* run in the enterprise (model provenance and approval), restrict local-inference egress, and treat uncensored-model use as a policy violation with technical detection (HuggingFace/llama.cpp process and egress monitoring). The white-box nature of the research also has a constructive reading: the same refusal-direction analysis can harden models — activation addition elicits refusal where it was weak[2][4] — which is the kind of work a mature AI-security team should track.

## 6 · Why defenses fail — the honest model

1. **Classifiers are probabilistic.** Prompt Shields, content filters, and watchdogs raise attacker cost but are demonstrably evadable (EchoLeak's production bypass[14][15]; measured evasion under character injection[13]). Design them like ASLR: assume eventual bypass.
2. **Single-turn defenses vs. multi-turn attacks.** Representation-level evidence shows the generalization gap directly.[3]
3. **Concatenation destroys context.** Without structural separation, the model cannot tell instructions from data — the root cause behind LLM01, and the reason *prompt engineering alone* is not a control.[5]
4. **Blast radius is set by permissions, not detection.** The injection is only as bad as what the agent's identity can reach — which is why least privilege and HITL, not better classifiers, are the controls that "actually hold."[12][13]

The deterministic floor — information-flow control, least-privilege tooling, tool-call gates, human-in-the-loop, exfil-pattern blocking — is the subject of `docs/04_Defensive_Guardrails.md`.

## Sources

[1] https://arxiv.org/abs/2404.01833 — The Crescendo Multi-Turn LLM Jailbreak Attack
[2] https://arxiv.org/abs/2406.11717 — Refusal in Language Models Is Mediated by a Single Direction
[3] https://arxiv.org/abs/2507.02956 — A Representation Engineering Perspective on the Effectiveness of Multi-Turn Jailbreaks
[4] https://proceedings.neurips.cc/paper_files/paper/2024/file/f545448535dfde4f9786555403ab7c49-Paper-Conference.pdf — Refusal Direction (NeurIPS 2024)
[5] https://aka.ms/spotlighting — Microsoft: Introducing Spotlighting in Azure AI Foundry
[6] https://owasp.org/www-project-top-10-for-large-language-model-applications/2_0_vulns/LLM06_ExcessiveAgency — OWASP LLM06:2025 Excessive Agency
[8] https://newsroom.ibm.com/2025-07-30-ibm-report-13-of-organizations-reported-breaches-of-ai-models-or-applications,-97-of-which-reported-lacking-proper-ai-access-controls — IBM Cost of a Data Breach Report 2025 (press release)
[9] https://genai.owasp.org/download/56857 — OWASP Top 10 for LLM Applications 2026 (PDF)
[11] https://csrc.nist.gov/pubs/ai/100/2/e2023/final — NIST AI 100-2 E2023: Adversarial Machine Learning Taxonomy
[12] https://arnav.au/2026/06/04/microsofts-ai-prompt-defense-stack-what-actually-works — Microsoft's AI Prompt Defense Stack: What Actually Works
[13] https://nitinksingh.com/posts/prompt-shields-spotlighting-defense-in-depth — Defense-in-Depth for AI Agents on Azure
[14] https://simonwillison.net/2025/Jun/11/echoleak — Breaking down EchoLeak - the First Zero-Click AI Vulnerability Enabling Data Exfiltration from Microsoft 365 Copilot (Simon Willison)
[15] https://arxiv.org/abs/2509.10540 — EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System
[16] https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection — Prompt Shields in Azure AI Content Safety - Microsoft Learn
