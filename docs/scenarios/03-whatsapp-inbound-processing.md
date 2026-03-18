# Scenario 03 --- WhatsApp A → AI Classification → Airtable → Trigger AI Reply

## Purpose

This scenario listens to incoming WhatsApp messages sent to the
automated WhatsApp Business number (WA A), classifies the message using
AI, stores it in Airtable, and triggers **Scenario 07** to generate an
AI support reply draft.

This scenario **does not send replies itself**.\
It acts as the **ingestion and classification pipeline**.

------------------------------------------------------------------------

# High-Level Flow

```
WhatsApp Webhook\
↓\
Filter inbound messages\
↓\
Extract variables\
↓\
AI classification\
↓\
Airtable create record\
↓\
Trigger Scenario 07 (AI reply generator)
```
------------------------------------------------------------------------

# Trigger

**Module:** Webhooks → Custom Webhook

This webhook receives events from the **Meta WhatsApp Business Cloud
API**.

Example payload:

``` json
{
  "entry": [
    {
      "changes": [
        {
          "field": "messages",
          "value": {
            "messages": [
              {
                "from": "971561345294",
                "text": {
                  "body": "When will my order arrive?"
                },
                "timestamp": "1773245406"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

------------------------------------------------------------------------

# Step 1 --- Filter Inbound Messages

**Filter name:**

    Message filter only

**Conditions:**

    4.entry[].changes[].field = messages
    AND
    4.entry[].changes[].value.messages exists

**Purpose**

Ignore non-message events such as:

-   delivery receipts\
-   read confirmations\
-   system status updates

Only real inbound customer messages pass through.

------------------------------------------------------------------------

# Step 2 --- Extract Variables

**Module:** Tools → Set Multiple Variables

Variables created for easier mapping later in the scenario.

### message_text

    {{4.entry[].changes[].value.messages[].text.body}}

### wa_number

    {{4.entry[].changes[].value.messages[].from}}

### channel

    whatsapp

### timestamp

    {{formatDate(parseDate(4.entry[].changes[].value.messages[].timestamp; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}

This converts the Unix timestamp from the webhook into Dubai local time.


---

# Step 3 --- AI Classification

**Module:** OpenAI

Purpose:

Classify the incoming message for routing, prioritization, and dataset
tagging.

Output format returned by the model:

    broad_category|issue_category|priority|confidence|escalation_flag

Example output:

    support|order_status|normal|0.94|false

These values are then mapped into Airtable fields.

------------------------------------------------------------------------

# Step 4 --- Airtable Create Record

**Module:** Airtable → Create Record

Table: `support_messages`

Fields mapped:

  Field               Value
  ------------------- ------------------
  channel             whatsapp
  message_text        {{message_text}}
  wa_number           {{wa_number}}
  timestamp           {{timestamp}}
  broad_category      OpenAI result
  issue_category      OpenAI result
  priority            OpenAI result
  confidence          OpenAI result
  escalation_flag     OpenAI result
  message_direction   inbound

The Airtable record ID generated here becomes the **primary reference ID
for the conversation**.

------------------------------------------------------------------------

# Step 5 --- Trigger Scenario 07 (AI Reply Draft)

**Module:** HTTP → Make a Request

Purpose:

Trigger the **central AI reply generator scenario**.

Scenario 07 generates a suggested support reply that will later be shown
to the operator in Telegram.

POST request body:

``` json
{
"airtable_record_id": "{{11.id}}",
"channel": "whatsapp_A",
"message_text": "{{11.message_text}}",
"broad_category": "{{11.broad_category}}",
"issue_category": "{{11.issue_category}}",
"sentiment": "{{ifempty(11.customer_sentiment; "unknown")}}",
"priority": "{{11.Priority}}",
"message_direction": "{{11.message_direction}}",
"timestamp_utc": "{{11.timestamp_utc}}",
"customer_phone": "{{11.wa_number}}",
"conversation_hash": "{{11.conversation_hash}}"
}
```

--- 
Scenario 07 then:

1.  Retrieves the Airtable record\
2.  Generates an AI reply draft\
3.  Updates Airtable with `ai_suggested_reply`\
4.  Sends a Telegram notification

------------------------------------------------------------------------

# Resulting System Architecture

Scenario 03 (Channel Ingestion)

WhatsApp / Instagram → Classification → Airtable → Trigger AI Reply

Scenario 07 (AI Brain)

Generate suggested reply → Update Airtable → Send Telegram notification

Scenario 08 (Operator Actions)

Telegram reply → Airtable lookup → Send message to customer

------------------------------------------------------------------------

# Key Design Principles

• Channel ingestion separated from AI reply logic\
• Centralized AI reply generator (Scenario 07)\
• Airtable used as the system of record\
• Telegram used as the operator control layer

This design allows scaling the system across:

-   WhatsApp\
-   Instagram\
-   Email\
-   Webchat

without duplicating AI logic.

------------------------------------------------------------------------

# Future Enhancement

Once the system is stable, Telegram human approval can be bypassed and
Scenario 07 can automatically send replies for low-risk support
questions.

This enables fully automated customer support workflows.
