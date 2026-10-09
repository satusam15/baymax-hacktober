# baymax-hacktober
# Circular Buddy

**Snap a school notice. Get the dates, the fees and a calendar entry, right inside Telegram.**

Built by Team Baymax (Sathvik G, Vinod, Srirama Rohit) at Hacktoberfest Hack Day Bengaluru '26, hosted at MSRIT.
Tracks: **Best Use of Gemma 4** and **Best Open-Source AI Project**.

---

## The problem

Parents get school circulars as photos in WhatsApp groups. The important bits (a date, a fee deadline, "please bring your ID card") are buried in dense paragraphs, often in Kannada or Hindi. Nobody retypes them into a calendar, so things get missed.

## What Circular Buddy does

You send a photo of a notice to a Telegram bot. It replies with:

- **Events** with date, time and venue
- **Deadlines** and **fees** (with amounts)
- **Documents to carry** and **which classes** the notice applies to
- A **Google Maps link** for the venue
- A **weather** note for the event day (when the event is within the forecast range)
- A question: *"Add these dates to your calendar?"* Nothing is added until you tap **Yes, add**.

### Why you can trust it

- **No OCR step.** The photo goes straight into Gemma 4's vision.
- **Every fact quotes its source.** Each item comes with the exact line it was read from, plus a confidence score.
- **It does not guess.** If confidence is low or a date is unclear, the bot asks for a clearer photo.
- **You stay in control.** Calendar events are created only after you approve.

---

## How it works

```
Telegram message
      |
   Router (photo or text?)
      |
 +----+---------------------------+
 |                                |
Photo                           Text
 |                                |
Gemma 4 reads the image         Groq: intent, then answer
 |                              (saves Gemini tokens)
Groq turns Gemma's notes
into clean JSON
 |
Confidence check  --low-->  "Please send a clearer photo"
 |
Maps link + weather for each event
 |
Summary + "Add to calendar?" (Telegram approval)
 |
Google Calendar events
```

| Part | What we use |
|---|---|
| Vision model | Gemma 4 (`gemma-4-26b-a4b-it`) through the Gemini API |
| Text model | Groq `openai/gpt-oss-120b` (structuring, summaries, answers) and `openai/gpt-oss-20b` (intent) |
| Orchestration | [n8n](https://n8n.io) (open source, self-hosted) |
| Chat interface | Telegram bot |
| Calendar | Google Calendar API |
| Maps | Google Maps search links (optional Google Geocoding for verified addresses) |
| Weather | [Open-Meteo](https://open-meteo.com) (free, no key) |
| Public webhook for local n8n | Cloudflare quick tunnel (`cloudflared`) |

---

## Setup

### What you need

- Node.js and n8n (`npm install -g n8n`) or Docker
- `cloudflared` for a public HTTPS URL to your local n8n
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A Gemini API key from [Google AI Studio](https://aistudio.google.com/)
- A Groq API key from [console.groq.com](https://console.groq.com/)
- A Google Cloud project with the **Google Calendar API** enabled and an OAuth client (type: Web application)

### 1. Start a tunnel

Telegram can only call HTTPS webhooks, so give your local n8n a public address:

```bash
cloudflared tunnel --url http://localhost:5678
```

Copy the `https://....trycloudflare.com` link it prints. The link changes every time you restart the tunnel.

### 2. Start n8n with the tunnel address

```bash
export WEBHOOK_URL="https://YOUR-TUNNEL.trycloudflare.com/"
export N8N_EDITOR_BASE_URL="http://localhost:5678"
export NODE_OPTIONS="--dns-result-order=ipv4first"
n8n start
```

- `WEBHOOK_URL` is what Telegram calls.
- `N8N_EDITOR_BASE_URL` keeps the Google sign-in redirect on `localhost`.
- `NODE_OPTIONS` avoids IPv6 connection errors to Telegram on some networks.

### 3. Import the workflow

In n8n open **Workflows → Import from file** and choose `workflows/circular_buddy.json`.

### 4. Add your credentials

| Credential | Type | Used by |
|---|---|---|
| Telegram | Telegram API (bot token) | Telegram Trigger and all Telegram nodes |
| Gemini API Key | Header Auth, name `x-goog-api-key` | Gemma Extract |
| Groq API Key | Header Auth, name `Authorization`, value `Bearer YOUR_KEY` | Groq Structure, Groq Intent, Groq Answer, Groq Friendly Summary |
| Google Calendar | Google Calendar OAuth2 | Create Calendar Event |
| Google Maps API Key (optional) | Query Auth, name `key` | Google Geocode |

Google OAuth notes:

- Redirect URI to register: `http://localhost:5678/rest/oauth2-credential/callback`
- While the consent screen is in "Testing", add your Google account under **Test users**.

Google Geocoding needs billing on Google Cloud. If you skip it (use a dummy value for the key), the bot still works: it falls back to a plain Google Maps search link for the venue.

### 5. Run it

1. Open the Telegram Trigger node and click **Listen for test event**, or **Publish** the workflow.
2. Message your bot `/start`.
3. Send a photo of a school notice.
4. Read the summary, tap **Yes, add**, and check your Google Calendar.

A fictional sample circular for testing is in `samples/` (if you added it).

---

## Things we learned

- Gemma 4 reads the notice photo well, but it tends to answer in prose instead of strict JSON. We keep Gemma for what it is best at (seeing) and let a small Groq step convert its notes into JSON.
- Telegram compresses photos, so we use the ~800px size. Clear, well-lit photos work best.
- Free-tier quotas are real. Plain text chat goes to Groq so the Gemma calls are saved for photos.

## Roadmap

- **Now (hackathon MVP):** Telegram bot, Gemma 4 photo reading, Maps and weather, approval-gated Google Calendar
- **Next:** handwritten and Kannada notices, memory per child, spotting clashes between notices, Gmail reminders
- **Later:** WhatsApp channel, a school-side dashboard, and an open community around the project

## Team Baymax

- Sathvik G
- Vinod
- Srirama Rohit

Contributions and ideas are welcome. Open an issue or send a pull request.
