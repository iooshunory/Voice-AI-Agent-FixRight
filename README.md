# 🤖 Voice AI Dispatcher — FixRight Appliances (Houston, TX)

Autonomous AI voice agent that handles **100% of inbound service calls** — filters spam, qualifies leads, checks service zones, books appointments in Google Calendar in real time, and sends the manager a structured analysis of every call. Zero human dispatcher involvement.

---

## 📌 Project Overview

| | |
|---|---|
| **Client** | FixRight Appliances — appliance repair, Houston TX |
| **My role** | AI automation developer (solo) |
| **Stack** | ElevenLabs · Twilio · n8n · Google Calendar API · OpenAI · Telegram |
| **Result** | 24/7 automated call handling + post-call analytics pipeline |

---

## 🎯 Business Problem

The client had a live dispatcher who:
- missed calls outside business hours
- wasted time on robocalls and out-of-zone callers
- manually checked availability and booked appointments
- had no visibility into what was said on missed or failed calls

**Goal:** Replace the dispatcher with an AI agent that sounds natural, handles the full call flow autonomously, and gives the manager full post-call visibility via Telegram.

---

## ⚙️ Architecture

```
📞 Inbound Call
      ↓
 Twilio
      ↓
 [n8n] pre_call_spam_filter
      │  → Twilio Lookup API + Nomorobo add-on
      │  → spam_score > 0: call rejected (~$0.10–0.20 saved per spam call)
      │  → spam_score = 0: proceed
      ↓
 ElevenLabs Conversational AI
      │  Voice: Adam (Authentic & Engaging)
      │  LLM: GPT-4.1
      │  Connected via native Twilio webhook (no intermediate server)
      │
      ├─ [tool: time] ──────────→ n8n → current CT date/time
      ├─ [tool: Get_slots] ─────→ n8n → Google Calendar → available 3-hr windows
      └─ [tool: PostEvent] ─────→ n8n → create Calendar event
                                       → Telegram notification to manager
      ↓
 [n8n] post_call_analysis
      │  ElevenLabs post-call webhook → full transcript + audio
      │  → Code node parses transcript
      │  → GPT-4.1-mini: booking status, ZIP zone, appliance, issue
      │  → Structured summary + audio file → manager's Telegram
```

**No intermediate server.** Twilio connects directly to ElevenLabs via native endpoint — no Node.js proxy, no Railway, no extra infrastructure.

---

## 🛠️ Tech Stack

| Layer | Tool |
|---|---|
| Voice AI platform | ElevenLabs Conversational AI |
| LLM (agent) | GPT-4.1 (via ElevenLabs) |
| LLM (post-call analysis) | GPT-4.1-mini (via n8n) |
| Voice | Adam — Authentic & Engaging |
| Telephony | Twilio |
| Orchestration | n8n (self-hosted on Elestio) |
| Calendar | Google Calendar API (OAuth2) |
| Spam filtering | Twilio Lookup API + Nomorobo add-on |
| Notifications | Telegram Bot API |

---

## 🔧 n8n Workflows

### 1. `pre_call_spam_filter`
Pre-call webhook triggered by Twilio before connecting to ElevenLabs.
Queries Twilio Lookup API with Nomorobo add-on to get a spam score for the caller ID.
If `spam_score > 0` → rejects the call before it reaches the AI agent.
**Cost saving:** ~$0.10–0.20 per filtered robocall.

### 2. `time`
Returns current date and time in CT (America/Chicago).
Called silently at conversation start and again before booking — prevents the agent from proposing impossible or past time slots.

### 3. `Get_slots`
Queries Google Calendar for available technician slots.
A JavaScript Code node calculates availability across **3-hour windows**: 8–11 AM, 11 AM–2 PM, 2–5 PM, 5–8 PM.
Marks slots as `free`, `busy`, or `past` based on current CT time.
Not called for out-of-zone, commercial, or unsupported appliance scenarios — saves unnecessary API calls.

