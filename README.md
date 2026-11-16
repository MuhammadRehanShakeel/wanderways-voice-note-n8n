# wanderways-voice-note-n8n
n8n workflow that takes a voice-note text via POST webhook, classifies it into JSON intent with the Groq LLM API, logs it to Google Sheets, and routes failures to an error branch returning HTTP 500.
