# Scenario 5 — Email Watcher (AI Classification)

## Objective

Monitor the support inbox **ai@gastronom.ae**, prevent duplicate processing, classify emails using AI, route important messages to Telegram, and log all email activity into Airtable for an AI‑ready dataset.

This scenario builds the foundation for **AI‑powered customer support intelligence**.

---

# Architecture Overview

```
Email – Watch Emails
↓
Airtable – Search Records (Message-ID Deduplication)
↓
Router – Duplicate Detection
  → Branch 1: Already Processed → STOP  
  → Branch 2: New Email
↓
Router – Direction Detection
  → OUTBOUND → Airtable Log → STOP  
  → INBOUND
↓
System Email Filter
↓
OpenAI Classification
↓
Router – Importance Filter
  → Telegram Notification  
  → Airtable Create Record
```

---

# Module 1 — Email: Watch Emails

Purpose:

Trigger when a new email arrives in the **ai@gastronom.ae** mailbox.

Captured fields:

- Subject
- Sender email address
- Text content
- Headers → message-id
- Date

Important fields used in the scenario:

- Sender Email address
- Subject
- Text content
- Headers.basic.message-id
- Date

---

# Module 2 — Airtable: Search Records (Message-ID Deduplication)

Purpose:

Prevent duplicate processing caused by:

• IMAP replays  
• scenario restarts  
• mailbox re-indexing  

Configuration:

Base: **AI Staff – Conversation Engine**  
Table: **conversation_log**

Formula:
```
{message_id_external} = "{{headers.message-id}}"
```
Limit: 1

If the Message-ID already exists, the scenario stops execution.

---

# Module 3 — Router: Duplicate Detection

Branch 1 — Already Processed

Condition
```
Total bundles > 0
```
Action

STOP

Branch 2 — New Email

Condition
```
Total bundles = 0
```
Action

Continue scenario

---

# Module 4 — Router: Direction Detection

Purpose:

Separate **staff replies** from **customer emails** before calling OpenAI.

## Branch A — OUTBOUND (Staff)

Condition:

Sender email equals:

hello@gastronom.ae  
ai@gastronom.ae  

Processing:

→ Airtable Create Record  
→ STOP

Reason:

- Prevent unnecessary OpenAI usage
- Prevent staff messages entering AI dataset  

---
### Airtable Logging — OUTBOUND Messages

When the email is sent by staff (`hello@gastronom.ae` or `ai@gastronom.ae`), the message is written to Airtable without AI classification.

Purpose:

• Preserve full conversation history  
• Track staff responses  
• Maintain thread linkage for analytics  

Configuration:

Airtable module  
Base: **AI Staff – Conversation Engine**  
Table: **conversation_log**

Field mapping:

| Field | Mapping |
|------|--------|
| **conversation_id** | `{{if(length(2.references[]) > 0; 2.references[]; 2.headers.`message-id`[])}}` |
| **wa_number** | empty |
| **order_id** | empty |
| **message_id_external** | `{{2.headers.`message-id`[]}}` |
| **message_direction** | outbound |
| **message_source** | email |
| **message_text** | `{{first(split(2.text; "\nOn "))}}` |
| **timestamp_utc** | `{{formatDate(parseDate(2.date / 1000; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}` |
| **channel** | email |
| **conversation_started** | empty |
| **conversation_hash** | `{{lower(2.from.address)}}` |
| **label_source** | human_labeled |
| **agent_name** | hello@gastronom.ae |

Notes:

• OpenAI classification is skipped for outbound messages  
• These records allow tracking response times and conversation lifecycle  
• Thread linkage relies on the **References header** when available

---

## Branch B — INBOUND (Customer)

Fallback route for all other emails.

Processing:

OpenAI Classification → Router 2

---

# Module 5 — System / Bounce Email Filter

Purpose

Prevent obvious automated system emails from reaching the OpenAI
classification module.

This reduces AI token usage and prevents bounce/system notifications
from polluting the support dataset.

Condition rules

The email is considered system-generated if ANY of the following
conditions match:

Sender email contains:

• stripe
• instashop
• t.shopifymail.com
• postmaster

OR

Headers.basic.return-path contains:

• bounce
• noreply
• no-reply
• mailer-daemon

OR

Headers.basic.content-type contains:

• report

---

# Module 6 — OpenAI Classification

Model

gpt-5-nano

## Prompt

