
# Scenario 6 – Instagram DM Watcher

```
Webhook → Set Variables → Instagram API Request → Prepare Message → AI Classification → Router → Telegram Alert → Airtable Log
```

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

```text
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
```

# Message Flow
```
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
```

---
### Captured data includes:

| Field | Description |
|--------|--------|
| entry.messaging.timestamp | Message timestamp |
| entry.messaging.sender.id | Instagram user ID |
| entry.messaging.message.mid | Unique message ID |
| entry.messaging.message.text | Message content |

Example Output

```json
[
    {
        "object": "instagram",
        "entry": [
            {
                "time": 1772631465118,
                "id": "17841448648228773",
                "messaging": [
                    {
                        "sender": {
                            "id": "1825206454632684"
                        },
                        "recipient": {
                            "id": "17841448648228773"
                        },
                        "timestamp": 1772631463969,
                        "message": {
                            "mid": "aWdfZAG1faXRlbToxOklHTWVzc2FnZAUlEOjE3ODQxNDQ4NjQ4MjI4NzczOjM0MDI4MjM2Njg0MTcxMDMwMTI0NDI1ODczMzMxNzg5NTkwODExNTozMjY5OTI3ODk1Mjg0OTI4MDEwNTk4NDc4NzA4ODQwODU3NgZDZD",
                            "text": "Do you have natakhtari lemonade?"
                        }
                    }
                ]
            }
        ]
    }
]

```

---
# Webhook Message Filter

After the Webhooks – Custom Webhook trigger, a filter is applied to ensure the scenario only processes events that contain actual message text.

This prevents the automation from triggering on other Instagram webhook events such as delivery confirmations, reactions, or typing indicators.

Filter Label: Instagram DM – Has text
Condition: 1.entry[].messaging[].message.text = Exists

Purpose
- Ensures the automation runs only for real Instagram messages
- Prevents unnecessary processing of non-message webhook events
- Reduces OpenAI and automation operations usage
- Keeps Airtable records limited to actual customer messages


---

# Module 2 – Variable Preparation

Tools – Set Multiple Variables

