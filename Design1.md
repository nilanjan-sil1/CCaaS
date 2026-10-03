# High Street Bank Contact Centre Modernization

## Engineering Architecture Specification: Agentic AI & Passive Voice Biometrics Platform

---

## 1. Executive Summary & Architectural Vision

This specification defines the production-grade engineering architecture for modernizing a Tier-1 High Street Bank's Contact Centre. The solution integrates **Genesys Cloud CX** (as the CCaaS orchestration and telephony engine) with **Google Cloud Platform (GCP)** (for Agentic AI, NLU, and Real-Time Co-Pilot) and a **Passive Voice Biometrics Engine** (e.g., Auraya EVA / Veridas) via real-time media streaming.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             HIGH STREET BANK CCaaS ENGINE                        │
│                           Genesys Cloud CX (Telephony/ACD)                       │
└─────────────────────────┬──────────────────────────────┬─────────────────────────┘
                          │                              │
                          │ AudioHook (gRPC / WS)        │ Real-time Event Stream
                          ▼                              ▼
┌─────────────────────────────────────────┐  ┌─────────────────────────────────────┐
│    PASSIVE VOICE BIOMETRICS ENGINE      │  │        GOOGLE CLOUD PLATFORM        │
│   (Auraya EVA / Veridas / Custom Engine)│  │     Agentic AI & Agent Assist       │
└─────────────────────────┬───────────────┘  └──────────────────┬──────────────────┘
                          │                                     │
                          │ Async Webhook / Event               │ Function Calling
                          └───────────────────┬─────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         ENTERPRISE API GATEWAY (Apigee)                          │
│                      Core Banking | CRM | Fraud Systems                          │
└──────────────────────────────────────────────────────────────────────────────────┘