### 4. `PostEvent`
Creates a Google Calendar event after the agent collects and confirms all booking details.
Maps all fields from the agent: name, phone, address, gate code, appliance, issue, brand/model, time window.
Triggers a formatted Telegram notification to the manager immediately after booking.

### 5. `post_call_analysis` (bonus)
Activated by ElevenLabs post-call webhook after every conversation.
Receives full transcript + audio recording.
A Code node parses the transcript with timestamps and speaker roles.
GPT-4.1-mini analyzes the call: booking outcome, ZIP zone status, appliance type, issue description.
Sends structured summary + audio file to manager's Telegram channel.

---

## 🧠 Agent Logic

**ZIP-code qualification (44 service zones)**
First thing the agent checks. If ZIP is not on the list — immediate polite refusal, no further data collected, no tools called.

**Residential vs commercial check**
Asked right after ZIP. If commercial — agent refuses and closes the call without collecting any further data.

**Unsupported appliance check**
Validated against a fixed list. Coffee machines, TVs, grills, water heaters — refused immediately after being named.

**Time-aware slot proposal**
Agent calls `time` tool first, then applies rules: no same-day slots after 5 PM CT, minimum 60-minute lead time, slots locked 1 minute after window start.

**Strict data collection sequence**
ZIP → service type → name → phone → address + gate code → appliance → issue → brand/model → `Get_slots`.
Breaking the sequence is forbidden — agent never skips or re-asks confirmed fields.

**Fallback handling**
If `Get_slots` fails → takes manual info, promises manager callback within 1 hour.
If `PostEvent` fails → confirms manually, promises callback within 30 minutes.

---

## 📋 Conversation Flow

```
Start
  └─ time (silent)
  └─ "What's going on with your appliance?"
       ↓
  ZIP code → [out-of-zone? → STOP]
       ↓
  Residential or commercial? → [commercial? → STOP]
       ↓
  Full name → Phone → Address + gate code
       ↓
  Appliance type → [unsupported? → STOP]
       ↓
  Problem description → Brand/model
       ↓
  Get_slots → Propose 3-hour window
       ↓
  Full confirmation: "Month Day, Year + window + address"
       ↓
  time (silent) + PostEvent (silent)
       ↓
  "You're scheduled for [date] [window].
   Technician calls 30 min before arrival.
   Diagnostic is $65, applied toward repair."
       ↓
  [post-call] ElevenLabs webhook → n8n → GPT-4.1-mini → Telegram
```

---

## 📂 Repository Structure

```
/
├── README.md
├── prompts/
│   └── system-prompt.md          ← Agent system prompt (anonymized)
├── tools/
│   └── agent-tools.json          ← n8n workflow export (all 5 workflows)
└── screenshots/
    ├── agent.png                  ← ElevenLabs agent configuration
    ├── agent2.png                 ← ElevenLabs agent tools
    └── Twilio.png                 ← Twilio voice configuration
```

---

## 💡 What Made This Project Interesting

- **No intermediate server** — Twilio routes directly to ElevenLabs native endpoint; no Node.js proxy, no Railway, no extra infrastructure
- **Spam filtering before AI costs** — Nomorobo integration at the telephony layer means robocalls never reach ElevenLabs and never generate charges
- **Time-aware slot logic in prompts** — agent correctly handles "it's 4:58 PM, can I still book today?" without hallucinating
- **Post-call analytics pipeline** — every conversation is automatically transcribed, parsed, and analyzed by GPT-4.1-mini; manager gets a Telegram summary within seconds of call ending
- **Strict conversation state machine** — data collected in fixed sequence, agent never backtracks or skips confirmed fields

---

## 📊 Outcome

- ✅ 100% inbound calls handled without human dispatcher
- ✅ Spam and robocalls filtered before consuming AI budget
- ✅ Real-time appointment booking directly into Google Calendar
- ✅ Manager receives structured call summary + audio after every conversation
- ✅ 24/7 availability vs. previous business-hours-only coverage
