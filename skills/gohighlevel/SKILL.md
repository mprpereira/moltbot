---
name: gohighlevel
description: Manage GoHighLevel (LeadConnector) CRM data via API v2 — contacts, pipelines/opportunities, conversations, calendars, and workflow enrollment — plus a chat menu pattern for driving GHL actions from a conversation.
homepage: https://marketplace.gohighlevel.com/docs
metadata: {"moltbot":{"emoji":"🟩","requires":{"bins":["jq"],"env":["GHL_API_KEY","GHL_LOCATION_ID"]},"primaryEnv":"GHL_API_KEY"}}
---

# GoHighLevel (GHL) Skill

Use the GoHighLevel API v2 (LeadConnector) to read/write CRM data for a sub-account
("location"): contacts, pipelines/opportunities, conversations, calendars, and
workflow enrollment. Also includes a pattern for driving a numbered chat menu
that maps user choices to GHL actions.

## Setup

1. In the GHL sub-account: **Settings → Private Integrations → Create new integration**.
2. Grant only the scopes you need (contacts, opportunities, conversations,
   calendars, workflows — read/write as applicable).
3. Copy the generated token and the sub-account's Location ID
   (**Settings → Business Info**, or the URL when viewing the location).
4. Store them:
   ```bash
   export GHL_API_KEY="pit-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
   export GHL_LOCATION_ID="your-location-id"
   ```
   Or in `moltbot.json`:
   ```json5
   {
     skills: {
       entries: {
         gohighlevel: {
           enabled: true,
           apiKey: "GHL_KEY_HERE",
           env: { GHL_LOCATION_ID: "your-location-id" }
         }
       }
     }
   }
   ```
5. One token/location per GHL sub-account. For multiple sub-accounts, create a
   separate Private Integration per location and switch `GHL_LOCATION_ID` /
   `GHL_API_KEY` per agent (see `agents.list[]` for per-agent env in
   [Multi-Agent Routing](/concepts/multi-agent)).

## API Basics

Base URL: `https://services.leadconnectorhq.com`

Every request needs:
```bash
curl -s "https://services.leadconnectorhq.com/<path>" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "Content-Type: application/json"
```

Most list/search endpoints require `locationId` as a query param or JSON body field.

## Contacts

**Search contacts:**
```bash
curl -s "https://services.leadconnectorhq.com/contacts/?locationId=$GHL_LOCATION_ID&query=jane" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.contacts[] | {id, firstName, lastName, phone, email}'
```

**Get one contact:**
```bash
curl -s "https://services.leadconnectorhq.com/contacts/{contactId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq
```

**Create a contact:**
```bash
curl -s -X POST "https://services.leadconnectorhq.com/contacts/" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"locationId":"'"$GHL_LOCATION_ID"'","firstName":"Jane","lastName":"Doe","phone":"+15551234567","email":"jane@example.com"}'
```

**Update a contact:**
```bash
curl -s -X PUT "https://services.leadconnectorhq.com/contacts/{contactId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"email":"new-email@example.com"}'
```

**Add / remove tags** (tags commonly drive GHL automations — see Workflows below):
```bash
curl -s -X POST "https://services.leadconnectorhq.com/contacts/{contactId}/tags" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"tags":["hot-lead"]}'

curl -s -X DELETE "https://services.leadconnectorhq.com/contacts/{contactId}/tags" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"tags":["hot-lead"]}'
```

## Pipelines & Opportunities

**List pipelines (and their stages):**
```bash
curl -s "https://services.leadconnectorhq.com/opportunities/pipelines?locationId=$GHL_LOCATION_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.pipelines[] | {id, name, stages}'
```

**Search opportunities:**
```bash
curl -s "https://services.leadconnectorhq.com/opportunities/search?location_id=$GHL_LOCATION_ID&pipeline_id={pipelineId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.opportunities[] | {id, name, pipelineStageId, status, monetaryValue}'
```

**Create an opportunity:**
```bash
curl -s -X POST "https://services.leadconnectorhq.com/opportunities/" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"locationId":"'"$GHL_LOCATION_ID"'","pipelineId":"{pipelineId}","pipelineStageId":"{stageId}","name":"Jane Doe - Website Lead","contactId":"{contactId}","status":"open"}'
```

**Move to a different stage:**
```bash
curl -s -X PUT "https://services.leadconnectorhq.com/opportunities/{opportunityId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"pipelineStageId":"{newStageId}"}'
```

## Conversations