```

### Key Capabilities Introduced

1. **Passive Voice Biometrics**: Silent, continuous identity verification operating during IVR navigation, Agentic AI dialogue, or human agent interactions via dual-channel AudioHook streaming.
2. **Customer-Facing Agentic AI**: Autonomous digital workers executing multi-turn dialog, handling intent recognition, and invoking secure banking API tool calls without human intervention when passive authentication succeeds.
3. **Real-Time Agent Co-Pilot (Agent Assist)**: Live transcription, automated knowledge RAG, passive verification status screen pops, next-best-action (NBA) guidance, and automated interaction summarization.

---

## 2. L2 Container Level Architecture Diagram

The L2 diagram defines the container boundaries, transport protocols, network isolation boundaries, and core system interfaces across Genesys Cloud CX, GCP, Third-Party Voice Biometrics, and the Bank’s On-Premises/Private Cloud Core Network.

```mermaid
graph TB

    subgraph Customer_Channels["Customer Ingress Layer"]
        PSTN_In["Voice Call (PSTN / SIP Trunk)"]
        Digital_In["Digital Channels (Web Chat / Mobile App)"]
    end

    subgraph Genesys_Cloud["Genesys Cloud CX Boundary (CCaaS Orchestration)"]
        Edge_Media["Genesys Cloud Voice Edge / BYOC"]
        ACD_Routing["ACD Engine & Architect Flow Manager"]
        AudioHook_Server["AudioHook Integration Service"]
        Event_Dispatcher["Notification API & Webhook Engine"]
        Agent_Desktop_Host["Genesys Embeddable Desktop Host"]
    end

    subgraph Voice_Biometrics_Engine["Third-Party Voice Biometrics Engine (Auraya / Veridas)"]
        Bio_Ingress["Biometrics Ingress Gateway"]
        Bio_Stream_Proc["Audio Processing & Feature Extraction"]
        Voiceprint_DB["Encrypted Voiceprint Profile Vault"]
        Bio_Scoring["Acoustic Matching & Anti-Spoofing Model"]
    end

    subgraph GCP_Ingress_Security["GCP Edge & Security Layer"]
        Cloud_Armor["Cloud Armor WAF / DDoS Mitigation"]
        Ingress_LB["External / Internal gRPC & HTTPS Load Balancers"]
        Cloud_DLP["Cloud DLP (Streaming PII / PCI Redaction)"]
    end

    subgraph GCP_Agentic_AI["GCP Agentic AI & Intelligence Core"]
        Dialogflow_CX["Dialogflow CX / Conversational AI"]
        Agentic_Orchestrator["Cloud Run Agentic Workflow Engine"]
        Vertex_AI["Vertex AI (Gemini 1.5 Flash / Vector Search)"]
        Agent_Assist["CCAI Agent Assist Runtime"]
        PubSub_Bus["Cloud Pub/Sub Message Bus"]
    end

    subgraph GCP_Data_State["GCP Analytics & Session State"]
        Firestore_State["Firestore (Real-Time Session & Auth State)"]
        BigQuery_DW["BigQuery (Transcripts & Biometric Audit)"]
        GCS_Bucket["Cloud Storage (Audio Logs & Analytics Dump)"]
    end

    subgraph Bank_Core["High Street Bank Core Network (Private VPC / On-Prem)"]
        Apigee_GW["Apigee Enterprise API Gateway (mTLS / OAuth2)"]
        Core_Banking["Core Banking Systems (Accounts, Transfers, Cards)"]
        CRM_Salesforce["Enterprise CRM (Salesforce / Dynamics 365)"]
        Bank_HSM["Cloud HSM / Key Management Service (CMEK)"]
    end

    subgraph Agent_Workspace["Agent Desktop Workspace"]
        Agent_Browser["Agent Desktop (Genesys Workspace)"]
        Biometrics_Widget["Passive Biometrics UI Plugin / Screen Pop"]
        CoPilot_Widget["Agent Co-Pilot Real-Time Widget"]
    end

    %% Customer Connections
    PSTN_In -->|SIP / RTP| Edge_Media
    Digital_In -->|HTTPS / WSS| ACD_Routing

    %% Genesys Internal & External Forking
    Edge_Media --> ACD_Routing
    Edge_Media -->|AudioHook gRPC / WSS L16 PCM Stream| Bio_Ingress
    AudioHook_Server -->|Audio Streaming WSS/gRPC| Ingress_LB

    %% Biometrics Flow
    Bio_Ingress --> Bio_Stream_Proc
    Bio_Stream_Proc --> Bio_Scoring
    Bio_Scoring <--> Voiceprint_DB
    Bio_Scoring -->|Async Verification Event / Webhook| Event_Dispatcher
    Bio_Scoring -->|Async Status Webhook| Firestore_State

    %% GCP Ingestion & AI Processing
    Ingress_LB --> Cloud_Armor
    Cloud_Armor --> Cloud_DLP
    Cloud_DLP --> Agentic_Orchestrator
    Cloud_DLP --> Agent_Assist

    ACD_Routing <--> |GCP Data Actions / gRPC| Dialogflow_CX
    Dialogflow_CX <--> Agentic_Orchestrator
    Agentic_Orchestrator <--> Vertex_AI

    %% Bank Core Integrations
    Agentic_Orchestrator -->|Tool Calling / REST| Apigee_GW
    Apigee_GW --> Core_Banking
    Apigee_GW --> CRM_Salesforce
    Apigee_GW --> Bank_HSM

    %% Real-time Desktop Sync
    Event_Dispatcher -->|WebSocket Event Push| Agent_Browser
    Agent_Assist -->|Real-Time Suggestions| CoPilot_Widget
    Firestore_State -->|Auth Status Event| Biometrics_Widget

    %% Analytics Pipelines
    Agentic_Orchestrator --> PubSub_Bus
    PubSub_Bus --> BigQuery_DW
    PubSub_Bus --> GCS_Bucket

    %% Desktop Hierarchy
    Agent_Desktop_Host --> Agent_Browser
    Agent_Browser --> Biometrics_Widget
    Agent_Browser --> CoPilot_Widget

