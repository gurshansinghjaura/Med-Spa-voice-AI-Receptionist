# Medspa Voice AI Receptionist

## What it does
An AI voice receptionist for a medical spa that answers inbound calls, books/reschedules/cancels appointments directly on Google Calendar, and answers service and pricing questions — without a human ever picking up the phone. It works with both Vapi.ai and ElevenLabs Conversational AI as the voice layer, with n8n as the backend brain.

## Demo
🎥 [Watch the 4-minute walkthrough](YOUR_LOOM_LINK_HERE)

## Architecture
A single webhook receives function/tool-call events from the voice platform (Vapi or ElevenLabs). The payload is normalized into one common shape regardless of which platform sent it, then a date/time parser converts whatever the caller said ("next Tuesday at 3", "15:00", "3pm") into a real calendar slot. From there the workflow routes to one of four tools — book, check availability, reschedule, cancel — all backed by Google Calendar. A single "Business Config" node centralizes services, pricing, hours, and policies so the agent's knowledge has one source of truth instead of being scattered across nodes. Every call is logged to Google Sheets, and important events (new bookings, cancellations) trigger a real-time Slack alert to the team.

Two supporting workflows round this out:
- **`medspa_knowledge_base_seeder.json`** — seeds a vector knowledge base with services, pricing, aftercare, and consultation-requirement info so the receptionist can answer policy questions without giving medical advice.
- **`daily_summary.json`** — runs every night at 8 PM IST, reads the day's call log, and posts a summary (bookings, reschedules, cancellations, questions answered, busiest service) to Slack so the owner has a nightly report without opening a spreadsheet.

![n8n canvas screenshot](YOUR_SCREENSHOT_HERE)

## Key technical decisions
- Centralized date/time parsing into a single node that accepts multiple formats and field names (used by both the booking and reschedule tools) to eliminate a class of silent parsing bugs where each tool call handled dates differently.
- Normalized the incoming payload shape immediately after the webhook, so the rest of the workflow doesn't care whether the request came from Vapi or ElevenLabs.
- Pulled all business-specific facts (services, prices, hours, cancellation policy) into one config node instead of hardcoding them into individual nodes, so updating a price or a working hour is a one-line change.
- Split the knowledge base and the daily reporting into their own standalone workflows rather than bolting them onto the receptionist, so each can be re-run or modified independently.

## Tech stack
n8n, Vapi.ai / ElevenLabs, Google Calendar API, Google Sheets API, Claude Sonnet 4.6, Google Gemini Embeddings, Slack API

## A real bug I fixed
Early on, the booking tool and the reschedule tool each parsed the caller's requested date/time independently, using slightly different field names and assumptions. A caller rescheduling to "3pm" would sometimes get silently dropped because the reschedule path didn't handle the same time formats the booking path did. I rebuilt this as one shared parsing function that normalizes combined datetime strings, separate date+time fields, and multiple time formats (24-hour, "3pm", "3:00 pm") into a single validated slot — used by every tool call, so there's now exactly one place that can get date parsing wrong instead of four.

## Setup
1. Import the workflow JSON files into n8n.
2. Reconnect credentials — Google Calendar, Google Sheets, Slack, and Google Gemini (or your embeddings provider of choice). The exported files use placeholder credential references (`YOUR_CREDENTIAL_ID`); n8n does not export real API keys.
3. Point your Vapi or ElevenLabs assistant's tool-call webhook at the `Webhook` node's URL.
4. Run `medspa_knowledge_base_seeder.json` once (manual trigger) to populate the knowledge base before going live.