```
Purpose

Classify the email below.

Return exactly 6 values separated by | in this order:

broad_category|issue_category|priority|confidence|customer_email|supplier_region

broad_category (choose exactly one):

support
supplier_prospect
b2b_sales
marketing
spam
other

Definitions (IMPORTANT):

IMPORTANT DISTINCTION:

b2b_sales applies ONLY when the sender wants to BUY physical products from us.

If the sender offers to generate customers, traffic, leads, or orders for us in exchange for commission, affiliate payment, marketing fee, promotion, SEO, advertising, or partnership to promote our store, this is NOT b2b_sales.

Those emails MUST be classified as marketing.

b2b_sales = B2B BUYER inquiry to us (hotel, catering, restaurant, retail store, distributor, GCC buyer asking to purchase our products, request price list, MOQ, delivery terms, partnership to BUY from us).

supplier_prospect = PRODUCER / factory / exporter offering PHYSICAL GOODS supply to us, asking to cooperate, distribute their brand, send catalog, commercial offer, or supply products. This category applies only to manufacturers or exporters of tangible goods.

support = Customer-related issue about an existing order or retail purchase (order status, payment issue, refund, complaint, delivery issue, product question).

marketing = Service providers, agencies, freelancers, affiliates, commission-based offers, SEO services, advertising proposals, traffic generation offers, listing platforms, digital growth proposals, performance marketing, or any service unrelated to purchasing physical products from us.

spam = Obvious spam, phishing attempts, suspicious payment warnings, fake invoices, cryptocurrency requests, password reset scams, impersonation of banks or couriers, irrelevant bulk emails.

spam also includes:

• mailing list messages
• Google Groups notifications
• helpdesk ticket updates from external systems
• automated discussion threads or ticket notifications
• system generated messages where the sender is not directly contacting Gastronom
• promotional newsletters unrelated to Gastronom business


If the email contains headers or indicators such as:

Precedence: list
Mailing-list
List-ID
List-Unsubscribe
Google Groups
Ticket system notifications

the email is likely an automated mailing list message and should be classified as spam unless the content clearly relates to Gastronom business.

other = Legitimate email that does not clearly fall into the above categories.

LANGUAGE NOTE:

Emails may be written in English, Russian, Arabic, French, or other languages.

Classification must be based on the meaning of the message, not the language used.

For example, emails offering to generate customers, traffic, sales, or orders in exchange for commission, affiliate payment, or promotion must always be classified as marketing, regardless of language.

issue_category (only if broad_category = support):

order_status
payment
product_question
complaint
refund
cancellation
address_change
other

Clarification:

order_status also includes questions about delivery time, shipping estimate, when an order will arrive, delivery delays, or requests for an estimated delivery date.

If not support, return null.

priority (choose exactly one):

high
normal
low

Priority rules:

If broad_category = b2b_sales → priority MUST be high.

If broad_category = supplier_prospect → priority = normal.
If email clearly mentions urgent shipment, UAE stock availability, exclusive brands, or large-scale proposal → priority = high.

If broad_category = support AND issue_category in (complaint, refund) → priority = high.

If broad_category = support AND other issue types → priority = normal.

If broad_category = marketing → priority = low.

If broad_category = spam → priority = low.

If broad_category = other → priority = normal.

confidence:

Return a number between 0.0 and 1.0.

customer_email:

Extract the customer's email address if it appears inside the message body.

Some emails may come from Shopify contact forms where the sender appears as:

[mailer@shopify.com](mailto:mailer@shopify.com)

In those cases the real customer email is included inside the message body in the format:

Email:
[customer@email.com](mailto:customer@email.com)

Return the extracted email address.

If no customer email appears in the message body, return:
null

supplier_region:

If broad_category = supplier_prospect, determine whether the supplier belongs to the priority region.

Priority region includes:

• Suppliers located in Russia, Armenia, Georgia, Uzbekistan, Kazakhstan, Belarus
• OR suppliers located in the UAE offering products specifically targeted at Russian/CIS customers (for example pelmeni, vareniki, manty, Eastern flatbread, or other CIS-style foods)

Return:
priority_region

If the supplier is clearly located outside this region and not related to CIS products (for example Vietnam, India, China exporters), return:
other_region

If the supplier country cannot be determined, return:
unknown

If broad_category is not supplier_prospect, return:
null

The country may appear in different grammatical forms or languages (e.g., Россия/России, Армения/Армении, Georgia/Грузии, etc.).

Return exactly one of:
priority_region
other_region
unknown

If broad_category is not supplier_prospect, return:
null

Return only the pipe-separated line. No explanation.


Subject:
{{2.subject}}

From:
{{2.from.address}}

Body:
{{first(split(2.text; "\nOn "))}}
```

---

# Module 7 — Router: Support Importance Filter

Purpose

Send only important messages to Telegram.

```
Condition:
  first(split(7.Result; "|"))
  Equal to
  support
OR
  {{first(split(7.result; "|"))}}
  Equal to
  b2b_sales
OR 
  {{first(split(7.result; "|"))}}
  Equal to
  supplier_prospect

AND
  {{get(split(7.result; "|"); 6)}}
  Equal to
  priority_region
```
---

# Module 8 — Telegram Notification

Purpose

Alert the support team of important emails.