```

---

## 3. L3 Component Level Architecture Diagrams

### 3.1 Passive Voice Authentication & Media Forking Pipeline

This component illustrates how raw dual-channel audio is split by Genesys Cloud CX and streamed simultaneously to the Voice Biometrics Engine and GCP Speech-to-Text pipelines, triggering asynchronous state updates without disrupting user conversation.

```mermaid
graph LR

    subgraph Genesys_Media_Edge["Genesys Media Edge"]
        RTP_Tap["AudioHook Media Tap"]
    end

    subgraph Biometrics_Engine["Voice Biometrics Runtime (Veridas / Auraya)"]
        WSS_Receiver["WebSocket / gRPC Audio Receiver"]
        VAD_Filter["Voice Activity Detector & Noise Filter"]
        Anti_Spoof["Liveness & Synthetic Voice Detector"]
        Feature_Extractor["Acoustic Feature Extractor"]
        Matcher["1:1 Profile Matcher (Score > Threshold)"]
    end

    subgraph Orchestration_Bus["Event & State Management"]
        Auth_Webhook["Biometrics Webhook Handler (Cloud Run)"]
        Genesys_Participant_API["Genesys Participant Data API"]
        Firestore_Session["Firestore Active Session State"]
    end

    RTP_Tap -->|Raw PCM Audio Stream| WSS_Receiver
    WSS_Receiver --> VAD_Filter
    VAD_Filter --> Anti_Spoof
    Anti_Spoof -->|Liveness Confirmed| Feature_Extractor
    Feature_Extractor --> Matcher
    Matcher -->|Confidence Score e.g. 0.96| Auth_Webhook

    Auth_Webhook -->|Set attribute: isAuthenticated=true| Genesys_Participant_API
    Auth_Webhook -->|Update Auth State| Firestore_Session

```

### 3.2 Agentic AI Voicebot & Autonomous Execution Engine

This pipeline handles autonomous voice conversations. The Agentic AI leverages dynamic tool calling to query banking backends, guarded by authentication flags continuously emitted by the Voice Biometrics Engine.

```mermaid
graph TD

    subgraph Caller_Voice["Caller Voice Input"]
        Audio_In["Inbound Audio Stream"]
    end

    subgraph Genesys_Architect["Genesys Architect / Voicebot Bridge"]
        Bot_Node["Dialogflow CX Audio Bot Connector"]
    end

    subgraph GCP_Agentic_Core["GCP Agentic Intelligence Layer"]
        NLU_Engine["Dialogflow CX NLU & Intent Parser"]
        Agent_Loop["Agentic ReAct Loop Engine (Cloud Run)"]
        Gemini_LLM["Vertex AI Gemini 1.5 Flash Engine"]
        Tool_Registry["OpenAPI / Function Calling Registry"]
    end

    subgraph Guardrails_and_Auth["Security Guardrail Layer"]
        Auth_Check["Session State Evaluator (Firestore)"]
        Policy_Validator["Banking Risk & Policy Engine"]
    end

    subgraph Enterprise_API["Enterprise Middleware"]
        Apigee_Gateway["Apigee Enterprise Gateway"]
        Core_APIs["Core Banking APIs (Refund, Address Update)"]
    end

    Audio_In --> Bot_Node
    Bot_Node --> NLU_Engine
    NLU_Engine --> Agent_Loop

    Agent_Loop --> Gemini_LLM
    Gemini_LLM --> Agent_Loop

    Agent_Loop -->|Request Tool Execution| Tool_Registry
    Tool_Registry --> Auth_Check

    Auth_Check -->|Check isAuthenticated State| Policy_Validator

    Policy_Validator -->|Auth Passed & Policy Valid| Apigee_Gateway
    Policy_Validator -->|Auth Failed / Missing| Bot_Node

    Apigee_Gateway --> Core_APIs

    Core_APIs -->|Execution Status Success| Agent_Loop

    Agent_Loop -->|Formulate Natural Response| Bot_Node