**List/search conversations:**
```bash
curl -s "https://services.leadconnectorhq.com/conversations/search?locationId=$GHL_LOCATION_ID&contactId={contactId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.conversations[] | {id, lastMessageBody}'
```

**Get messages in a conversation:**
```bash
curl -s "https://services.leadconnectorhq.com/conversations/{conversationId}/messages" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.messages.messages[] | {direction, body, dateAdded}'
```

**Send an outbound message** (SMS/Email/etc.; `type` selects the channel):
```bash
curl -s -X POST "https://services.leadconnectorhq.com/conversations/messages" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"type":"SMS","contactId":"{contactId}","message":"Hey Jane, following up on your request!"}'
```

## Calendars & Appointments

**List calendars:**
```bash
curl -s "https://services.leadconnectorhq.com/calendars/?locationId=$GHL_LOCATION_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.calendars[] | {id, name}'
```

**Free slots for a calendar (epoch ms range):**
```bash
curl -s "https://services.leadconnectorhq.com/calendars/{calendarId}/free-slots?startDate={startMs}&endDate={endMs}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq
```

**Book an appointment:**
```bash
curl -s -X POST "https://services.leadconnectorhq.com/calendars/events/appointments" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"locationId":"'"$GHL_LOCATION_ID"'","calendarId":"{calendarId}","contactId":"{contactId}","startTime":"2026-08-01T15:00:00-03:00","endTime":"2026-08-01T15:30:00-03:00"}'
```

**Cancel an appointment:**
```bash
curl -s -X DELETE "https://services.leadconnectorhq.com/calendars/events/{eventId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28"
```

## Workflows

GHL's API does **not** let you author new workflow logic (triggers/actions/branches) —
the automation canvas itself is UI-only. What the API supports is **enrollment**:
listing existing workflows and adding/removing a contact from one. Treat "create a
flow" requests as "enroll the contact into the right existing workflow" (build the
actual automation once in the GHL UI, then drive it from here), or as tagging a
contact with a trigger tag (see Contacts above) if the workflow is tag-triggered.

**List workflows:**
```bash
curl -s "https://services.leadconnectorhq.com/workflows/?locationId=$GHL_LOCATION_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" | jq '.workflows[] | {id, name, status}'
```

**Enroll a contact into a workflow:**
```bash
curl -s -X POST "https://services.leadconnectorhq.com/contacts/{contactId}/workflow/{workflowId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28" -H "Content-Type: application/json" \
  -d '{"eventStartTime":"2026-07-27T00:00:00Z"}'
```

**Remove a contact from a workflow:**
```bash
curl -s -X DELETE "https://services.leadconnectorhq.com/contacts/{contactId}/workflow/{workflowId}" \
  -H "Authorization: Bearer $GHL_API_KEY" -H "Version: 2021-07-28"
```

## Chat Menu Pattern

For end-user conversations (WhatsApp/Telegram/Discord/etc.), present a plain
numbered menu — it renders consistently across every Moltbot channel without
needing channel-specific interactive UI. Map the reply to a GHL action:

```
Oi! Como posso ajudar hoje?
1) Agendar um horário
2) Ver status do meu atendimento
3) Falar com um humano
```

- **1 → Agendar**: look up the right calendar, call free-slots, offer 2-3
  options, then book with the appointment endpoint above. Resolve the contact
  first (search by phone/email from the channel identity; create if missing).
- **2 → Status**: search opportunities for the contact's `contactId` and read
  back `pipelineStageId`/`status` in plain language (map stage IDs to names via
  the pipelines endpoint — don't expose raw IDs to the user).
- **3 → Humano**: tag the contact (e.g. `precisa-humano`) so a tag-triggered GHL
  workflow/notification fires, and/or enroll into a "handoff" workflow if one
  exists; tell the user a human will follow up.
- Keep menus short (3-5 options), always offer an escape ("digite 'menu' para
  voltar"), and re-resolve the contact once per conversation rather than on
  every turn.

## Notes

- Rate limits: ~burst 100 requests / 10s and ~200,000/day per Private
  Integration token (location-scoped) — batch/cache reads where possible.
- The Private Integration token is location-scoped; it cannot see other
  sub-accounts. Never mix `GHL_LOCATION_ID` values across requests.
- Treat the token as a secret with full CRM read/write for that location —
  don't log it or echo it back in chat.
- IDs (`contactId`, `pipelineId`, `pipelineStageId`, `calendarId`,
  `workflowId`) are opaque strings; look them up via the list/search endpoints
  above rather than guessing.
