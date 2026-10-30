# Real Estate AI CRM: Omnichannel Lead Generation

Omnichannel lead capture and CRM automation for an Egyptian real estate company. Customers message on Telegram, WhatsApp, Instagram, or Messenger (text, voice notes, photos). An AI agent understands the request, searches available properties, logs the lead, books viewings, and escalates to a broker. A dashboard API feeds the CRM front end, and a daily job sends follow-up reminders.

Built for the Egyptian market (Egyptian Arabic replies, EGP budgets, local areas such as New Cairo, Sheikh Zayed, 6th of October, New Capital).

## Features

**Omnichannel intake**
- Telegram, WhatsApp (Evolution API, plus the Cloud API payload shape), Instagram DM, Messenger
- One normalizer turns every payload into a single `lead_info` object (source, user id, name, phone, media, text)
- Images are described by a vision model, voice notes are transcribed (Arabic), and both are merged into the customer's message
- Webhook protection: Meta verify challenge, Meta signature check, Evolution apikey check

**AI agent (8 tools)**
- `find_or_create_client`: upsert the customer in Postgres and return `client_id`
- `search_properties`: filter available listings by city, type, budget, bedrooms, return the 5 closest by price
- `log_inquiry`, `schedule_appointment`: append to Google Sheets
- `get_assigned_broker`, `notify_broker`: pick an active Property Consultant and alert them on WhatsApp
- `log_contract`, `log_payment`: record deals and payments, always followed by a broker notification
- Normalizes Arabic input (property types, areas, finishing, budgets like "3 مليون") and returns one structured JSON per turn
- Egyptian Arabic replies, never invents listings, prices, or availability. Postgres chat memory per channel and user (last 20 messages)

**Safety net**
- Validator parses the agent JSON, forces human approval when the reply touches contracts or payments, and falls back to a safe message plus broker escalation on a parse failure

**Dashboard API**
- `GET /webhook/realestate-dashboard-data` returns clients, properties, brokers (with closed deals and value), inquiries, appointments, contracts, payments, and chart data (revenue by month, lead funnel, leads by source)

**Follow-ups**
- Daily at 06:00: sends reminders for pending follow-ups on the client's channel and marks them completed

## Tech stack

| Layer | Tools |
|---|---|
| Automation | n8n (webhooks, Telegram trigger, AI Agent, tool nodes) |
| Database | Postgres (clients, properties, cities, property types, employees, follow-ups, chat memory) |
| Lead log | Google Sheets (inquiries, appointments, contracts, payments) |
| LLM | OpenRouter `openai/gpt-4o-mini` (agent, image analysis) |
| Voice | OpenAI transcription |
| Channels | Telegram Bot API, Evolution API v2, Meta Graph API (Instagram, Messenger) |

## Architecture

```
Telegram ──────────────────────────────┐
WhatsApp ─► apikey check ──────────────┤
Instagram ┐                            ├─► Normalize ─► Route media
Messenger ┴► signature check ──────────┘                │
                                          text ─────────┤
                                          image ─► download ─► vision ──┤
                                          voice ─► download ─► transcribe ┤
                                                                         ▼
                                                                   Build message
                                                                         │
                                                                         ▼
                                                  AI Agent ◄── Postgres memory (20)
                                                     │   ◄── 8 tools (Postgres, Sheets, WhatsApp)
                                                     ▼
                                                 Validate JSON
                                                     │
                                       needs approval? ─► broker alert
                                                     ▼
                                    Route by source ─► reply on the same channel

Dashboard ─► GET /realestate-dashboard-data ─► 7 sources ─► merge ─► JSON
Daily 06:00 ─► pending follow-ups ─► route by channel ─► send ─► mark completed
```

## Getting started

### 1. Import

Import the workflow JSON into n8n and re-link every credential.

### 2. Environment variables

| Variable | Description |
|---|---|
| `META_VERIFY_TOKEN` | Token for the Meta webhook verification challenge |
| `META_APP_SECRET` | App secret for the `X-Hub-Signature-256` check |
| `EVOLUTION_API_KEY` | Key expected on incoming Evolution webhooks |

