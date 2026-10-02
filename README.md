# AI Cybersecurity — The Double-Edged Sword

**A technical analysis of the dual-use nature of AI in cybersecurity: offensive LLM attack vectors (Red Team), internal corporate risks (Shadow AI), and the defensive architecture that contains both (Blue Team).**

> **Audience:** technical recruiters, CISOs, SOC managers, and senior security engineers.
> **Status:** `v1.0.0` · October 2026 · Maintained by [@chesuka21](https://github.com/chesuka21)

---

## Why this repository exists

Enterprise AI adoption is running ahead of AI security. IBM's 2025 *Cost of a Data Breach* report (Ponemon Institute, 600 organizations) found that 13% of organizations have already suffered breaches of AI models or applications — and 97% of those reported lacking proper AI access controls.[8] One in five organizations reported a breach caused by **Shadow AI** (unregulated, unauthorized use of AI), with an average cost premium of **USD 670,000** for organizations with high levels of Shadow AI.[8]

At the same time, the offensive cost curve is collapsing:

- **Multi-turn jailbreaks** such as *Crescendo* escalate from benign questions to policy-violating output in a handful of turns; its automated variant (Crescendomation) outperformed prior state-of-the-art jailbreaks by 29–61% on GPT-4 and 49–71% on Gemini-Pro.[1]
- **Uncensored open-weight models** make safety bypass nearly free: peer-reviewed research shows refusal behavior in chat models is mediated by a *single direction* in the residual stream — erasing that one direction (or a rank-one weight edit) disables refusal across 13 open-source models up to 72B parameters.[2][4]
- **Zero-click prompt injection** has moved from theory to production incidents: *EchoLeak* (CVE-2025-32711, CVSS 9.3) achieved remote data exfiltration from Microsoft 365 Copilot with no user interaction, chaining an XPIA-classifier bypass, a Markdown link-filter bypass, and a CSP allow-list abuse.[14][15]

This repository treats these as **one problem with two faces**: the same under-governed technology that leaks corporate data through an employee's browser tab also lowers the barrier for an attacker's laptop. Effective defense requires understanding both sides — which is why every offensive vector documented here maps to a concrete mitigation in `docs/04_Defensive_Guardrails.md` and a control in `templates/llm_hardening_checklist.md`.

## The three pillars

| Pillar | Scope | Where |
|---|---|---|
| **1 · Offensive vectors (Red Team)** | Prompt injection, jailbreaks, multi-turn attacks, uncensored/abliterated open-weight models, automated attack tooling | `docs/03_Adversarial_Vectors.md` |
| **2 · Internal corporate risks (Shadow AI & governance)** | Unregulated third-party LLM use, PII/source-code/IP exposure, excessive agency in agentic deployments | `docs/01_Executive_Summary.md`, `docs/02_Threat_Modeling_AI.md` |
| **3 · Defensive guardrails (Blue Team / SOC)** | Defense-in-depth: Prompt Shields, Spotlighting, engine isolation, DLP output filtering, human-in-the-loop | `docs/04_Defensive_Guardrails.md`, `templates/llm_hardening_checklist.md` |

## Headline numbers (IBM Cost of a Data Breach 2025)[8]

| Metric | Value |
|---|---|
| Organizations breached via AI models/applications | 13% (97% lacked AI access controls) |
| Breaches involving Shadow AI | 1 in 5 organizations (20%) |
| Added breach cost for high Shadow AI | +USD 670,000 |
| PII compromised in Shadow-AI incidents vs. global average | 65% vs. 53% |
| Intellectual property compromised vs. global average | 40% vs. 33% |
| Organizations with policies to manage/detect Shadow AI | 37% |
| Savings from extensive AI/automation in security operations | USD 1.9M and 80 days faster |
| Global average breach cost 2025 | USD 4.44M (first decline in five years) |
| US average breach cost 2025 (record) | USD 10.22M |

## OWASP Top 10 for LLMs — 2025 vs. 2026 edition

The 2026 edition (published August 4, 2026) reordered the list significantly: Excessive Agency climbed to #3 ("agentic deployments are where the damage is landing"), Unbounded Consumption rose four places, and System Prompt Leakage was re-scoped and renamed **Hidden Context Exposure**.[9][10] Knowing both editions — and *why* entries moved — is baseline literacy for anyone securing LLM systems.

| # | 2025 | 2026 | Movement |
|---|---|---|---|
| LLM01 | Prompt Injection | Prompt Injection | steady (now covers cross-modal attacks) |
| LLM02 | Sensitive Information Disclosure | Sensitive Information Disclosure | steady |
| LLM03 | Supply Chain | Excessive Agency | ▲ escalated (was #6) |
| LLM04 | Data and Model Poisoning | Supply Chain | ▲ (was #3) |
| LLM05 | Improper Output Handling | Data and Model Poisoning | — (was #4; absorbs fine-tuning subversion) |
| LLM06 | Excessive Agency | Unbounded Consumption | ▲ (was #10) |
| LLM07 | System Prompt Leakage | Misinformation | — (was #9) |
| LLM08 | Vector and Embedding Weaknesses | Hidden Context Exposure | renamed/re-scoped (was "System Prompt Leakage", #7) |
| LLM09 | Misinformation | Vector and Embedding Weaknesses | — (was #8) |
| LLM10 | Unbounded Consumption | Improper Output Handling | ▼ deprioritized (was #5) |

Note the boundary the OWASP GenAI project itself draws: the LLM Top 10 owns risk when the model is a **component** of your application; the moment the model becomes an **actor** (tools, memory, downstream consequences), risk moves to the OWASP Top 10 for Agentic Applications (ASI).[9]

## Repository structure

```text
AI-Cybersecurity-DoubleEdgedSword/
├── README.md                                 ← you are here
├── SOUL.md                                   ← project identity & governance
├── docs/
│   ├── 01_Executive_Summary.md               ← business impact & breach economics
│   ├── 02_Threat_Modeling_AI.md              ← STRIDE + OWASP applied to LLM systems
│   ├── 03_Adversarial_Vectors.md             ← injection, jailbreaks, uncensored models
│   └── 04_Defensive_Guardrails.md            ← defense-in-depth architecture
├── benchmarks/
│   ├── prompt_injection_cases.json           ← anonymized, structured attack test cases
│   └── risk_evaluation_matrix.csv            ← offensive impact vs. defensive posture
└── templates/
    └── llm_hardening_checklist.md            ← enterprise hardening checklist
```

## Suggested reading paths

- **Technical recruiter (5–10 min):** this README → `docs/01_Executive_Summary.md` → `benchmarks/risk_evaluation_matrix.csv`.
- **Security engineer / red teamer:** `docs/02_Threat_Modeling_AI.md` → `docs/03_Adversarial_Vectors.md` → `docs/04_Defensive_Guardrails.md` → `benchmarks/prompt_injection_cases.json`.
- **CISO / SOC manager:** `docs/01_Executive_Summary.md` → `docs/02_Threat_Modeling_AI.md` → `templates/llm_hardening_checklist.md`.

## Frameworks referenced

- **OWASP Top 10 for LLM Applications** — 2025 and 2026 editions[7][9][10]
- **NIST AI 100-2 E2023** — Adversarial Machine Learning: taxonomy of attacks and mitigations[11]
- **Microsoft Zero Trust guidance for AI workloads**; Azure AI **Prompt Shields** and **Spotlighting**[5][16]
- **MITRE ATT&CK tactics** as applied to AI systems (via the Defender for Cloud AI threat alert taxonomy)[12]

## Methodology & ethics

- All offensive material is documented at the **conceptual and architectural level**. This repository contains **no operational exploit code** and no target-specific payloads.
- Attack cases in `benchmarks/prompt_injection_cases.json` are **synthetic and anonymized**, designed as defensive regression vectors (what a guarded system *should* do), not as an attack cookbook.
- Referenced vulnerabilities (e.g., CVE-2025-32711) are publicly disclosed and patched; analysis cites the vendors' own publications and peer-reviewed research.[14][15]
- Intent is defensive-first: Red Team knowledge is included because Blue Teams cannot engineer mitigations for threats they do not understand.

## Sources

[1] https://arxiv.org/abs/2404.01833 — The Crescendo Multi-Turn LLM Jailbreak Attack
[2] https://arxiv.org/abs/2406.11717 — Refusal in Language Models Is Mediated by a Single Direction
[4] https://proceedings.neurips.cc/paper_files/paper/2024/file/f545448535dfde4f9786555403ab7c49-Paper-Conference.pdf — Refusal Direction (NeurIPS 2024)
[5] https://aka.ms/spotlighting — Microsoft: Introducing Spotlighting in Azure AI Foundry
[7] https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025 — OWASP Top 10 for LLM Applications 2025
[8] https://newsroom.ibm.com/2025-07-30-ibm-report-13-of-organizations-reported-breaches-of-ai-models-or-applications,-97-of-which-reported-lacking-proper-ai-access-controls — IBM Cost of a Data Breach Report 2025 (press release)
[9] https://genai.owasp.org/download/56857 — OWASP Top 10 for LLM Applications 2026 (PDF)
[10] https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/blob/main/2026/README.md — OWASP LLM Top 10 2026 - canonical source (GitHub)
[11] https://csrc.nist.gov/pubs/ai/100/2/e2023/final — NIST AI 100-2 E2023: Adversarial Machine Learning Taxonomy
[12] https://arnav.au/2026/06/04/microsofts-ai-prompt-defense-stack-what-actually-works — Microsoft's AI Prompt Defense Stack: What Actually Works
[14] https://simonwillison.net/2025/Jun/11/echoleak — Breaking down EchoLeak - the First Zero-Click AI Vulnerability Enabling Data Exfiltration from Microsoft 365 Copilot (Simon Willison)
[15] https://arxiv.org/abs/2509.10540 — EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System
[16] https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection — Prompt Shields in Azure AI Content Safety - Microsoft Learn
