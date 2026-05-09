# SafeRoute AI — Hackolution Urban Safety Agent

An AI-powered urban safety system built for Kolkata that analyses live GPS coordinates, assesses crime risk in real time, and recommends the safest route to your destination — all via Telegram and a mobile web UI.

---

## Overview

SafeRoute AI is a RAG-based safety agent built on n8n. When a user shares their live location on Telegram, the system reverse-geocodes it, queries a Pinecone vector database of local crime incidents, and uses Google Gemini to classify the area as LOW, MEDIUM, or HIGH risk. If the risk is HIGH, an automated SOS alert flow is triggered with a 5-minute confirmation window. For route planning, the system uses Apify to scrape real-time local data and generates a structured safe-route response.

---

## Architecture

```
Telegram (user shares location)
        │
        ▼
   [Telegram Trigger]
        │
        ▼
   [If2] ── location exists? ──────────────────────────────┐
        │ YES                                               │ NO (text message)
        ▼                                                   ▼
   [AI Agent]  ←── Google Gemini Chat Model           [Code JS1]
        │       ←── Pinecone Vector Store (RAG)              │
        │       ←── Simple Memory (session)                  ▼
        ▼                                          [Apify Actor Run]
   [Code JS]  ── parse risk_level JSON                      │
        │                                                    ▼
        ▼                                             [Code JS2]
      [If] ── risk == HIGH?                                  │
        │ YES                                                ▼
        ▼                                            [AI Agent1] ← Gemini + Pinecone
  [Send Alert]  ── "⚠️ You're in a high-risk area"          │
        │                                                    ▼
        ▼                                             [Code JS3]
      [Wait 20s]                                            │
        │                                                   ▼
        ▼                                        [Send Route Message]
  [HTTP Request] ── poll Telegram for reply                 │
        │                                                   ▼
        ▼                                        [Send Safety Tips]
     [If1] ── user replied "Yes"?
        │ NO
        ▼
  [HTTP Request1] ── trigger emergency call
        │
        ▼
  [Make a call] → [Wait1] → [Make a call1]
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow automation | n8n |
| AI language model | Google Gemini (via PaLM API) |
| Vector database | Pinecone (`hackolution` index) |
| Embeddings | Google Gemini Embeddings |
| Messaging | Telegram Bot API |
| Web scraping | Apify Actor |
| Session memory | n8n Buffer Window Memory |
| Frontend UI | Vanilla HTML/CSS/JS |

---

## Features

**Area Safety Check**
- User shares live GPS coordinates via Telegram
- AI agent queries Pinecone for nearby crime incidents within 2–5 km radius
- Risk classified as LOW / MEDIUM / HIGH using Gemini
- Considers time of day (late-night raises risk score)

**Automated SOS Alert Flow**
- Triggers on HIGH risk detection
- Sends a Telegram warning message with a 5-minute response window
- Polls for user confirmation ("Yes = safe")
- If no confirmation, escalates to emergency phone calls (×2 attempts)

**Safe Route Planning**
- Triggered when user sends a text message (source → destination)
- Apify scrapes real-time local data: hospitals, police stations, risky zones
- Gemini generates a structured Telegram-friendly route response
- Includes nearby hospitals, police stations, risky zones, and safety advice

**Mobile Web UI** (`urban_safety_agent.html`)
- Standalone single-file frontend — no build step required
- Three panels: Area Check, Safe Route, Alert
- Live countdown timer for SOS confirmation
- Emergency quick-call buttons (Police 100, SSKM, Women's Helpline 1091)

---

## Project Structure

```
hackolution/
├── rag_Hackolution.json          # n8n workflow export
├── urban_safety_agent.html       # Mobile web UI (self-contained)
└── README.md                     # This file
```

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Telegram Bot token (from [@BotFather](https://t.me/BotFather))
- Google Gemini / PaLM API key
- Pinecone account with an index named `hackolution`
- Apify account + API token

### 1. Import the workflow

In your n8n instance, go to **Workflows → Import** and upload `rag_Hackolution.json`.

### 2. Configure credentials

| Credential | Where to add |
|---|---|
| `telegramApi` | Settings → Credentials → Telegram API |
| `googlePalmApi` | Settings → Credentials → Google Gemini (PaLM) |
| `pineconeApi` | Settings → Credentials → Pinecone |

### 3. Set up Pinecone

Create an index named `hackolution` and populate it with crime incident records for your city. Each record should include fields like `type`, `location`, `date`, `description`, and coordinates.

### 4. Activate the workflow

Toggle the workflow to **Active** in n8n. The Telegram webhook will register automatically.

### 5. Open the UI

Open `urban_safety_agent.html` directly in any browser — no server needed.

---

## Usage

**Via Telegram**

1. Start a chat with your bot
2. Share your live location → receive a risk assessment
3. Send a message like `From Esplanade to Salt Lake` → receive a safe route

**Via Web UI**

1. Open `urban_safety_agent.html`
2. Tap **Analyse Area Safety** on the Area Check tab
3. Enter source and destination on the Route tab
4. The Alert tab shows the SOS countdown and emergency contacts

---

## Risk Level Logic

| Level | Condition |
|---|---|
| LOW | Few or no incidents within radius, daytime |
| MEDIUM | Moderate incidents, or isolated location at night |
| HIGH | Repeated violent crime, known hotspot, or late-night isolation |

Radius used for incident lookup: **2 km** in dense urban areas, **5 km** in sparse areas.

---

## Emergency Contacts (Kolkata)

| Service | Number |
|---|---|
| Police | 100 |
| Women's Helpline | 1091 |
| SSKM Hospital | 033-2223-0000 |
| Ambulance | 102 |

---

## Built At

**Hackolution Hackathon** — AI for Urban Safety track  
Stack: n8n · Google Gemini · Pinecone · Telegram · Apify
