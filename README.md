# 🤖 Voice AI Dispatcher — FixRight Appliances (Houston, TX)

Autonomous AI voice agent that handles **100% of inbound service calls** — qualifies leads, checks service zones, and books appointments in Google Calendar in real time, without any human involvement.

---

## 📌 Project Overview

| | |
|---|---|
| **Client** | FixRight Appliances — appliance repair, Houston TX |
| **My role** | AI automation developer (solo) |
| **Stack** | ElevenLabs Conversational AI · Twilio · n8n · Google Calendar API |
| **Result** | Fully automated inbound call handling, 24/7 |

---

## 🎯 Business Problem

The client had a live dispatcher who:
- missed calls outside business hours
- wasted time on out-of-zone and spam calls
- manually checked calendar and booked appointments

**Goal:** Replace the dispatcher with an AI voice agent that handles the full call flow autonomously — sounds natural, qualifies leads, and books real appointments.

---

## ⚙️ Architecture

```
📞 Inbound Call
      ↓
 Twilio (+1 346 214 1235)
      ↓  webhook → api.us.elevenlabs.io/twilio/inbound_call
 ElevenLabs Conversational AI
      │  Voice: Adam (Authentic & Engaging)
      │  LLM: GPT-4.1
      │
      ├─ [tool: time] ──────────────→ n8n webhook → current CT time
      ├─ [tool: Get_slots] ─────────→ n8n webhook → Google Calendar free slots
      └─ [tool: PostEvent] ─────────→ n8n webhook → create Calendar event
                                                   → Telegram notification
```

**No intermediate server.** Twilio routes the call directly to ElevenLabs via native endpoint — eliminates latency and infrastructure overhead.

---

## 🛠️ Tech Stack

| Layer | Tool |
|---|---|
| Voice AI platform | ElevenLabs Conversational AI |
| LLM | GPT-4.1 (via ElevenLabs) |
| Voice | Adam — Authentic & Engaging |
| Telephony | Twilio (inbound calls) |
| Orchestration | n8n (webhook-based tools) |
| Calendar | Google Calendar API |
| Notifications | Telegram Bot API |

---

## 🔧 Agent Tools (n8n webhooks)

The agent has 3 custom tools, each backed by an n8n workflow:

### `time`
Returns current date and time in CT (America/Chicago).  
Called silently at conversation start and before booking — ensures the agent never proposes past or impossible time slots.

### `Get_slots`
Checks Google Calendar for available technician slots.  
Returns availability in **3-hour windows only**: 8–11 AM, 11 AM–2 PM, 2–5 PM, 5–8 PM.  
Not called for out-of-zone, commercial, or unsupported appliance scenarios — cost optimization.

### `PostEvent`
Creates a calendar event after full client confirmation.  
Input: `appointment_datetime_start`, `appointment_datetime_end`, `event_summary`, `notes`.  
Triggers a Telegram notification to the manager after successful booking.

---

## 🧠 Key Logic

**ZIP-code qualification (44 service zones)**  
First thing the agent checks. If ZIP is not on the list — immediate polite refusal, no further data collected, no tools called.

**Spam/robocall protection**  
Handled at the Twilio level before the call reaches ElevenLabs — prevents unnecessary AI costs (~$0.10–0.20 saved per spam call).

**Time-aware slot proposal**  
Agent calls `time` tool, then applies rules: no same-day slots after 5 PM CT, minimum 60-minute lead time, only 3-hour windows offered.

**Commercial vs residential check**  
Asked right after ZIP. If commercial — agent refuses and closes the call without collecting any further data.

**Unsupported appliance check**  
Validated against a fixed list (refrigerator, washer, dryer, oven, dishwasher, cooktop, etc.). Coffee machines, TVs, grills — refused immediately.

**Fallback scenarios**  
If `Get_slots` fails → agent takes manual info, promises manager callback within 1 hour.  
If `PostEvent` fails → agent confirms manually, promises callback within 30 minutes.

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
  Get_slots → Propose window
       ↓
  Full confirmation: "Month Day, Year + window + address"
       ↓
  time (silent) + PostEvent (silent)
       ↓
  "You're scheduled for [date] [window].
   Technician calls 30 min before arrival.
   Diagnostic is $65, applied toward repair."
```

---

## 📂 Repository Structure

```
/
├── README.md
├── prompts/
│   └── system-prompt.md          ← Full agent system prompt
├── tools/
│   └── agent-tools.json          ← Tool definitions (ElevenLabs format)
└── screenshots/
    ├── elevenlabs-agent-config.png
    ├── elevenlabs-agent-tools.png
    └── twilio-voice-configuration.png
```

---

## 💡 What Made This Project Interesting

- **No intermediate server** — Twilio routes directly to ElevenLabs native endpoint; no Node.js proxy, no Railway, no extra infrastructure
- **Time-aware logic in prompts** — agent correctly handles edge cases like "it's 4:58 PM, can I still book today?" without hallucinating impossible slots
- **Strict data collection sequence** — ZIP → service type → name → phone → address → appliance → issue → slots; agent never skips or re-asks confirmed fields
- **Cost optimization** — tools not called for disqualified callers (out-of-zone, commercial, unsupported appliance), spam filtered at telephony layer

---

## 📊 Outcome

- ✅ 100% inbound calls handled without human dispatcher
- ✅ Out-of-zone and spam calls filtered before consuming AI budget
- ✅ Real-time appointment booking directly into Google Calendar
- ✅ 24/7 availability vs. previous business-hours-only coverage

---

*Built by [iooshunory](https://www.upwork.com/freelancers/iooshunory) — Voice AI & Automation Developer*
