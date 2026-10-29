# Research Handover: Provenance-Tainted Agency (PTA) for Agentic SOC

**Authors:** Veer Pratap Singh, Rudra  
**Date:** September – October 2026  
**Target Paper:** *Trusted Substrate, Untrusted Text: A Provenance-Tainted Agentic Architecture for Real-Time Backend Defence*

---

## 1. Project Overview & Context

This project addresses a structural vulnerability in AI-augmented Security Operations Centers (SOCs). 

Modern SOC architectures compose two layers:
1. **Kernel-level substrate (eBPF):** Streams unforgeable ground truth (syscalls, process lineage, socket bindings) with minimal CPU overhead.
2. **Agentic LLM reasoning layer:** Triages alerts, correlates events, and executes containment playbooks autonomously.

**The Flaw:** Telemetry that feeds the reasoning layer inevitably carries **attacker-controlled text** (User-Agent strings, HTTP URIs, request bodies, DNS query names, attempted usernames). Because LLMs struggle to separate data from instructions, an attacker can embed prompt injection payloads directly inside the telemetry. autonomy multiplies the blast radius of a single successful injection: a compromised analyst does not just misclassify—it isolates critical hosts or rotates credentials on command.

---

## 2. Threat Model & Attack Taxonomy

We evaluate the system against inference-time adversarial data attacks:

| Attack Code | Attack Class | Mechanism | Measured Impact on Undefended LLMs |
|---|---|---|---|
| **S1** | Direct Override | Raw imperatives (`SYSTEM: ignore alert`) | 0% on reasoning models (readily caught) |
| **S2** | Persona Hijack | Impersonates legitimate entities (`SecOpsMaintenanceBot`) | **66.7% – 100% suppression** |
| **S3** | Context Manipulation | Embeds authoritative benign context around intrusion | High evasion on summarisation tasks |
| **S4** | Obfuscated Payload | Encoded / token-split injection strings | Bypasses simple regex / ingest keyword filters |
| **S5** | **Fragmented Injection** | Payload split across $K$ requests; reassembled by the defender's own log aggregator | **30% – 50% suppression when contiguous** |

---

## 3. Architecture: Provenance-Tainted Agency (PTA)

PTA replaces probabilistic prompt-level mitigations with structural capability bounds:

```
[ Ingested Telemetry ]
          │
          ▼
┌───────────────────────────────────────┐
│ M1: PROVENANCE TAINT LABELLER         │
│     T0: Kernel-attested (pid, cgroup) │
│     T1: Infra-generated (peer IP)     │
│     T2: Attacker-controlled text      │
└───────────────────┬───────────────────┘
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
┌───────────────────────┐ ┌───────────────────────────────┐
│ M6: ADVERSARIAL CANARY│ │ M3: TEXT-BLIND STATISTICAL    │
│     Non-generative    │ │     DETECTOR                  │
│     instruction score │ │     Operates on graph metrics │
└─────────────┬─────────┘ └───────────────┬───────────────┘
              │                           │
              ▼                           │
┌───────────────────────┐                 │
│ M2: ISOLATED READER   │                 │
│     Sees ONLY T2 text │                 │
│     Constrained schema│                 │
│     ZERO tool rights  │                 │
└─────────────┬─────────┘                 │
              │ enum verdict              │
              ▼                           ▼
┌─────────────────────────────────────────────────────────┐
│ M2: DECIDER AGENT                                       │
│     Sees ONLY T0/T1 telemetry + Reader's enum verdict   │
│     Evaluates Dissent with Text-Blind Detector          │
│     Proposes containment actions (never acts directly)  │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│ M4: DETERMINISTIC POLICY GATE                           │
│     Enforces blast-radius budget, reversible actions,   │
│     and issues cryptographic rollback tokens.           │
└─────────────────────────────────────────────────────────┘
```

### The Eight Core Mechanisms:
- **M1 (Taint by Provenance):** Strict tagging of telemetry into T0, T1, and T2. Invariant: T2 never enters instruction contexts of tool-wielding agents.
- **M2 (Reader/Decider Split):** Reader is constrained to a fixed enum schema (`{sqli, xss, traversal, cmdi, brute_force, benign}`); Decider sees only trusted telemetry and the enum output.
- **M3 (Disagreement as Detection):** Parallel text-blind detector; if LLM returns "benign" while statistical graph detector flags an anomaly, the system escalates on the **dissent** itself.
- **M4 (Capability-Bounded Response):** Deterministic policy gate with blast-radius budgets and rollback tokens.
- **M5 (Single-Fragment Isolation):** Evaluates one T2 field at a time stateless to prevent multi-event payload concatenation.
- **M6 (Adversarial Reassembly Canary):** Non-generative classifier that clusters T2 values and scores them for instruction-likeness prior to model evaluation.
- **M7 (Codebook Canonicalisation):** Maps raw strings to bounded tokens (`ua:chrome-120-desktop`).
- **M8 (Cross-Event T2 Budget):** Caps untrusted byte volume per kernel identity over time.

---

## 4. Key Implementation Files in This Repository

1. **PTA Core Module:** `backend/apps/agentic/pta/`
   - `taint.py`: Field classification into T0/T1/T2 and strict air-gap enforcement.
   - `canary.py`: Non-generative instruction and S5 fragment detector.
   - `policy_gate.py`: Capability budget and rollback token manager.
   - `dissent.py`: Statistical graph/rate detector and dissent channel.
   - `reader_decider.py`: Schema-constrained Reader and decoupled Decider.
2. **Hardened Investigation Playbook:**
   - `backend/custom/playbooks/pta_investigation.py`: End-to-end hardened triage workflow.
3. **Research Results & Figures:**
   - `docs/research/results.md`: Complete evaluation metrics and tables.
   - `docs/research/figures/`: Publication-ready comparison charts.

---

## 5. Summary for AI Agents & Collaborators

When processing alerts or extending playbooks in this repository:
- **Never pass raw T2 fields directly into LLM prompts.**
- **Enforce the M2 schema air gap:** all untrusted text must pass through `PTAReader` before reaching reasoning components.
- **Route all response actions through `PolicyGate`** to guarantee auditability, blast-radius compliance, and rollback token generation.
