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

# Trigger - Module 4 - Webhooks → Custom Webhook

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
 AND
    {{4.entry[].changes[].value.messages[].type}} =  text
```

**Purpose**

Ignore non-message events such as:

-   delivery receipts\
-   read confirmations\
-   system status updates

Only real inbound customer messages pass through.

------------------------------------------------------------------------

# Step 2 --- Extract Variables

**Module: 13** Tools → Set Multiple Variables

Variables created for easier mapping later in the scenario.

### message_text
```
    {{if(4.entry[].changes[].value.messages[].type = "text"; 4.entry[].changes[].value.messages[].text.body; )}}
```
### wa_number
```
    {{4.entry[].changes[].value.messages[].from}}
```
### channel
```
    whatsapp
```
### timestamp

    {{formatDate(parseDate(4.entry[].changes[].value.messages[].timestamp; "X"); "YYYY-MM-DD HH:mm:ss"; "Asia/Dubai")}}

This converts the Unix timestamp from the webhook into Dubai local time.

---

# Step 3 --- Airtable 35 Create Record (Message Buffer)

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

# Step 4 --- Sleep 33 (Buffer Window)

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

# Step 5 --- Airtable 17 Search Records (Message Buffer Lookup)

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

# Step 6 --- Text Aggregator 18 (Merge Messages)

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

# Step 7 🔀 Module 51 — If / Else (Image Detection)

Condition: {{4.entry[].changes[].value.messages[].type}} = image

## 🖼️ IF Branch — Image Processing Flow
🔧 Module 53 — Set Variable

Purpose: Extract image ID from WhatsApp payload
image_id = {{4.entry[].changes[].value.messages[].image.id}}

### 🌐 Module 56 — HTTP (Get Image Metadata)

Request:
  Method: GET
  URL: https://graph.facebook.com/v18.0/{{53.image_id}}
  Headers: Authorization: Bearer <ACCESS_TOKEN>
  ⚠️ Uses token from unpublished Meta app

### ⬇️ Module 59 — HTTP (Download Image File)

  Request: Method: GET
  URL: {{57.data.url}}
  Headers: Authorization: Bearer <ACCESS_TOKEN>
  
### 🤖 Module 61 — OpenAI (Image Analysis)
  Model: gpt-4.1-mini
  Mode: System
  Image Input: HTTP file (Module 59)

#### Prompt
```
Analyze this image.

Classify into ONE of these types:

ORDER = order confirmation, receipt, invoice screenshot  
SCREENSHOT = product page, website, catalog, pricing screen  
PHOTO = real-life photo taken by user (product, delivery, groceries, etc.)

Return exactly 5 values separated by | in this order:

type|order_number|product_name|issue|summary

RULES:

1. order_number
- ONLY for ORDER
- Otherwise null

2. product_name
- Extract if clearly visible
- Otherwise null

3. issue
- ALWAYS null (handled later by main classifier)

4. summary
- Short description of what is visible
- If PHOTO → mention "real-life" or "delivered" when relevant
- Never null

5. General
- Use exact word: null
- No explanations
- No JSON
- Return one single line
- Do not use double quotes or single quotes anywhere
- Do not wrap any value in quotes
- Do not use the | character inside values
```

### 🧩 Module 62 — Set Multiple Variables

Purpose: Parse OpenAI output
```
image_type    = {{get(split(61.result; "|"); 1)}}
order_number  = {{get(split(61.result; "|"); 2)}}
product_name  = {{get(split(61.result; "|"); 3)}}
image_issue   = {{get(split(61.result; "|"); 4)}}
image_summary = {{get(split(61.result; "|"); 5)}}
has_image     = true
```

### 🗂️ Module 68 — Airtable (Create Record — Temp Buffer)

Purpose: Store image-derived data for later aggregation

Fields:
```
wa_number     = {{13.wa_number}}
timestamp     = {{13.timestamp}}

