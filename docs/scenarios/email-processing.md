# Scenario: Email Processing and Classification

## Purpose
Automatically process incoming emails, classify them using AI, and store structured data in Airtable.

This scenario ensures that only meaningful customer emails are analyzed while spam or irrelevant messages are ignored.

---

# Trigger

Module: Gmail Watch Emails

Trigger conditions:
- New email received
- Inbox monitored continuously

---

# Step 1 — Preprocessing

Module: Text Processing

Actions:
- Remove email signature
- Remove quoted previous messages
- Extract only the latest message

Output:
clean_email_text

---

# Step 2 — AI Classification

Module: OpenAI

Prompt used:
Classify the email and extract structured fields.

Expected output fields:

- issue_category
- issue_priority
- customer_intent
- language
- order_number
- phone_number
- customer_name

Example output:

```json
{
  "issue_category": "order_status",
  "priority": "medium",
  "language": "ru",
  "order_number": "4109"
}
