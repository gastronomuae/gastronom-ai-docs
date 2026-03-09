
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

## Meta Developer Application

A Meta Developer application was created to enable access to the Instagram Messaging API.

Application details:
- App Name: direct_messages_in-IG
- Instagram App ID: 841816832207699

The application provides the API access required to:
- receive Instagram Direct Messages
- retrieve sender information
- process messaging events via webhook
- enable automation workflows.

Enabled Meta Use Cases
The following Meta platform use cases were enabled for the application:
- Manage messaging & content on Instagram
- Engage with customers on Messenger from Meta

Test use cases
- These use cases allow the application to receive and process messaging events originating from Instagram conversations.

Instagram API Configuration
- The application was configured using API Setup with Instagram Login.
- This configuration connects the Instagram Business account to the Meta Developer application and enables the application to interact with Instagram messaging services.

Permissions Granted
The following permissions were enabled for the application:
- instagram_business_basic
- instagram_manage_comments
- instagram_business_manage_messages
  
These permissions allow the system to:
- access the Instagram Business account
- read incoming Direct Messages
- retrieve sender information
- process messaging events from conversations.

Webhook Configuration

A webhook endpoint was configured to deliver Instagram messaging events to the automation system.

Webhook settings:
- Callback URL: https://hook.eu1.make.com/7kx7f6dolf6eykvkxjqlawvga4xcgepu 
- Verify Token: gastronom123

The webhook connects the Meta platform with the Make.com Instagram DM automation scenario.

Whenever a new Instagram Direct Message is received, Meta sends an event to this webhook endpoint which triggers the automation workflow.

Subscribed Webhook Events

The application was subscribed to the following webhook events:
- messages
- messaging_postbacks

These events allow the system to detect when:
- a new Instagram Direct Message is received
- messaging interactions occur within a conversation.
  
Connected Facebook Page
- The Meta application was connected to the Facebook Page associated with the company Instagram account.
- Page Name: gastronom.uae
- Page ID: 107318278322469

This connection allows the webhook to receive messaging events originating from the Instagram Business account.

Access Token Generation
A Page access token was generated for the connected page.
The token is used to:
- authorize API requests
- enable webhook subscriptions
- allow the application to interact with messaging endpoints.

App Roles
The following roles were assigned in the Meta Developer application:
- Administrator: Vahagn Ohanjanyan
- Instagram Tester: vohanjanyan

The Instagram tester role allows the messaging integration to be tested while the application remains in development mode.

Integration Result
With this configuration, all Instagram Direct Messages received by the connected Instagram Business account are automatically delivered to the automation system.

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
### Captured data includes:

| Field | Description |
|--------|--------|
| entry.messaging.timestamp | Message timestamp |
| entry.messaging.sender.id | Instagram user ID |
| entry.messaging.message.mid | Unique message ID |
| entry.messaging.message.text | Message content |

---


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
