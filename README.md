# N8N-AI-Voice-Receptionist

An AI voice receptionist built with **n8n + Vapi**. It answers phone calls 24/7, books appointments in Google Calendar, calls customers to confirm tomorrow's bookings, and alerts your team on Slack when a call needs a human.

![Architecture](ai-voice-agent-diagram.svg)

> **Status:** early version. The Vapi webhook format was built from Vapi's public docs, but the workflow still needs testing with real calls in your own account. Run a few test calls before using it with customers.

## What it does

**Flow 1: Inbound call (live)**
Vapi handles the conversation (speech-to-text, LLM, text-to-speech). During the call it asks n8n to run four tools:

| Tool | What n8n does |
| --- | --- |
| `check_availability` | Reads Google Calendar and returns free slots for a date |
| `book_appointment` | Re-checks the slot, creates the calendar event, logs the booking in Google Sheets |
| `update_appointment` | Confirms or cancels an existing booking |
| `escalate_to_human` | Sends an instant Slack alert during the call when the caller asks for a human, is upset, reports an emergency, or the assistant failed twice. The assistant then takes a message and ends the call |

Every tool answers Vapi with `{ "results": [{ "name", "toolCallId", "result" }] }`, and Vapi speaks the result to the caller. If Vapi re-sends the same tool call, the booking is not duplicated.

**Flow 2: After every call**
Vapi sends an end-of-call report. An AI step extracts intent, outcome, sentiment and urgency, the call is saved to a `Calls` sheet (upsert on `call_id`, so no duplicates), and urgent, negative or unfinished calls trigger a Slack alert with a summary and the recording link.

**Flow 3: Reminder calls (Mon-Fri, 4 PM)**
n8n finds tomorrow's `booked` appointments that have not been reminded yet, and Vapi calls each customer with a reminder assistant. The customer confirms or cancels during the call, and the sheet and calendar are updated.

## Repository files

| File | Purpose |
| --- | --- |
| `ai-voice-agent-workflow.json` | The n8n workflow (56 nodes). Import with *Import from File* |
| `vapi-assistants.json` | Templates for the two Vapi assistants (inbound receptionist and outbound reminder) |
| `appointments-template.csv` | Headers for the `Appointments` tab |
| `calls-template.csv` | Headers for the `Calls` tab |
| `ai-voice-agent-diagram.svg` / `.png` | Architecture diagram |
| `error-handler-workflow.json` | Optional: logs failed nodes to a sheet and alerts Slack |

## Setup

1. **Google Sheet:** create a sheet with two tabs named `Appointments` and `Calls`, and paste the headers from the CSV templates.
2. **n8n credentials:**
   - Header Auth named *Vapi Server Secret* (header name `X-Vapi-Secret`, value = a long random secret)
   - Header Auth named *Vapi API Key* (header name `Authorization`, value `Bearer YOUR_VAPI_PRIVATE_KEY`)
   - Google Calendar, Google Sheets, OpenAI, Slack
3. **Import** `ai-voice-agent-workflow.json` into n8n.
4. **Edit the `Business Config` node:** business name, timezone, opening hours, open days, slot length.
5. **Replace the placeholders** in the workflow: `YOUR_CALENDAR_ID`, `YOUR_GOOGLE_SHEET_ID`, `YOUR_SLACK_CHANNEL_ID`, `YOUR_REMINDER_ASSISTANT_ID`, `YOUR_VAPI_PHONE_NUMBER_ID`.
6. **Create the two assistants in Vapi** from `vapi-assistants.json` (dashboard or API). Replace the server URL, secret and voice ID. The server URL is your n8n **production** webhook URL, ending in `/webhook/vapi-voice`.
7. **Activate** the workflow and call your Vapi phone number.
8. *(Optional)* Import `error-handler-workflow.json` and select it under **Settings, Error workflow** in the main workflow.

`vapi-assistants.json` contains templates with placeholders. Never commit real secrets or API keys.

## Sheet columns

**Appointments:** `booking_id, created_at, customer_name, phone, service, start, end, status, reminded, event_id, call_id`

**Calls:** `call_id, time, phone, duration_sec, ended_reason, intent, outcome, sentiment, urgent, caller_name, summary, follow_up, recording_url`

## Design decisions

- The assistant only states availability and confirmations that come from tool results, so it does not guess.
- Slots are re-checked right before booking to avoid double bookings.
- The webhook is protected with a secret header, so random requests are rejected.
- Human escalation is judgment-based (set in the assistant prompt, not a keyword list or confidence score): an instant Slack alert during the call, plus a second AI pass on the transcript after the call. Calls are also capped at 10 minutes.
- The assistant tells callers it is an AI virtual assistant.

## Known limitations

- Only the first tool call in each Vapi message is processed.
- One calendar, and slots of a fixed length set in `Business Config`.
- English voice and transcriber by default. Other languages need separate testing.
- No live transfer to a human: the assistant takes a message, the team gets a Slack alert, and someone calls back.
- Reminder calls need the customer's phone number in international format.

## Legal and privacy notes

Call recording, AI disclosure and outbound calling are regulated in many countries. Check the rules that apply to you (for example consent for recording and for automated calls) before going live. Only call customers who booked and gave you their number. This project is not designed for cold calling.

## Costs

Vapi, the voice provider, the LLM and your phone number are all billed by usage. Test with a few short calls first.

## Roadmap

- Live transfer to a human during business hours
- Multi-calendar and multi-location support
- Rescheduling tool
- SMS or email confirmation after booking