```

### 3.3 Real-Time Co-Pilot & Passive Verification Desktop Screen Pop

This pipeline drives the agent desktop experience, delivering live transcription, passive voice biometric verification indicators, and AI-generated answer recommendations during live agent interactions.

```mermaid
graph LR

    subgraph Media_Stream["Genesys Dual Audio"]
        Dual_Audio["AudioHook Dual Channel Stream"]
    end

    subgraph GCP_CoPilot["GCP CCAI Co-Pilot Subsystem"]
        Streaming_STT["Cloud Speech-to-Text V2 (Telephony Model)"]
        DLP_Filter["Cloud DLP (PCI / PII Redaction)"]
        Knowledge_RAG["Vertex AI Vector Search (Bank KB)"]
        CoPilot_LLM["Vertex AI Gemini (Contextual Assist)"]
    end

    subgraph Desktop_UI["Agent Genesys Desktop"]
        Bio_Pop["Passive Biometrics Indicator (Green Banner)"]
        RAG_Card["Suggested Knowledge / Action Card"]
        Manual_Override["Manual Active Auth / KBA Button"]
    end

    Firestore_Session["Firestore Session State"]
    Genesys_Architect["Genesys Architect IVR"]

    Dual_Audio --> Streaming_STT
    Streaming_STT --> DLP_Filter
    DLP_Filter --> Knowledge_RAG
    Knowledge_RAG --> CoPilot_LLM

    CoPilot_LLM -->|WebSocket Push| RAG_Card

    %% Passive Auth Event Triggering Screen Pop
    Firestore_Session -->|Push Verification Status| Bio_Pop

    Manual_Override -->|Trigger DTMF PIN / KBA Flow| Genesys_Architect

```

---

## 4. Operational Workflows & Architectural Execution

### Workflow 1: Passive Authentication in IVR & Agentic AI Self-Service

| Step | Component | Action / Protocol | Data Payload / State Change |
| --- | --- | --- | --- |
| **1.1** | Genesys Edge | Inbound call established. Initiates AudioHook session. | `AudioHook WebSocket Open` (gRPC/WSS, L16 PCM 8kHz/16kHz). |
| **1.2** | AudioHook | Duplicates customer audio stream to Voice Biometrics Engine and GCP. | Dual-channel media fork. |
| **1.3** | Agentic Voicebot | Asks natural open-ended prompt: *"Welcome to High Street Bank. How can I help you today?"* | Audio playback via Genesys Architect. |
| **1.4** | Biometrics Engine | Analyzes acoustic features across 3–5 seconds of natural Speech. Evaluates liveness and compares against stored voiceprint profile. | Computes voiceprint similarity score (e.g., $Score = 97\%$). |
| **1.5** | Biometrics Engine | Invokes GCP Auth Webhook asynchronously upon passing threshold ($Threshold \ge 90\%$). | `POST /v1/auth-event` -> `{"interaction_id": "12345", "authenticated": true, "confidence": 0.97}` |
| **1.6** | Cloud Run / Firestore | Updates interaction session state in Firestore and sets Genesys Participant Data attribute. | `Set Participant Data: context.isAuthenticated = true` |
| **1.7** | Agentic AI Bot | Checks `isAuthenticated` state attribute before executing protected intents (e.g., funds transfer, address change). | Allows execution path without requesting PIN or security questions. |

### Workflow 2: Autonomous Backend Execution by Agentic AI Worker

```
Caller Speech ---> Speech-to-Text ---> Dialogflow CX / Gemini Core
                                                │
                                                ▼
                                    Function Call Requested
                                    (e.g., process_refund)
                                                │
                                                ▼
                                   Evaluate Session Auth State
                                   (Firestore / Participant Data)
                                                │
                                ┌───────────────┴───────────────┐
                                │                               │
                     [isAuthenticated == true]        [isAuthenticated == false]
                                │                               │
                                ▼                               ▼
                     Invoke Apigee Gateway              Trigger Step-Up Auth
                     Execute Core Banking API            (Active PIN or Agent Handoff)

```

### Workflow 3: Live Agent Escalation & Desktop Screen Pop

```
[Agentic AI Handoff / Queue Routing]
               │
               ▼
   Genesys ACD Routes Call to Human Agent Workspace
               │
               ├──────────────────────────────────────────────┐
               ▼                                              ▼
  Genesys Embeddable Desktop                             Genesys AudioHook
  Opens Agent Workspace                                  Streams Dual Audio to GCP CoPilot
               │                                              │
               ▼                                              ▼
  Fetch Session Auth Status from Firestore               Streaming STT + Vertex RAG
               │                                              │
      ┌────────┴────────┐                                     ▼
      ▼                 ▼                         Generate Next-Best Action Cards
