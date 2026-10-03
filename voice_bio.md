Voice Authentication (Voice Biometrics) is a powerful security and friction-reduction tool. It is important to note that **Genesys Cloud CX does not have a native, built-in voice biometrics engine**. Instead, it acts as the orchestration and telephony layer, relying on integrations with specialized third-party providers like **Veridas, Auraya, Pindrop, or Nuance**.

Here is a comprehensive breakdown of how Voice Authentication works across different flows and how to architect the integration with platforms like Veridas or Auraya.

---

### Part 1: How Voice Authentication Works Across Flows

Voice biometrics generally operates in two modes: **Active** (the user is prompted to say a specific passphrase) and **Passive** (the user's voice is analyzed continuously during natural conversation). 

#### 1. In an IVR Flow (Genesys Architect)
*   **Active Authentication:** The Architect flow plays a prompt: *"For security, please say your first and last name."* The system records the audio snippet, sends it to the biometrics engine via a REST API (Data Action), and receives a response (Match/No Match). The flow then branches based on the result.
*   **Passive Authentication:** As the caller navigates the IVR, the system silently forks the audio stream to the biometrics engine in the background. If the engine reaches a high confidence score, it sends a webhook/event back to Genesys, which can dynamically alter the flow (e.g., skipping security questions and routing directly to a Tier 2 queue).

#### 2. In a Human Agent Flow (Agent Desktop)
*   **Agent Assist & Verification:** When a call is routed to an agent, the audio is streamed to the biometrics engine. 
*   **Screen Pops & Alerts:** If the caller is successfully authenticated passively, a notification pops up on the agent’s Genesys desktop (e.g., *"Caller Verified: 98% Confidence"*). 
*   **Agent Override:** If the biometrics engine fails to verify the caller (e.g., background noise, cold/illness), the agent can manually trigger an Active Authentication prompt from their desktop, or fall back to traditional Knowledge-Based Authentication (KBA) like asking for a PIN.

#### 3. In an Agentic AI / Voicebot Flow
*   **Conversational Context:** The AI Voicebot (e.g., Genesys Cloud CX AI or a third-party bot like Dialogflow integrated via Data Actions) engages the user. 
*   **Seamless Handoff:** The bot asks a natural question (*"To get started, could you just tell me your name and date of birth?"*). As the user speaks, the audio is sent to the biometrics engine. 
*   **State-Driven Actions:** Once the biometrics engine confirms the voiceprint matches the CRM record, the AI agent updates its internal state to `isAuthenticated = true` and proceeds to execute secure tasks (e.g., processing a refund, changing a password) without ever asking for a PIN or password.

---

### Part 2: Integration Architecture with Veridas or Auraya

To integrate Genesys Cloud CX with a platform like **Veridas** or **Auraya**, you must bridge the telephony audio with their biometric APIs. The architecture depends on whether you are doing Active or Passive authentication.

#### Architecture A: Active Authentication (REST API via Data Actions)
Used when the caller is prompted to say a specific phrase.
1.  **Audio Capture:** Genesys Architect records the caller's audio into a temporary audio file (WAV/MP3).
2.  **Data Action Execution:** Architect triggers a custom **Data Action**.
3.  **API Call:** The Data Action sends a POST request to the Veridas/Auraya REST API. The payload includes the audio file (or a URL to the file) and the customer's unique identifier (e.g., Phone Number/ANI or CRM Account ID).
4.  **Response Handling:** The biometrics engine processes the audio and returns a JSON response (e.g., `{"status": "VERIFIED", "confidence": 95}`).
5.  **Flow Branching:** Architect maps this response to a flow variable and uses an "If/Else" decision component to route the call accordingly.

#### Architecture B: Passive Authentication (Real-Time Audio Streaming / Media Forking)
Used for continuous, background authentication during a live call (IVR or Agent).
1.  **Media Forking / Audio Streaming:** Genesys must duplicate the live audio stream. In Genesys Cloud CX, this is done using **Audio Streaming** (via WebSockets) or **SIPREC / Media Forking** configurations on the Edge/SIP trunk.
2.  **WebSocket Connection:** Genesys opens a persistent WebSocket connection to the Auraya/Veridas streaming endpoint.
3.  **Continuous Streaming:** Raw audio chunks (PCM) are streamed in real-time to the biometrics engine as the caller speaks.
4.  **Asynchronous Events:** The biometrics engine analyzes the stream. When it reaches a confidence threshold, it sends an asynchronous event (via Webhook or WebSocket message) back to Genesys.
5.  **Event Handling:** Genesys receives the event via an **Integration Webhook** or updates a custom data attribute on the interaction, which triggers a screen pop for the agent or alters the IVR routing.

---

### Part 3: Step-by-Step Implementation Guide

If you are integrating Veridas or Auraya into Genesys Cloud CX, follow these steps:

#### Step 1: Check Genesys AppFoundry
Before building from scratch, check the **Genesys AppFoundry**. Major biometric providers often have pre-built integration apps. 
*   *Why?* A pre-built app will handle the complex media forking, WebSocket management, and Data Action configurations out-of-the-box, reducing implementation time from weeks to days.

#### Step 2: Configure the Telephony / Edge for Audio Streaming (If Passive)
If a pre-built app isn't available and you need passive authentication:
*   Work with your Genesys SIP trunk provider or configure your Genesys Edge appliances to enable **Media Forking** or **Audio Streaming**.
*   Ensure your network allows outbound WebSocket connections from the Genesys Edge to the Veridas/Auraya cloud or on-premise servers.

#### Step 3: Build Custom Data Actions (For Active)
*   Go to **Admin > Integrations > Data Actions**.
*   Create a new Data Action. Define the REST API contract based on the Veridas/Auraya API documentation.
*   **Input:** `caller_ani` (string), `audio_file_url` (string).
*   **Output:** `auth_status` (string), `confidence_score` (number).
*   Configure the authentication (e.g., OAuth2 or API Key) required by the biometrics provider.

#### Step 4: Map Variables in Architect / Conversational AI
*   In **Architect**, add the Data Action to your Inbound Call Flow.
*   Pass the `Call.ANI` (caller ID) and the recorded audio to the Data Action.
*   Save the output to a Flow variable (e.g., `Flow.VoiceAuthStatus`).
*   Add a Decision component: *If `Flow.VoiceAuthStatus` == "VERIFIED", transfer to Queue. Else, play prompt for PIN.*

#### Step 5: Agent Desktop Integration (CRM Pop)
*   If using a CRM (Salesforce, Dynamics), use the **Genesys CRM Connector** or **Embedded Framework**.
*   Map the `confidence_score` returned by the biometrics engine to a custom field in the CRM.
*   Configure the CRM layout to display a "Verified" badge if the score is above the provider's recommended threshold (usually 85-90%).

---

### Part 4: Critical Best Practices & Considerations

1.  **Define Clear Thresholds:** Work with Veridas/Auraya to define your "Accept", "Reject", and "Review" thresholds. A 90% match might auto-verify, 70-89% might require an agent to ask one KBA question, and <70% requires full manual authentication.
2.  **Enrollment Strategy:** Voice biometrics requires a voiceprint. 
    *   *First Call:* The system must capture the audio and send it to the provider's **Enrollment API** to create the voiceprint, linking it to the customer's CRM ID.
    *   *Subsequent Calls:* The system uses the **Verification API**.
3.  **Fallback Mechanisms:** Voice biometrics is not 100% perfect. Callers may have a cold, be in a noisy environment, or experience network jitter. **Always design a graceful fallback** in Architect (e.g., "I'm having trouble verifying your voice, let's use your 4-digit PIN instead").
4.  **Privacy and Consent:** Voiceprints are considered biometric data and are heavily regulated (GDPR, CCPA, BIPA). 
    *   Your IVR flow **must** include a consent prompt before capturing the audio (e.g., *"For security purposes, your voice will be recorded and analyzed. Do you consent? Press 1"*).
    *   Ensure your integration with Veridas/Auraya complies with data residency requirements (e.g., ensuring audio is processed in the correct geographic region).
5.  **Liveness Detection:** Ensure the Veridas/Auraya integration includes **Liveness Detection** (anti-spoofing) to prevent attackers from using recorded audio of the legitimate customer to bypass authentication.