| Variable | Purpose |
|--------|--------|
| channel | instagram |
| message_id_external | Instagram message ID |
| sender_id | Instagram user ID |
| message_text | Raw message |
| timestamp | formatDate(parseDate(1.entry[].messaging[].timestamp / 1000; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai") |

Example Output

```json
[
    {
        "message_text": "Do you have natakhtari lemonade?",
        "sender_id": "1825206454632684",
        "message_id": "aWdfZAG1faXRlbToxOklHTWVzc2FnZAUlEOjE3ODQxNDQ4NjQ4MjI4NzczOjM0MDI4MjM2Njg0MTcxMDMwMTI0NDI1ODczMzMxNzg5NTkwODExNTozMjY5OTI3ODk1Mjg0OTI4MDEwNTk4NDc4NzA4ODQwODU3NgZDZD",
        "timestamp": "04-03-2026"
    }
]

```

---

# Module 3 – Instagram API Enrichment

HTTP – Make Request
```
GET https://graph.facebook.com/v18.0/{sender_id}
```

Returns:

| Field | Description |
|------|-------------|
| username | Instagram username |
| profile_pic | Profile picture |
| name | Display name |

```json
[
    {
        "data": {
            "username": "vohanjanyan",
            "id": "1825206454632684"
        },
        "headers": {
            "etag": "\"97309fbd2bee1ebffbb8da9c8aa31e411def1241\"",
            "content-type": "application/json",
            "vary": "Origin",
            "x-fb-aed": "764",
            "x-ad-api-version-warning": "The call has been auto-upgraded to v25.0 as v19.0 has been deprecated.",
            "cross-origin-resource-policy": "cross-origin",
            "x-app-usage": "{\"call_count\":1,\"total_cputime\":0,\"total_time\":3}",
            "access-control-allow-origin": "*",
            "facebook-api-version": "v25.0",
            "strict-transport-security": "max-age=15552000; preload",
            "pragma": "no-cache",
            "cache-control": "private, no-cache, no-store, must-revalidate",
            "expires": "Sat, 01 Jan 2000 00:00:00 GMT",
            "x-fb-request-id": "AwgjesH6rmQQp3hxcA__ZUD",
            "x-fb-trace-id": "Cb+uI0bUHiQ",
            "x-fb-rev": "1034469067",
            "x-fb-debug": "ge7VDrk9kaIhQIOK5DzGBNuhYYlyvyO8MHQEipM7FCV1573E4P1fC0MhCGI2aWFT4tVs0gpW+idScgE710OBtw==",
            "date": "Wed, 04 Mar 2026 13:37:46 GMT",
            "x-fb-connection-quality": "EXCELLENT; q=0.9, rtt=17, rtx=0, c=10, mss=1380, tbw=3458, tp=-1, tpl=-1, uplat=603, ullat=0",
            "alt-svc": "h3=\":443\"; ma=86400",
            "connection": "keep-alive",
            "content-length": "50"
        },
        "statusCode": 200
    }
]

```

---

# Module 4 – Sender Handle Resolution

Tools – Set Variable
```
sender_handle  
{{ifempty(username; sender_id)}}
```

Logic:

- If username exists → use username
- Otherwise → fallback to sender_id

---

# Module 5 – AI Message Classification

OpenAI Model: gpt-5-nano

Output format:
```
broad_category|issue_category|priority|confidence
```

Example:
```
support|product_question|normal|0.82
```

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
## Prompt

```
Classify the Instagram direct message below.

Return exactly 4 values separated by | in this order:

broad_category|issue_category|priority|confidence

broad_category (choose exactly one):

support
supplier_prospect
b2b_sales
marketing
spam
other

Definitions:

b2b_sales = A business buyer (restaurant, hotel, nursery, catering, retail shop, distributor, reseller) asking to purchase our products, request wholesale price list, MOQ, delivery terms, or partnership to BUY from us.

supplier_prospect = Producer, factory, exporter, or brand offering PHYSICAL GOODS supply to us, asking to distribute their products or send catalog / commercial offer.

support = Customer inquiry related to product availability, order status, delivery, payment, refund, complaint, store location, working hours, or any retail purchase.

marketing = Service providers, agencies, influencers, SEO, advertising offers, collaboration proposals, digital marketing, affiliate offers, traffic generation.

spam = Scam, crypto, suspicious links, fake courier warning, impersonation, irrelevant bulk messages.

other = Legitimate message that does not clearly fit above (e.g., job inquiry, greeting without context).


--- IMPORTANT CLASSIFICATION RULES ---


CONTEXT CONSISTENCY RULE

If the message appears to be part of an ongoing conversation, the issue_category should normally remain the same as the original customer inquiry.

Outbound replies from the store should inherit the issue_category of the customer message they are answering unless the topic clearly changes.

Do not change issue_category unless the customer explicitly introduces a new topic.

If the message contains only a greeting (e.g., "Hi", "Hello", "Здравствуйте") without any additional question or context → other | null | low | 0.80

If a greeting message contains an additional question about products, delivery, orders, or payment in the same message, ignore the greeting and classify based on the question.

- If asking about job / вакансии → other | null | low
- If asking about product availability → support | product_question
- If asking about order delay → support | order_status
- If complaining about delivery or wrong product → support | complaint
- If business wants to BUY from us → b2b_sales (priority MUST be high)
- If producer wants to SELL goods to us → supplier_prospect

Do not use "other" if the message clearly relates to a retail purchase or customer support.

If the message relates to:
• product availability
• delivery
• payment
• order timing
• order confirmation
• store information

it must be classified as support with the appropriate issue_category.


DELIVERY / SERVICE QUESTIONS

Questions about delivery availability, delivery areas, shipping coverage, or whether delivery works in a specific location must be classified as:

support | delivery_area

Examples:
"Do you deliver to Dubai Marina?"
"Do you deliver to Alain?"
"Is delivery available today?"
"Do you deliver to JVC?"

These questions are about service coverage, not order tracking.

issue_category (only if broad_category = support):

order_status
delivery_area
order_modification
payment
product_question
complaint
refund
cancellation
address_change
other

Use the same issue_category across the conversation whenever possible unless the topic clearly changes.


Clarification:

order_status also includes questions about delivery time, shipping estimate, when an order will arrive, delivery delays, or requests for an estimated delivery date.

If broad_category is NOT support, issue_category MUST be null.
Never return an issue_category value for non-support messages.

ORDER MODIFICATION QUESTIONS

Requests to change an existing order must be classified as:

support | order_modification

Examples:
"Can you change my order?"
"I want to remove one item"
"Please change delivery time"
"Add one more product to my order"


OUTBOUND MESSAGE CLASSIFICATION

If the message is written by the store (agent reply), use the same issue_category as the customer's message that triggered the reply unless the reply introduces a completely different topic.

Example:

Customer: "Do you deliver to Dubai Marina?"
Store: "Yes, delivery is available."

Both messages should use:
support | order_status


priority rules:

If broad_category = b2b_sales → high.
If broad_category = supplier_prospect → normal (high only if clearly urgent/large-scale).
If support AND issue_category in (complaint, refund) → high.
If support other → normal.
If marketing → low.
If spam → low.
If other → low.

confidence:
Return a number between 0.0 and 1.0.

Return only the pipe-separated line. No explanation.

Message:
{{1.entry[].messaging[].message.text}}
```
---

# Module 6 – Router

Messages are split into two processing routes.

After the OpenAI classification module, a Router is used to split the scenario into two processing paths based on the AI-detected message category.

The purpose of this routing step is to ensure that important messages requiring human attention trigger alerts, while lower-priority or irrelevant messages are simply logged for record keeping.

Why Routing Is Used

Not every Instagram DM requires immediate action. The AI classification determines whether a message is important for the business (customer support, sales, supplier inquiries) or non-critical (general chatter, marketing outreach, spam, etc.).


### Important DM
```
  first(split(7.Result; "|")) = support
OR
  first(split(7.Result; "|")) = b2b_sales
OR
  first(split(7.Result; "|")) = supplier_prospect
```

These trigger Telegram notifications.

### Unimportant DM

marketing  
spam  
other

These are logged only.

This route is configured as the fallback route, meaning it will process any messages that do not match the Important DM Filter conditions.

---

# Module 7 – Telegram Notification

For messages classified as important, the scenario sends a notification to the internal support Telegram group.
This allows the team to quickly see and respond to relevant Instagram inquiries without needing to monitor Instagram directly.
The Telegram message includes key information extracted and enriched earlier in the scenario.

Message Structure:


Message format:
```
📩 Instagram DM

👤 {{15.sender_handle}}
💬 {{6.message_text}}  
📅 {{6.timestamp}}
```
Parse Mode: HTML disabled (plain text recommended)

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
| timestamp_utc | {{6.timestamp}} |
| priority | get(split(Result,"|"),3) |
| confidence_score | get(split(Result,"|"),4) |
| resolution_status | unresolved |
| conversation_status | open |
| conversation_started | {{6.timestamp}} |
| conversation_hash | sender_id |


---

## Example Record

```json
[
    {
        "conversation_id": "1825206454632684",
        "message_id_external": "aWdfZAG1faXRlbToxOklHTWVzc2FnZAUlEOjE3ODQxNDQ4NjQ4MjI4NzczOjM0MDI4MjM2Njg0MTcxMDMwMTI0NDI1ODczMzMxNzg5NTkwODExNTozMjY5OTI3ODk1Mjg0OTI4MDEwNTk4NDc4NzA4ODQwODU3NgZDZD",
        "message_direction": "inbound",
        "message_source": "instagram dm",
        "message_text": "Do you have natakhtari lemonade?",
        "timestamp_utc": "2026-03-03T20:00:00.000Z",
        "resolution_status": "unresolved",
        "conversation_status": "open",
        "issue_category": "product_question",
        "confidence_score": 0.82,
        "channel": "instagram",
        "conversation_started_at": "2026-03-03T20:00:00.000Z",
        "conversation_hash": "1825206454632684",
        "Priority": "normal",
        "broad_category": "support",
        "id": "recbQ7svsXqxi4kgd",
        "createdTime": "2026-03-04T13:37:50.000Z"
    }
]
```

---
## HTTP — Send Classified Message to Conversation Engine

This module sends the classified Instagram message to the **Conversation Engine webhook**, which creates the Airtable conversation record and triggers the AI support workflow.

### Module
HTTP → **Make a request**

### Configuration

| Field | Value |
|------|------|
Authentication | No authentication |
Method | POST |
URL | https://hook.eu1.make.com/o08o3kqzpk8m0l2cbs1y2km2jjv8hsxh |
Content type | JSON |

---

### JSON Body

```json
{
"airtable_record_id": "{{22.id}}",
"channel": "instagram",
"message_text": "{{replace(22.message_text; newline; "\n")}}",
"broad_category": "{{22.broad_category}}",
"issue_category": "{{22.issue_category}}",
"priority": "{{22.Priority}}",
"message_direction": "{{22.message_direction}}"
}