[Auth Verified]  [Auth Unverified]                            │
      │                 │                                     │
      ▼                 ▼                                     │
Display Green      Display Yellow/Red Indicator               │
Banner (98%)       Prompt Agent for KBA Verification          │
      │                 │                                     │
      └─────────────────┴──────────────────┬──────────────────┘
                                           ▼
                            Agent Desktop Fully Hydrated

```

---

## 5. Voice Biometrics Technical Deep-Dive

### 5.1 Media Forking Architecture (AudioHook Protocol)

Genesys Cloud CX uses the **AudioHook API** to establish a low-latency, bidirectional or unidirectional media stream from the Genesys Cloud Edge to external endpoints:

* **Protocol**: gRPC over HTTP/2 with TLS 1.3 or secure WebSockets (`wss://`).
* **Audio Format**: Linear PCM (L16) at 8,000 Hz or 16,000 Hz, mono or dual-channel, chunked into 20ms frames.
* **Header Metadata**: Includes `conversationId`, `participantId`, `mediaFormat`, and `correlationId`.

### 5.2 Passive vs. Active Authentication Threshold Matrix

| Feature Parameter | Passive Authentication Mode | Active Authentication Fallback |
| --- | --- | --- |
| **Audio Requirement** | 3 to 5 seconds of natural speech. | Specific passphrase (e.g., *"At High Street Bank my voice is my password"*). |
| **User Friction** | Zero friction (Background operation). | High friction (User explicitly prompted). |
| **Biometric Score Threshold** | $\ge 90\%$ confidence score for high-risk; $\ge 80\%$ for low-risk. | $\ge 85\%$ confidence score. |
| **Anti-Spoofing (Liveness)** | Synthetic voice / replay detection active on stream. | Replay detection on dynamic passphrase. |
| **Fallback Action** | Silent transition to KBA or dynamic Agentic AI prompt. | Agent manual override / PIN entry on phone key pad. |

---

## 6. Agentic AI & Function Calling Architecture

The Agentic AI framework utilizes **Vertex AI (Gemini Flash/Pro)** operating in a **ReAct (Reasoning + Acting)** execution loop, orchestrated via GCP Cloud Run microservices.

```
+-----------------------------------------------------------------------------------+
|                            REACT AGENTIC EXECUTION LOOP                           |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  1. PERCEIVE: Transcript input + Session Auth Context                             |
|  2. REASON:   Determine required action using Banking System Prompts              |
|  3. DECIDE:   Formulate Tool Call Spec (JSON) e.g. update_address(new_addr)       |
|  4. GUARD:    Validate user isAuthenticated == true & Transaction <= Limit        |
|  5. ACT:      Execute HTTP POST via Apigee Enterprise Gateway                     |
|  6. RESPOND:  Convert API Response JSON into Natural Conversational Speech        |
|                                                                                   |
+-----------------------------------------------------------------------------------+

```

### OpenAPI Tool Specification Example (Apigee Integration)

```yaml
openapi: 3.0.1
info:
  title: High Street Bank Core Service API
  version: 1.0.0
paths:
  /v1/accounts/{accountId}/refund:
    post:
      summary: Execute automated transaction refund
      operationId: processRefund
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                transactionId:
                  type: string
                amount:
                  type: number
                reason:
                  type: string
      responses:
        '200':
          description: Refund processed successfully

```

---

## 7. Security, Risk, & Compliance Architecture

