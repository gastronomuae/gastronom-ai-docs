
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
spam  
other  

Example output

support|order_status|normal|0.83|customer@email.com|null

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
