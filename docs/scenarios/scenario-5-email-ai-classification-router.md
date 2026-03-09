
---
scenario_id: 05
scenario_name: email_ai_classification_router
system: make.com
trigger: email watcher
status: active
---

# Scenario 5 — Email Watcher (AI Classification)

## Objective

Monitor the support inbox **ai@gastronom.ae**, prevent duplicate processing, classify emails using AI, route important messages to Telegram, and log all email activity into Airtable for an AI‑ready dataset.

This scenario builds the foundation for **AI‑powered customer support intelligence**.

---

# Architecture Overview

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

OpenAI Classification

↓

Router – Importance Filter

→ Telegram Notification  
→ Airtable Create Record

---

# Module 1 — Email: Watch Emails

Purpose:

Trigger when a new email arrives in the **ai@gastronom.ae** mailbox.

Captured fields:

• Subject  
• Sender email address  
• Text content  
• Headers → message-id  
• Date

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

{message_id_external} = "{{headers.message-id}}"

Limit: 1

If the Message-ID already exists, the scenario stops execution.

---

# Module 3 — Router: Duplicate Detection

Branch 1 — Already Processed

Condition

Total bundles > 0

Action

STOP

Branch 2 — New Email

Condition

Total bundles = 0

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

• Prevent unnecessary OpenAI usage  
• Prevent staff messages entering AI dataset  

---

## Branch B — INBOUND (Customer)

Fallback route for all other emails.

Processing:

OpenAI Classification → Router 2

---

# Module 5 — OpenAI Classification

Model

gpt-5-nano

Purpose

Classify the email into structured fields used by the support intelligence system.

Output format

broad_category|issue_category|priority|confidence|customer_email|supplier_region

Possible broad_category values

support  
supplier_prospect  
b2b_sales  
marketing  
other  

Example output

support|order_status|normal|0.83|customer@email.com|null

Meaning:
- broad_category = support
- issue_category = order_status
- priority = normal
- confidence = 0.83
- customer_email = customer@email.com
  if the message is submitted via the website contact form or email address found in body of email (example: signature) and the email header shows a generic sender (e.g., a Shopify notification address), the AI should
  instead extract and return the email address provided by the customer inside the message body. Otherwise = null
- supplier_region = used only when broad_category = supplier_prospect.
  Determines whether the supplier belongs to Gastronom’s priority sourcing region.

Possible values:
- priority_region
Supplier located in Russia, Armenia, Georgia, Uzbekistan, Kazakhstan, Belarus, 
OR suppliers located in the UAE producing CIS-style products targeted at Russian / Eastern  European customers (e.g., pelmeni, vareniki, manty, CIS bakery products).

other_region:
- Supplier located outside the priority region and not related to CIS-style food supply (e.g., Vietnam, India, China, generic global exporters).

unknown
- Supplier region cannot be determined from the email content.

null
- Returned when the message is not classified as supplier_prospect.


PROMPT

```json
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

# Module 6 — Router: Support Importance Filter

Purpose

Send only important messages to Telegram.

Condition

broad_category = support  
OR  
broad_category = b2b_sales  
OR  
(broad_category = supplier_prospect AND supplier_region = priority_region)

---

# Module 7 — Telegram Notification

Purpose

Alert the support team of important emails.

Message format

📧 CUSTOMER EMAIL

From: {{sender}}  
Subject: {{subject}}

Message:
{{first_250_characters_of_email}}

---

# Module 8 — Airtable: Create Record

Base

AI Staff – Conversation Engine

Table

conversation_log

Purpose

Create a structured dataset for analytics and AI training.

---

# Field Mapping — INBOUND

conversation_id → message-id  
message_id_external → message-id  
message_direction → inbound  
message_source → email  
channel → email  
message_text → first trimmed section of email  
timestamp_utc → parsed email date  

AI fields

broad_category → OpenAI result  
issue_category → OpenAI result  
priority → OpenAI result  
confidence_score → OpenAI result  

Status fields

resolution_status → unresolved  
conversation_status → open

conversation_hash

Uses extracted customer email or sender email.

---

# Field Mapping — OUTBOUND

conversation_id → message-id  
message_direction → outbound  
message_source → email  
channel → email  
message_text → trimmed email content  
timestamp_utc → email date  

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