image_type    = {{62.image_type}}
order_number  = {{62.order_number}}
product_name  = {{62.product_name}}
image_issue   = {{62.image_issue}}
image_summary = {{62.image_summary}}
has_image     = true
```

## 💬 ELSE Branch — Text Processing
🔧 Module 60 — Set Variable : message_text = {{18.text}}

## 🔄 Shared Flow (Post Merge) 51

After IF/ELSE, flow continues to:

Airtable (conversation history)
Aggregator (message buffer)
OpenAI (main classification)

---

# Step 8 --- Airtable Search rcords 47 (Airtable Lookup)

The scenario searches the **conversation_log** table for recent messages belonging to the same conversation.

Lookup parameters:

| Field | Condition |
|------|------|
| Sort | timestamp ascending |
| Limit | 4 |
| Formula | {conversation_hash} = "{{13.wa_number}}" |

This returns the most recent conversation history between the customer and the store.

---

---
## Filter - Latest Run Only
```
  {{length(trim(13.message_text))}} > 0
AND
  {{18.text}} Ends with (case insensitive) {{13.message_text}}
OR
  {{length(trim(13.message_text))}} = 0

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

# Step 9 --- Tools - Text Aggregator 48 - Context Formatting

The retrieved messages are formatted into a text block and passed to the AI classifier as conversation context.
Source Module: Airtable 47
Row separator: New row
Text: {{47.conversation_hash}}: {{47.message_text}}

Example:

Customer: When will my order arrive?
Store: It should arrive today before 6pm
Customer: 4177

Customer: When will my order arrive?
Store: It should arrive today before 6pm
Customer: 4177



# Step 10 --- OpenAI Classification 44

**Module:** OpenAI

Purpose:

Classify the incoming message for routing, prioritization, and dataset
tagging.

Output format returned by the model:

    broad_category|issue_category|priority|confidence|escalation_flag

Example output:

    support|order_status|normal|0.94|false

These values are then mapped into Airtable fields.

## Prompt
```
Classify the WhatsApp message below.

Return exactly 5 values separated by | in this order:

broad_category|issue_category|priority|confidence|escalation_flag

broad_category (choose exactly one):

support
supplier_prospect
b2b_sales
marketing
spam
other

Definitions:

support = Customer inquiry related to product availability, order status, delivery, payment, refund, complaint, store location, working hours, or any retail purchase.

b2b_sales = A business buyer (restaurant, hotel, nursery, catering, retail shop, distributor, reseller) asking to purchase our products, request wholesale price list, MOQ, delivery terms, or partnership to BUY from us.

supplier_prospect = Producer, factory, exporter, or brand offering PHYSICAL GOODS supply to us, asking to distribute their products or send catalog / commercial offer.

marketing = Service providers, agencies, influencers, SEO, advertising offers, collaboration proposals, digital marketing, affiliate offers, traffic generation.

spam = Scam, crypto, suspicious links, fake courier warning, impersonation, irrelevant bulk messages.

other = Legitimate message that does not clearly fit above (e.g., job inquiry, greeting without context).

---

IMPORTANT CLASSIFICATION RULES

CONTEXT CONSISTENCY RULE

If the message appears to be part of an ongoing conversation, the issue_category should normally remain the same as the original customer inquiry.

Outbound replies from the store should inherit the issue_category of the customer message they are answering unless the topic clearly changes.

Do not change issue_category unless the customer explicitly introduces a new topic.

If the message contains only a greeting (e.g., "Hi", "Hello", "Здравствуйте") without any additional question or context →  
other | null | low | 0.80

If a greeting message contains an additional question about products, delivery, orders, or payment in the same message, ignore the greeting and classify based on the question.

Examples:

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

---

DELIVERY / SERVICE QUESTIONS

Questions about delivery availability, delivery areas, shipping coverage, or whether delivery works in a specific location must be classified as:

support | delivery_area

Examples:

Do you deliver to Dubai Marina?  
Do you deliver to Al Ain?  
Is delivery available today?  
Do you deliver to JVC?

These questions are about service coverage, not order tracking.

---

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

---

ORDER MODIFICATION QUESTIONS

Requests to change an existing order must be classified as:

support | order_modification

Examples:

Can you change my order?  
I want to remove one item  
Please change delivery time  
Add one more product to my order

---

OUTBOUND MESSAGE CLASSIFICATION

If the message is written by the store (agent reply), use the same issue_category as the customer's message that triggered the reply unless the reply introduces a completely different topic.

Example:

Customer: Do you deliver to Dubai Marina?  
Store: Yes, delivery is available.

Both messages should use:

support | delivery_area

---

priority rules:

If broad_category = b2b_sales → high.  
If broad_category = supplier_prospect → normal.  
If support AND issue_category in (complaint, refund) → high.  
If support other → normal.  
If marketing → low.  
If spam → low.  
If other → low.

---

confidence:

Return a number between 0.0 and 1.0.

---

ESCALATION RULES

Set escalation_flag = true when human intervention is required.

Escalate in the following situations:

1. issue_category = complaint  
2. issue_category = refund  
3. issue_category = cancellation  
4. broad_category = b2b_sales  
5. broad_category = supplier_prospect  
6. The sender explicitly asks to speak with a human, manager, or support agent.

In all other cases:

escalation_flag = false

---

ESCALATION SAFETY RULE

If broad_category = spam or marketing  
then escalation_flag must always be false.

---

Return only the pipe-separated values.  
Do not include explanations.

RECENT CONVERSATION CONTEXT {{48.text}}

Sender: {{18.text}}
New Message: {{13.message_text}}

---

IMAGE CONTEXT (if exists):
Type: {{63.image_type}}
Order: {{63.order_number}}
Product: {{63.product_name}}

Visual summary: {{63.image_summary}}


Detected issue (from image):
{{63.image_issue}}

If IMAGE CONTEXT fields are empty or null, ignore them completely and classify based only on the text message and conversation context.

```
---

