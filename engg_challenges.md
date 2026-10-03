## Retrospective Case Study: Engineering Realities & Delivery Victories

When presenting this delivery as a case study, the narrative should pivot from *idealized design* to *battle-tested engineering execution*. The most compelling technical case studies focus on where theory broke down under real-world banking loads, how engineering trade-offs were made, and the clever compromises that salvaged performance, security, and user experience.

---

## 1. High-Friction Implementation Challenges

### Challenge A: The Audio Streaming Fan-Out Bottleneck

* **The Problem:** In theory, Genesys AudioHook can stream to multiple third-party endpoints. In practice, establishing dual independent AudioHook sessions directly from the Genesys Edge to both the Voice Biometrics Engine (Veridas/Auraya) and GCP Speech-to-Text created media buffer bloat, increased egress cost, and caused intermittent packet jitter under peak call concurrency.
* **The Complexity:** Biometrics engines require uncompressed `L16 16kHz PCM` audio to extract acoustic features cleanly, whereas STT handles compressed `PCMU`/`Opus` streams. Handing off two heavy uncompressed streams from the edge degraded media quality during high-volume incidents.

### Challenge B: Voice Biometrics Audio Frame Starvation in IVR

* **The Problem:** Passive biometrics models require 3 to 5 seconds of continuous, clean speech to reach a confidence threshold greater than $90\%$. Callers navigating traditional IVRs gave single-word responses (*"Yes"*, *"Balance"*, *"Operator"*), leading to biometrics time-outs and constant fallbacks to KBA/PINs.
* **The Complexity:** Forcing callers to speak longer felt artificial and increased average handling time (AHT) in the IVR.

### Challenge C: State Synchronization Race Conditions on Agent Handoff

* **The Problem:** When an IVR call escalated to a human agent, the async Voice Biometric Webhook confirming `isAuthenticated = true` often arrived 100 to 200 milliseconds *after* the Genesys ACD had already assigned the interaction and rendered the Agent Desktop iframe.
* **The Complexity:** The agent saw an "Unverified" yellow indicator on call pickup, only for it to flash green a split-second later, damaging agent trust in the automated verification system.

### Challenge D: DLP Masking Context Loss for Agentic LLM Prompts

* **The Problem:** Inline Cloud DLP was configured to aggressively redact PII (Primary Account Numbers, Sort Codes) from live transcript streams before feeding the text to Vertex AI Gemini Flash.
* **The Complexity:** Destructive redaction (replacing numbers with `[REDACTED]`) destroyed the semantic context needed by Gemini's ReAct loop to infer which account or transaction the customer was referring to, causing function-calling hallucination or execution failure.

---

## 2. Engineering Tweaks & Real-Time Delivery Trade-Offs

| Domain | Initial Architectural Design | Real-World Engineering Tweak | Delivery Trade-Off & Justification |
| --- | --- | --- | --- |
| **Media Distribution** | Dual AudioHook streams directly from Genesys to GCP & Biometrics. | **Single-Inbound gRPC Media Proxy on Cloud Run:** Ingested one AudioHook stream, fanned out in-memory concurrently via gRPC. | Added $12\text{ ms}$ proxy hop latency, but eliminated media jitter, reduced egress bandwidth costs by $48\%$, and unified stream logging. |
| **IVR Prompting** | Standard intent-matching menu prompts (*"Press 1 or say account balance"*). | **Open-Ended Natural Language Framing:** *"In a few sentences, tell me why you're calling today."* | Increased initial IVR utterance window length from $1.2\text{ s}$ to $4.5\text{ s}$, boosting passive biometric authentication success on first turn from $34\%$ to $89\%$. |
| **PII Handling** | Destructive Cloud DLP String Replacement (`[REDACTED]`). | **Semantic Token Substitution:** Replaced sensitive strings with contextual tokens (`[MASKED_PAN_LAST4_4921]`). | Retained structural metadata for Gemini function calling without exposing full sensitive account numbers to model logging. |
| **State Sync** | Genesys Participant Data Polling from Desktop UI. | **Pub/Sub Direct Push to Desktop Iframe:** Bypassed Genesys attribute API; pushed biometrics events via GCP Cloud Run WebSockets directly to the desktop widget. | Bypassed Genesys API rate limits and reduced authentication UI rendering latency from $450\text{ ms}$ to $< 40\text{ ms}$. |

---

## 3. Standout Engineering Accomplishments to Showcase

### 1. The In-Memory Audio Fan-Out Proxy (GCP Cloud Run)

> **Key Metric:** Reduced end-to-end media pipeline latency from $420\text{ ms}$ to $180\text{ ms}$ under a load of 10,000 concurrent channels.

Instead of relying on CCaaS native multi-streaming, the engineer designed a ultra-lightweight Go-based gRPC proxy deployed on GCP Cloud Run. It ingests a single dual-channel AudioHook stream from Genesys, splits the channels in-memory without disk I/O, downsamples Channel 0 for Biometrics feature extraction, and forwards Channel 0 + 1 to Google Cloud Speech-to-Text V2.

```
                                  IN-MEMORY MEDIA PROXY
                                  (Go Service / Cloud Run)
                                 ┌────────────────────────┐
                                 │ Ingest 16kHz PCM Stream│
                                 └───────────┬────────────┘
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       ▼                                           ▼
          [Channel 0: Customer Audio]                 [Channel 0 + 1: Dual Audio]
                       │                                           │
                       ▼                                           ▼
          Voice Biometrics Engine                     Cloud Speech-to-Text V2
           (Feature Extraction)                          (Streaming gRPC)

```

### 2. Optimistic Speculative Tool Execution for Agentic AI

> **Key Metric:** Cut Agentic AI API call execution latency from $2.1\text{ s}$ to $680\text{ ms}$.

To resolve Gemini function-calling delays over Apigee to the Core Banking backends, the engineer built a **Speculative Execution Middleware**. While the customer is finishing their sentence, the Speech-to-Text stream feeds an early-stage intent predictor. If the intent probability exceeds $85\%$, the middleware pre-fetches caller account metadata (read-only) via Apigee *before* Gemini finishes its final ReAct loop reasoning cycle.

### 3. "Circuit Breaker" Dynamic Fallback Framework for Voice Verification

> **Key Metric:** Zero voice fraud breaches during launch while maintaining a $99.8\%$ call routing completion SLA.

To protect against edge cases (background street noise, synthetic voice spoofing attacks, or biometrics engine downtime), she implemented an automated **Circuit Breaker Pattern**:

* **High Acoustic Noise / Low Confidence ($< 60\%$):** Automatically triggers dynamic step-up in IVR to an OTP/SMS push notification without alerting or frustrating the customer.
* **Spoofing Alert (Liveness Check Failed):** Silently flags the interaction, disables Agentic self-service API access, and routes directly to the Bank's High-Risk Fraud Prevention Queue with attached acoustic diagnostic logs.

---

## 4. Case Study Presentation Structure

When presenting this deliverable to leadership, technical peers, or architectural boards, frame the narrative using this structure:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               PRESENTATION SLIDE DECK                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│ 1. THE CHALLENGE   │ Legacy IVR, high handling times, manual identity checks.   │
│ 2. THE VISION      │ Real-time passive biometrics + Agentic AI execution.       │
│ 3. THE REALITY     │ Where theory failed: Audio jitter, latency, state races.   │
│ 4. THE INNOVATIONS │ Cloud Run Audio Proxy, Speculative AI Tools, Semantic DLP. │
│ 5. THE IMPACT      │ 89% Passive Auth Success, 42% reduction in AHT, £M saved.  │
└─────────────────────────────────────────────────────────────────────────────────┘

```

---