```
                               SECURITY BOUNDARY
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Genesys / GCP Data in Transit    ===>   TLS 1.3 / mTLS Protocol       │
│   Data at Rest Encryption          ===>   CMEK with Cloud HSM Keys      │
│   PII / PCI Masking                ===>   Streaming Cloud DLP           │
│   Voiceprint Profile Privacy       ===>   Encrypted Hash (Irreversible) │
│   Network Isolation                ===>   VPC Service Controls (VPC-SC) │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

| Security & Compliance Domain | Architectural Enforcement Mechanism |
| --- | --- |
| **Data in Transit** | TLS 1.3 enforced across all WebSocket, gRPC, and REST communication. mTLS implemented between GCP and Apigee API Gateway. |
| **Data at Rest & Encryption** | GCP Customer-Managed Encryption Keys (CMEK) via Cloud HSM. Voiceprint DB stored as encrypted vector representations (non-reconstructible into original audio). |
| **PCI-DSS & Data Privacy** | Real-time Cloud DLP inspects transcribed streams to redact Primary Account Numbers (PAN), CVVs, and SSNs before passing payload to Vertex AI models. |
| **Regulatory Compliance (GDPR/PSD2)** | Biometric consent collected during enrollment or explicitly checked prior to enrollment verification. Strong Customer Authentication (SCA) compliance maintained via multi-factor step-up for out-of-band transfers. |
| **Network Security & Isolation** | GCP infrastructure deployed within isolated **VPC Service Controls (VPC-SC)** perimeters. Direct connectivity to bank core mainframe via dedicated **Cloud Interconnect** over IPsec VPN. |

---

## 8. Target SLAs & Latency Budgets

```
Target Total End-to-End Latency for Real-Time Agent Assist Suggestion: < 800 ms

 [Audio Capture] ---> [AudioHook Stream] ---> [Cloud DLP] ---> [Speech-to-Text] ---> [Vertex AI RAG] ---> [WS Push to Agent Desktop]
    (0 ms)                (50 ms)              (100 ms)           (250 ms)             (300 ms)               (800 ms)

```

| Operation / Path | Target SLA | Max Allowed Latency | Architectural Strategy |
| --- | --- | --- | --- |
| **AudioHook Streaming Latency** | P99 | $< 100\text{ ms}$ | Direct gRPC connection over low-latency regional interconnect (`europe-west2`). |
| **Passive Biometric Score Generation** | P95 | $< 2.0\text{ s}$ | Continuous background processing on sliding 3-second audio windows. |
| **Speech-to-Text Transcription** | P95 | $< 300\text{ ms}$ | Google Cloud Speech-to-Text V2 Telephony model with streaming gRPC. |
| **Agentic AI Tool Execution Loop** | P90 | $< 1.5\text{ s}$ | Parallel function evaluation, Cloud Run warm instances, Apigee API connection pooling. |
| **Agent Assist Co-Pilot Screen Pop** | P95 | $< 800\text{ ms}$ | Vector Search index caching, quantized Gemini models for fast prompt inference. |
| **System Availability** | Multi-Region | $99.99\%$ | Genesys Cloud CX Multi-Region deployment with GCP Dual-Region High Availability setup. |

---

## 9. Deployment Architecture & CI/CD Pipeline

The entire GCP and Genesys configuration is deployed using **Infrastructure as Code (IaC)** via Terraform and GitOps practices.

```
┌─────────────────┐       ┌──────────────────┐       ┌────────────────────────┐
│  Git Repository │ ===>  │ Terraform Engine │ ===>  │ GCP Environment        │
│  (Config & IaC) │       │ (Automated CI/CD)│       │ (Cloud Run, Vertex AI) │
└─────────────────┘       └──────────────────┘       └────────────────────────┘
                                   │
                                   ▼
                          ┌────────────────────────┐
                          │ Genesys CX Environment │
                          │ (Architect Flows, CX)  │
                          └────────────────────────┘

```

* **Genesys Cloud Infrastructure**: Managed via the `genesyscloud` Terraform Provider (CX Architect Flows, Integration Webhooks, Data Actions, AudioHook Configuration).
* **GCP Infrastructure**: Managed via Google Terraform Modules (Cloud Run, Dialogflow CX Agents, Firestore, Cloud DLP, VPC-SC, Cloud HSM).
* **Release Strategy**: Blue/Green deployments for Cloud Run microservices to guarantee zero-downtime deployment for live contact centre traffic.