Message format
```
📧 CUSTOMER EMAIL

From: {{2.Sender: Email address}}
Subject: {{2.Subject}}

Message:
{{substring(trim(toString(ifempty(2.text; ))); 0; 250)}}
```
---

# Module 9 — Airtable: Create Record

Base

AI Staff – Conversation Engine

Table

conversation_log

Purpose

Create a structured dataset for analytics and AI training.

---

## Field Mapping — INBOUND

| Field | Mapping |
|------|--------|
| **conversation_id** | `{{if(length(2.references[]) > 0; 2.references[]; 2.headers.`message-id`[])}}` |
| **wa_number** | `empty` |
| **order_id** | `empty` |
| **message_id_external** | `{{2.headers.`message-id`[]}}` |
| **message_direction** | `inbound` |
| **message_source** | `email` |
| **message_text** | `{{if(length(first(split(2.text; "\nOn "))) > 250; substring(first(split(2.text; "\nOn ")); 0; 250) + "..."; first(split(2.text; "\nOn ")) )}}` |
| **broad_category** | `first(split(7.Result; "|"))` |
| **issue_category** | `{{if(trim(get(split(7.result; "|"); 2)) = "null"; ; trim(get(split(7.result; "|"); 2)))}}` |
| **timestamp_utc** | `{{formatDate(parseDate(2.date / 1000; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}` |
| **resolution_status** | `unresolved` |
| **conversation_status** | `open` |
| **escalation_flag** | `{{ifempty(get(split(7.result; "|"); 7); false)}}` |
| **priority** | `trim(get(split(7.Result; "|"); 3))` |
| **confidence_score** | `trim(get(split(7.Result; "|"); 4))` |
| **channel** | `email` |
| **conversation_started** | `{{formatDate(parseDate(2.date / 1000; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}` |
| **conversation_hash** | `if(get(split(7.result;|); 5) != null; lower(get(split(7.result;|); 5)); lower(2.from.address))` |

### conversation_id

Used to group inbound and outbound messages belonging to the same conversation thread.

For emails this is derived from the `message-id` header.

While each email technically has its own `message-id`, the system uses this field to associate replies within the same exchange.

### message_id_external

Unique identifier of the specific email message event received from the mailbox.

### conversation_hash

Used to identify the customer across messages. Normally uses the sender email address.

However when emails are sent via the Shopify contact form the header sender is:


---

## Field Mapping — OUTBOUND

| Field | Mapping |
|------|--------|
| **conversation_id** | `2.Headers → basic → message-id` |
| **message_id_external** | `2.Headers → basic → message-id` |
| **message_direction** | `outbound` |
| **message_source** | `email` |
| **channel** | `email` |
| **message_text** | `{{first(split(2.text; "\nOn "))}}` |
| **timestamp_utc** | `{{formatDate(parseDate(now; "X"); "YYYY-MM-DD HH:mm:ss"; "UTC")}}` |
| **resolution_status** | `unresolved` |
| **conversation_status** | `open` |
| **conversation_started** | `{{formatDate(parseDate(2.date / 1000; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}` |

No AI classification fields used.

---

# What This Scenario Achieves

• Deterministic inbound vs outbound detection  
• Message-ID deduplication  
• AI classification of emails  
• Telegram escalation of important messages  
• Structured Airtable dataset  
• Foundation for AI-powered support analytics  

---

# Current Limitations

• No response time calculation yet  
• No automated resolution logic  
• No B2B routing automation yet  
• No sentiment scoring

---

# Technical Notes

• Email thread trimming relies on split by "\nOn "  
• conversation_hash equals sender email unless extracted from body  
• Thread linking uses Message-ID  
• Priority stored for SLA design


---

# Change Log

## 2026-03-10

### Email system filter improvements

Two updates were introduced to improve dataset quality and reduce unnecessary OpenAI processing.

**1. Filter renamed**

Previous name:
System generated emails filter

New name:
System / Bounce Email Filter

This better reflects that the filter blocks both automated system emails and delivery failure notifications.

**2. Additional deterministic rules added**

The filter now blocks obvious automated emails before reaching the OpenAI classification module.

Rules added:

Sender email contains:
- stripe
- instashop
- t.shopifymail.com
- postmaster

Headers.basic.return-path contains:
- bounce
- noreply
- no-reply
- mailer-daemon

Headers.basic.content-type contains:
- report

Purpose:
Prevent bounce notifications, system alerts, and known automated service emails from entering the AI processing pipeline.

**3. OpenAI prompt improvement**

Additional instructions were added to the OpenAI classification prompt to better detect automated or mailing-list emails.

The model now classifies as **spam** when indicators such as the following appear:

- mailing list headers
- Google Groups notifications
- helpdesk ticket updates from external systems
- automated discussion threads
- promotional newsletters unrelated to Gastronom business

Goal:
Improve classification accuracy and prevent automated messages from entering the structured dataset.
