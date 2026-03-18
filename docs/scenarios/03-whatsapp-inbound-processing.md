# Scenario 03 — WhatsApp A → Buffer → Aggregate → AI Classification → Airtable → Trigger AI Reply

## Purpose
This scenario listens to incoming WhatsApp messages sent to the automated WhatsApp Business number (WA A), temporarily buffers them, merges multiple customer messages sent within a short time window into a single combined message, classifies the merged message using AI, stores it in Airtable, and triggers **Scenario 07** to generate an AI support reply draft.

This scenario **does not send the customer reply directly itself**.  
It acts as the **buffering, aggregation, ingestion, and classification pipeline**.

------------------------------------------------------------------------

# High-Level Flow

```
WhatsApp Webhook
↓
Filter inbound messages
↓
Extract variables
↓
Airtable create record (message_buffer)
↓
Sleep 30 seconds
↓
Airtable search records (same wa_number)
↓
Text Aggregator (merge messages)
↓
Filter latest run only
↓
AI classification
↓
Airtable create record (conversation_log / support_messages)
↓
Trigger Scenario 07 (AI reply generator)
↓
Airtable search records again (same wa_number)
↓
Airtable delete buffered records
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
```
    4.entry[].changes[].field = messages
    AND
    4.entry[].changes[].value.messages exists
```

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

# Step 3 --- Airtable Create Record (Message Buffer)

Table: message_buffer

Purpose:
Store every inbound WhatsApp message first in a temporary buffer table so that multiple messages sent within a short period can be merged before AI processing.

| Field | Value |
|------|--------|
| **wa_number** | {{wa_number}} |
| **message_text** | {{message_text}} |
| **timestamp** | {{timestamp}} |
| **processed** | false / empty |

---

# Step 4 --- Sleep (Buffer Window)

Value: 30 seconds

Purpose: Allow the customer to send follow-up messages in quick succession before processing begins.

Example:
```
“Hello”
“Where is my order?”
“4201”
```
These should be treated as one grouped support message instead of three independent AI requests.

---

# Step 5 --- Airtable Search Records (Message Buffer Lookup)

Table: message_buffer
All rows - sorted by timestamp (ascending) 
Formula: NOT({processed})


Purpose:
Retrieve all buffered messages from the table that are not processed.

---
## Filter - same wa_number
{{17.wa_number}} = {{13.wa_number}}
phone number from table we searhced matches phone number from sender (webhook -> set varialbles wa_number)

---

# Step 6 --- Text Aggregator (Merge Messages)

Source module: Airtable Search Records (message_buffer)

Grouped by: wa_number
Text field aggregated: message_text
Row separator: New row
Stop processing after an empty aggregation: Yes

Example output:

Hello
Where is my order?
4201

Purpose:
Combine messages togetehr as a one text, each message starts with new line. So evantuallt we can send as a one mesgae to Telegram, rather than each message as a separate note.

---

## Filter - Latest Run Only
```
{{18.text}}
Ends with (case insensitive)
{{13.message_text}}
```

Purpose:
Because the scenario is triggered once per inbound message, several parallel runs may exist at the same time.
This filter ensures that only the run triggered by the latest message in the buffered sequence continues downstream.

Example:
- Combined text:
    Hello, where is my order?\n4201
- Current message:
    4201

Only the run whose original message_text matches the end of the aggregated text is allowed to proceed.

This prevents:

- duplicate AI calls
- duplicate Airtable merged records
- duplicate Telegram notifications


---

# Step 7 --- AI Classification

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

# Step 8 --- Airtable Create Record

**Module:** Airtable → Create Record

Table: `support_messages`

| Field | Value |
|------|--------|
| **channel ** | whatsapp |
| **message_text ** | {{message_text}} |
| **wa_number** | {{wa_number}} |
| **timestamp** | {{timestamp}} |
| **broad_category** | OpenAI result |
| **issue_category** | OpenAI result |
| **priority** | OpenAI result |
| **confidence** | OpenAI result |
| **escalation_flag** | OpenAI result |
| **message_direction** | inbound |


The Airtable record ID generated here becomes the **primary reference ID
for the conversation**.

------------------------------------------------------------------------

# Step 9 --- Trigger Scenario 07 (AI Reply Draft)

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

---

# Step 10 --- Airtable Search Records Again (Buffer Cleanup)

Table: message_buffer

Formula: Search all buffered rows for the same wa_number (same as in step 10, just with different formula)
```
{wa_number} = "{{13.wa_number}}"
```

Purpose:
Re-fetch buffered records after the merged conversation has already been logged and Scenario 07 has been triggered.
This second search is used only for cleanup.

---

# Step 11 --- Airtable Delete Record(s) (Buffer Cleanup)

Purpose: Delete all buffered rows returned by the cleanup search.

This ensures:

- next customer message starts a fresh buffer window
- old messages are not merged with future ones
- buffer table remains temporary and lightweight

Important:

- Cleanup happens after OpenAI and HTTP trigger
- Otherwise downstream modules may execute multiple times due to multi-bundle search results

---

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
