# Voice Note Intent Classifier (n8n)

A minimal n8n workflow that receives a POST request (a simulated voice-note upload),
sends the text to a free LLM (Groq), classifies it into a JSON intent, writes the
result to a Google Sheet, and routes any failure to an error branch that logs the
problem and returns HTTP 500.

Workflow file: `voice-note-intent-workflow.json`

---

## Flow

```
Webhook (POST /voice-note)
        |
Classify with Groq (HTTP Request)  --error-->  Build Error Row
        |                                              |
Parse Intent (Code)                --error-->   Log to Errors Sheet
        |                                              |
Log to Results Sheet (Google Sheets) --error-->  Respond 500
        |
Respond 200
```

Every node on the success path has its error output wired to the same error branch,
so a bad API key, invalid LLM JSON, or a Sheets failure all end in one place.

---

## 1. Run n8n locally

Node.js 18+ required.

```bash
npx n8n
```

Open http://localhost:5678

## 2. Import the workflow

In n8n: **Workflows → Import from File →** choose `voice-note-intent-workflow.json`.

## 3. Add credentials

**Groq (free tier)**

1. Get a key from https://console.groq.com/keys
2. In n8n: **Credentials → New → Header Auth**
   - Name: `Authorization`
   - Value: `Bearer YOUR_GROQ_KEY`
3. Open the **Classify with Groq** node and select this credential.

**Google Sheets**

1. In n8n: **Credentials → New → Google Sheets OAuth2** and connect your account.
2. Open **Log to Results Sheet** and **Log to Errors Sheet** and select it.

## 4. Prepare the Google Sheet

Create one spreadsheet with two tabs:

| Tab | Header row |
|---|---|
| `Results` | `timestamp` , `text` , `intent` |
| `Errors` | `timestamp` , `text` , `error` |

Copy the spreadsheet ID from its URL
(`https://docs.google.com/spreadsheets/d/<THIS_PART>/edit`) and paste it into both
Sheets nodes in place of `YOUR_GOOGLE_SHEET_ID`.

## 5. Activate and test

Click **Execute workflow** (test URL) or **Activate** (production URL), then:

```bash
curl -X POST http://localhost:5678/webhook/voice-note \
  -H "Content-Type: application/json" \
  -d '{"text":"I need a flight to Paris next Friday"}'
```

Success response:

```json
{
  "status": "ok",
  "intent": { "intent": "flight_query", "details": { "destination": "Paris" } }
}
```

Row added to **Results**:

```
2026-11-16T10:15:00.000Z | I need a flight to Paris next Friday | {"intent":"flight_query","details":{"destination":"Paris"}}
```

While testing use `/webhook-test/voice-note` instead of `/webhook/voice-note`.

## 6. Test the error branch

Break the Groq credential on purpose (change the key to `Bearer wrong`) and send the
same curl request. You should get HTTP 500:

```json
{ "status": "error", "error": "401 - Invalid API Key" }
```

and a new row in the **Errors** tab with the timestamp, the original text, and the
error message.

---

## The prompt

The system prompt tells the model to reply with JSON only, in this shape:

```json
{ "intent": "...", "details": {} }
```

`temperature` is `0` and Groq's `response_format: json_object` is enabled, so the
output stays parseable. The **Parse Intent** node still re-parses it and throws if
the JSON is broken or has no `intent` field — that throw is what feeds the error
branch.

## Notes / not included

- No real audio transcription; the webhook takes a `text` field, as the brief asks.
- No auth on the webhook itself (local use only).
- Credentials are stored in n8n, never in this repo.
