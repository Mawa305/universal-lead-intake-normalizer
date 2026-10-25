# Universal Lead Intake Normalizer

A webhook that accepts leads from any source, such as a website form, a landing page, or a third-party tool. Field names differ between sources (`name`, `full_name`, `customer`, and so on), so the workflow maps them to one standard shape. It cleans the phone number and email, checks that the lead is usable, skips duplicates, and saves it to Supabase. Invalid leads are logged separately, and save failures are sent to Telegram so they can be added by hand.

## Architecture

```
Receive Lead (POST /lead-intake)
   → Normalize Lead (map fields, clean phone and email, add received_at)
   → Is Valid Lead? (has name, and a phone of 10+ digits or an email)
        ├── Yes → Check Duplicate (Supabase: leads, by phone)
        │            → Is New Lead?
        │                 ├── Yes → Save Lead (Supabase: leads)
        │                 │            └── error → Alert Lead Save Failed (Telegram)
        │                 └── No  → (duplicate, not saved)
        └── No  → Log Invalid Lead (Supabase: invalid_leads)
```

## Features

* Accepts field names from different sources (`name` or `full_name`, `phone` or `mobile`, `email` or `email_address`, and so on)
* Cleans the phone number to digits (and a leading `+`), so `(555) 123-4567` and `555.123.4567` match
* Lowercases email addresses
* Requires a name and either a 10-digit phone or an email before saving
* Skips a lead when a lead with the same phone already exists
* Logs invalid leads with the raw request body, so nothing is lost
* Sends a Telegram alert with the lead details when saving fails

## Tech Stack

* n8n 2.x (workflow automation, webhooks)
* Supabase (leads and invalid_leads tables)
* Telegram Bot API (save failure alerts)

## Setup

1. Create two Supabase tables:

   `leads`:
   ```
   id | name | phone | email | service | message | source | received_at
   ```

   `invalid_leads`:
   ```
   id | raw | source
   ```

   The `received_at` column name must match exactly. If your table already has `recieved_at` from an earlier version, rename it to `received_at`.

2. Import `universal_lead_intake_normalizer.json` into n8n.
3. Add credentials: Supabase and Telegram. Assign them to the matching nodes.
4. Replace `YOUR_TELEGRAM_CHAT_ID` in the Alert Lead Save Failed node with your chat ID.
5. Activate the workflow. Copy the production webhook URL from the Receive Lead node.
6. Send a test lead:

```
curl -X POST https://YOUR_N8N_DOMAIN/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d '{"full_name":"  Ali Khan ","mobile":"(555) 123-4567","email_address":"ali.test@example.com","service_needed":"AC repair","message":"AC not cooling","source":"website"}'
```

Expected result: one row in `leads` with phone `5551234567`, email `ali.test@example.com`, and source `website`.

## Field Mapping

| Standard field | Accepted source names (first match wins) |
|---|---|
| name | `name`, `full_name`, `customer`, `customer_name` |
| phone | `phone`, `mobile`, `phone_number`, `contact` |
| email | `email`, `email_address` |
| service | `service`, `service_needed`, `issue` |
| message | `message`, `notes`, `details` |
| source | `source`, `channel` (defaults to `unknown`) |

## Known Limits

* Duplicates are checked by phone only. Leads with no phone are saved even if the same email was already received. Add an email check if you need that.
* A duplicate lead is not saved and not logged. Add a branch to the "No" output of Is New Lead if you want to track duplicates.

## Project Notes

* Built as part of my AI Automation portfolio.
* Keeping field mapping in one Code node means a new lead source only needs a new field name added to `pick(...)`, not a new workflow.
* No AI model is used. Cleaning and matching are done with plain code, so the workflow is fast and cheap to run.# universal-lead-intake-normalizer
n8n webhook that accepts leads from any source, maps differing field names to one format, cleans phone and email, skips duplicates, and saves to Supabase. Invalid leads are logged and save failures alert on Telegram.
