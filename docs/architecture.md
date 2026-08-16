# Architecture — AI Lead Capture & Management Chatbot

## Overview

This document describes the technical architecture of the AI Lead Capture & Management Chatbot, implemented as an n8n workflow. The system combines a conversational AI agent with a structured automation pipeline to capture, store, and notify on inbound leads without any manual intervention.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            LOCAL ENVIRONMENT                                │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────────────────────────────────────┐   │
│  │   Browser   │     │                   n8n (Docker)                  │   │
│  │  (User /    │────▶│                                                 │   │
│  │  Chat UI)   │     │  Chat Trigger → AI Agent → If → Extractor →    │   │
│  └─────────────┘     │                                  Google Sheets  │   │
│                      │                                  Gmail (x2)     │   │
│                      └─────────────────────────────────────────────────┘   │
│                                         │                                   │
│                                         ▼                                   │
│                      ┌─────────────────────────────────┐                   │
│                      │         Ollama Server            │                   │
│                      │   Model: llama3.2               │                   │
│                      │   Port: 11434                   │                   │
│                      └─────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                   ┌─────────────────────┴──────────────────────┐
                   ▼                                             ▼
        ┌──────────────────────┐                   ┌────────────────────────┐
        │    Google Sheets     │                   │         Gmail          │
        │  (Lead CRM Storage)  │                   │ (Email Notifications)  │
        └──────────────────────┘                   └────────────────────────┘
```

---

## Workflow Nodes — Detailed Specification

### 1. When chat message received
- **Type:** `@n8n/n8n-nodes-langchain.chatTrigger` (v1.4)
- **Role:** Entry point for every user message
- **Mechanism:** Exposes a webhook endpoint; the n8n built-in chat widget sends messages to this endpoint on each user submission
- **Output:** `{ chatInput: "<user message>", sessionId: "<session>" }`

---

### 2. AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
- **Role:** Drives the multi-turn lead capture conversation
- **Configuration:**
  - System prompt defines strict collection rules for: Name, Email, Phone, Service
  - Progressive collection: only asks for the next missing field
  - Completion trigger: outputs the phrase *"Thanks! I've captured your details. Our sales team will contact you shortly."* only when all four fields are confirmed
  - Email preservation: never converts email to Markdown link
  - Phone preservation: never modifies digits
- **Connected inputs:**
  - `ai_languageModel`: Ollama Chat Model
  - `ai_memory`: Simple Memory
- **Output:** `{ output: "<agent response text>" }`

---

### 3. Ollama Chat Model (for AI Agent)
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOllama` (v1)
- **Role:** Provides Llama 3.2 language model inference for the AI Agent
- **Credential:** `ollamaApi` (configured to point to local Ollama instance)

---

### 4. Simple Memory
- **Type:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (v1.4)
- **Role:** Maintains a rolling window of the full conversation history within a session
- **Scope:** Session-level; memory is cleared when a new session begins
- **Connected to:** AI Agent (`ai_memory` input)

---

### 5. If Node
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Role:** Acts as a completion gate — only allows the pipeline to proceed when the AI Agent has confirmed all four fields
- **Condition:**
  ```
  $json.output CONTAINS "Thanks! I've captured your details. Our sales team will contact you shortly."
  ```
- **True branch:** → Information Extractor
- **False branch:** (no connection — message is returned to user; conversation continues)

---

### 6. Information Extractor
- **Type:** `@n8n/n8n-nodes-langchain.informationExtractor` (v1.2)
- **Role:** Second LLM pass — parses the AI Agent's final message and extracts the four fields into structured JSON
- **Input text:** `$json.output` (the AI Agent's completion message)
- **Output schema:**
  ```json
  {
    "name": "string",
    "email": "string",
    "phone": "string",
    "service": "string"
  }
  ```
- **Connected LLM:** Ollama Chat Model1

---

### 7. Ollama Chat Model1 (for Information Extractor)
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOllama` (v1)
- **Role:** Provides Llama 3.2 inference for the Information Extractor
- **Credential:** Same `ollamaApi` credential as the primary model

---

### 8. Append row in sheet
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Role:** Appends a new row to the configured Google Sheet
- **Columns mapped:**
  | Column | Source |
  |---|---|
  | Name | `$json.output.name` |
  | Email | `$json.output.email` |
  | Phone | `$json.output.phone` |
  | Service | `$json.output.service` |
  | Date | `$now.setZone('Asia/Kolkata').toFormat('yyyy-MM-dd HH:mm:ss')` |
- **Credential:** Google Sheets OAuth2

---

### 9. Send a message (Lead Confirmation)
- **Type:** `n8n-nodes-base.gmail` (v2.2)
- **Role:** Sends a confirmation email to the lead
- **Recipient:** `$json.Email` (the lead's own email address)
- **Subject:** `Thank you for contacting us {{ Name }}`
- **Credential:** Gmail OAuth2

---

### 10. Send a message1 (Sales Notification)
- **Type:** `n8n-nodes-base.gmail` (v2.2)
- **Role:** Notifies the sales team of the new lead
- **Recipient:** Sales team inbox (configured at setup)
- **Subject:** `New Lead Received — {{ name }}`
- **Body:** Includes all four lead fields and a follow-up prompt
- **Credential:** Gmail OAuth2

---

## Data Flow Diagram

```
[User] ─── message ──▶ [Chat Trigger]
                              │
                         chatInput
                              │
                              ▼
                        [AI Agent] ◀────── [Simple Memory]
                              │                   │
                         [Ollama LLM]        (conversation
                              │                 history)
                              │
                         AI response
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
               (ongoing          "Thanks! I've
               conversation)      captured..."
                                       │
                                       ▼
                                   [If Node]
                                   condition
                                   = true
                                       │
                                       ▼
                          [Information Extractor]
                                  │
                             [Ollama LLM1]
                                  │
                           structured JSON
                           { name, email,
                             phone, service }
                                  │
                                  ▼
                       [Google Sheets — Append]
                              │         │
                              ▼         ▼
                       [Gmail:       [Gmail:
                        Lead          Sales
                        Confirm]      Notify]
```

---

## Security Considerations

- **No external LLM APIs**: All inference is local via Ollama — no data leaves the machine
- **OAuth2 credentials**: Google Sheets and Gmail use n8n's OAuth2 integration — no raw tokens stored in the workflow file
- **Credential references**: The workflow JSON stores only n8n internal credential IDs, not actual secrets
- **Lead data**: Real customer data should never be committed to the repository

---

## Deployment Notes

| Component | Recommended Setup |
|---|---|
| n8n | Docker container, port 5678 |
| Ollama | Native install or Docker, port 11434 |
| Google Sheets | OAuth2 via n8n credentials UI |
| Gmail | OAuth2 via n8n credentials UI |
| Network | If n8n runs in Docker, use `host.docker.internal` for Ollama |
