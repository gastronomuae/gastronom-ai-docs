
# Scenario 6 – Instagram DM Watcher

Webhook → Set Variables → Instagram API Request → Prepare Message → AI Classification → Router → Telegram Alert → Airtable Log

---

## Overview

This scenario processes incoming Instagram Direct Messages (DMs) received via a webhook, enriches the data using the Instagram Graph API, classifies the message using AI, and logs the interaction in Airtable while notifying the support team through Telegram when necessary.

The structure mirrors the Email Automation system to ensure consistent conversation tracking and CRM logging.

---

## Objective

The automation performs the following:

- Capture every inbound Instagram DM via webhook
- Enrich sender data using Instagram Graph API
- Classify message intent using AI
- Log messages in Airtable
- Notify the team in Telegram when action is required

This ensures Instagram inquiries are tracked consistently alongside other support channels such as email and WhatsApp.

---

# Module 1 – Webhooks – Custom Webhook

The scenario starts when Instagram sends a webhook event for a new Direct Message.

Webhook name:

ig_dm_inbound

### Webhook Configuration

Callback URL:
https://hook.eu1.make.com/7kx7f6dolf6eykvkxjqlawvga4xcgepu

Verify Token:
gastronom123

Whenever a new Instagram DM is received, Meta sends an event to this webhook which triggers the automation.

---

## Subscribed Webhook Events

messages  
messaging_postbacks

---

## Connected Facebook Page

Page Name: gastronom.uae  
Page ID: 107318278322469

---

# Message Flow

Instagram Direct Message  
↓  
Meta Webhook Event  
↓  
Make Automation Scenario  
↓  
AI Message Classification  
↓  
Router (Important / Non‑Important)  
↓  
Telegram Alert + Airtable Logging

---

# Module 2 – Variable Preparation

Tools – Set Multiple Variables

| Variable | Purpose |
|--------|--------|
| channel | instagram |
| message_id_external | Instagram message ID |
| sender_id | Instagram user ID |
| message_text | Raw message |
| timestamp | formatDate(parseDate(timestamp/1000,"X"),"DD-MM-YYYY","Asia/Dubai") |

---

# Module 3 – Instagram API Enrichment

HTTP – Make Request

GET https://graph.facebook.com/v18.0/{sender_id}

Returns:

| Field | Description |
|------|-------------|
| username | Instagram username |
| profile_pic | Profile picture |
| name | Display name |

---

# Module 4 – Sender Handle Resolution

Tools – Set Variable

sender_handle  
{{ifempty(username; sender_id)}}

Logic:

- If username exists → use username
- Otherwise → fallback to sender_id

---

# Module 5 – AI Message Classification

OpenAI Model: gpt-5-nano

Output format:

broad_category|issue_category|priority|confidence

Example:

support|product_question|normal|0.82

Broad categories:

support  
supplier_prospect  
b2b_sales  
marketing  
spam  
other

Support issue categories:

order_status  
payment  
product_question  
complaint  
refund  
cancellation  
address_change  
other

---

# Module 6 – Router

Messages are split into two processing routes.

### Important DM

support  
b2b_sales  
supplier_prospect

These trigger Telegram notifications.

### Unimportant DM

marketing  
spam  
other

These are logged only.

---

# Module 7 – Telegram Notification

Message format:

📩 Instagram DM

👤 {{sender_handle}}  
💬 {{message_text}}  
📅 {{timestamp}}

---

# Module 8 – Airtable Logging

Base: AI Staff – Conversation Engine  
Table: conversation_log

Purpose:

- Maintain centralized conversation history
- Enable analytics and AI training

---

## Airtable Field Mapping – INBOUND

| Field | Mapping |
|------|--------|
| conversation_id | sender_id |
| message_id_external | message.mid |
| message_direction | inbound |
| message_source | instagram |
| channel | instagram |
| wa_number | empty |
| message_text | message_text |
| broad_category | first(split(Result,"|")) |
| issue_category | get(split(Result,"|"),2) |
| priority | get(split(Result,"|"),3) |
| confidence_score | get(split(Result,"|"),4) |
| resolution_status | unresolved |
| conversation_status | open |
| conversation_hash | sender_id |
| timestamp_utc | formatDate(timestamp,"YYYY-MM-DDTHH:mm:ssZ") |

---

## Example Record

{
  "conversation_id": "1825206454632684",
  "message_direction": "inbound",
  "message_source": "instagram",
  "message_text": "Do you have natakhtari lemonade?",
  "issue_category": "product_question",
  "priority": "normal",
  "confidence_score": 0.82
}