# Step 11 --- 🧱 Module 67 — Set Variable (Final Message)

Purpose: Combine text + image context into one message
```
final_message_text = 
{{trim(18.text)}}
{{if(63.image_type != ""; " Image type:   " + 63.image_type; "")}}
{{if(63.product_name != ""; " Image product:  " + 63.product_name; "")}}
{{if(63.image_issue != ""; " Image issue:  " + 63.image_issue; "")}}
{{if(63.image_summary != ""; " Image summary:  " + 63.image_summary; "")}}
```


# Step 12 --- Airtable Create Record 45

**Module:** Airtable → Create Record

Table: `support_messages`

| Field | Value |
|------|--------|
| **conversation_id** | {{4.entry[].changes[].value.messages[].id}} |
| **wa_number** | {{13.wa_number}} |
| **message_id_external** | {{4.entry[].changes[].value.messages[].id}} |
| **message_direction** | inbound |
| **message_source** | whastapp_A |
| **message_text** | {{63.order_number}} |
| **broad_category** | support |
| **issue_category** | {{if(get(split(44.result; "|"); 2) = "null"; ""; get(split(44.result; "|"); 2))}} |
| **timestamp** | {{13.timestamp}} |
| **conversation_status** | open |
| **priority** | {{get(split(44.result; "|"); 3)}} |
| **channel** | whastapp_A |
| **confidence** | {{get(split(44.result; "|"); 4)}} |
| **conversation_hash** | {{4.entry[].changes[].value.messages[].from}} |



The Airtable record ID generated here becomes the **primary reference ID
for the conversation**.

---
# Step 13 --- 🧾 Module 70 — JSON (Create JSON)

Purpose: Prepare structured payload for HTTP module

Fields:
```
{
  "message_text": "",
  "image_type": "",
  "order_number": "",
  "product_name": "",
  "image_issue": "",
  "image_summary": ""
}
```
Mapping:
```
message_text  = {{63.message_text}}
image_type    = {{63.image_type}}
order_number  = {{63.order_number}}
product_name  = {{63.product_name}}
image_issue   = {{63.image_issue}}
image_summary = {{63.image_summary}}
```


---

# Step 14 --- Trigger Scenario 07 (AI Reply Draft) - HTTP Module 46

Purpose: Trigger the **central AI reply generator scenario**.

Scenario 07 generates a suggested support reply that will later be shown
to the operator in Telegram.

POST request body:

``` json
{
"airtable_record_id": "{{11.id}}",
"channel": "whatsapp_A",
"message_text": "{{63.message_text}}",
"image_type": "{{63.image_type}}",
"product_name": "{{63.product_name}}",
"image_issue": "{{63.image_issue}}",
"image_summary": "{{63.image_summary}}",
"image_type": "{{63.image_type}}",
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

# Step 15 --- Airtable 42 Search Records Again (Buffer Cleanup) 

Table: message_buffer

Formula: Search all buffered rows for the same wa_number (same as in step 10, just with different formula)
```
{wa_number} = "{{13.wa_number}}"
```

Purpose:
Re-fetch buffered records after the merged conversation has already been logged and Scenario 07 has been triggered.
This second search is used only for cleanup.

---

# Step 16 --- Airtable 43 Delete Record(s) (Buffer Cleanup)

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