n8n blocks `$env` access in Code nodes by default. Set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` or the checks skip silently (see Known issues).

### 3. Credentials

| Credential | Type | Used by |
|---|---|---|
| Postgres | `postgres` | Tools, memory, dashboard, follow-ups |
| Google Sheets | `googleSheetsOAuth2Api` | Inquiry, appointment, contract, payment logs |
| OpenRouter | `openRouterApi` | Agent, image analysis |
| OpenAI | `openAiApi` | Voice transcription |
| Telegram | `telegramApi` | Trigger, replies, alerts |
| Header Auth | `httpHeaderAuth` | Evolution API (name `apikey`, value the key, no `Bearer`), Meta tokens |

### 4. Database and sheets

Postgres tables: `clients` (unique on `source, external_user_id`), `properties`, `property_types`, `cities`, `employees` (`role = 'Property Consultant'`, `status = 'Active'`), `follow_ups` (`status`, `follow_up_date`, `channel`, `recipient_id`, `client_name`).
Sheet tabs: `log_inquiry1`, `schedule_appointment`, `log_contract`, `log_payment`.

### 5. Channels

1. Telegram: create a bot, add the token as a credential.
2. WhatsApp: Evolution instance `whatsapp_agent`, webhook `POST {N8N_URL}/webhook/whatsapp`.
3. Instagram and Messenger: Meta webhook to `/webhook/instagram` and `/webhook/messenger`, verify token = `META_VERIFY_TOKEN`.

### 6. Run

1. Activate the workflow.
2. Message the bot, for example "عايز شقة في التجمع ميزانيتي 3 مليون".
3. Open the dashboard endpoint and check the logs.

## Known issues

| Issue | Fix |
|---|---|
| **Secrets are hardcoded** in the Evolution, Instagram, and Messenger send nodes (and the follow-up copies) instead of using the credentials | Rotate all of them now and move them into credentials |
| **Agent can write contracts and payments from chat.** Approval is checked on the reply, after `log_contract` and `log_payment` already ran. A customer saying "I paid" creates a record | Remove both tools from the agent, or write to a `pending` table that a broker confirms |
| `log_payment` needs a `contract_id` "returned by log_contract", but a Sheets append returns no generated id, so the agent has to guess | Move contracts and payments to Postgres and return the new id |
| **Prompt tool names do not match node names**: prompt says `search_properties1`, `log_inquiry1`, `schedule_appointment1`, `find_or_create_client`, the nodes are `...2` | Rename nodes or update the prompt, the model sees the node name |
| `Force Notify Broker` sends to the customer's Telegram chat (`Telegram Trigger2`), leaking the internal alert, and errors silently for other channels | Send to the broker's chat or WhatsApp from `get_assigned_broker` |
| Agent `maxTokens` is 300 but it must return a full JSON object with a reply, so output truncates and everything escalates as a parse failure | Raise to 1500 or more |
| Verification fails open: missing `$env` values skip the Evolution key check, accept any Meta verify token, and skip the signature check | Fail closed when the secret is missing |
| Meta signature is computed over `JSON.stringify(body)`, not the raw body, so valid Arabic messages will not match | Use the raw request body |
| Two webhooks use path `whatsapp` (`whatsapp2`, `whatsapp3`), and `whatsapp3` is not connected | Delete `whatsapp3` |
| Follow-up Telegram node wraps the chat id in quotes and sends a JSON string as the message text, so customers receive raw JSON | Use a plain chat id and plain text |
| Nothing in this workflow creates `follow_ups` rows | Add a tool or a step that inserts them |
| Messenger and Instagram follow-ups go out as `RESPONSE`, which Meta rejects outside the 24 h window | Use an allowed message tag or a template flow |
| Follow-up status update interpolates `{{$json.id}}` into SQL | Use a query parameter |
| `get_assigned_broker` always returns the lowest id, no round robin | Order by last assignment or workload |
| Agent never sees the customer's name or phone from `lead_info`, so it asks for data the channel already has | Pass them in the prompt |
| Property types in the prompt (`TOWNHOUSE`, `TWIN_HOUSE`, `CHALET`, `STUDIO`, `OFFICE`) differ from the search tool's list (`DUPLEX`, ...) | Use one list |
| Unsupported media (stickers, documents) falls out of the switch silently | Add a fallback reply |
| `clients.status` does not exist, so the funnel chart is always `NEW`. Broker `appts` is always 0 | Add the column, store `employee_id` on appointments |

## Security notes

- **Rotate the exposed tokens.** The Instagram and Messenger page tokens and the Evolution key sit in plain text in the exported JSON. Anyone who has that file can send as your pages. Revoke and reissue them.
- **Protect the dashboard endpoint.** `/realestate-dashboard-data` is public and returns client names, phones, contracts, and payments. Require a header secret or put it behind your app's auth.
- Customers can give a national ID through chat, and the agent writes it to the database. Collect only what you need and restrict access to that column.
- Keep contracts and payments human-confirmed. The agent should suggest, a broker should record.
- Keep all keys in n8n credentials. Never inline them in nodes or prompts.
- Do not commit exported workflow JSON with credential IDs, sheet IDs, or tokens to a public repo.

## Project structure

```
workflow.json     exported n8n workflow
README.md
```

## Author

**Faris Ataa Lashen**, AI Automation Engineer (n8n, AI Agents, RAG, LLMs)
Portfolio: https://farislashen7-hub.github.io/portfolio · LinkedIn: https://linkedin.com/in/farislashen
