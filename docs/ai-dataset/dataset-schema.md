# Dataset Schema

Each dataset row represents a **sender block** in a conversation (one or more consecutive messages from the same sender).

The dataset is stored in Airtable and used to train and evaluate AI classification models.

All fields must exist in every dataset row.  
If a value is unknown or not applicable, leave the field empty.

---

# Core Identifiers

conversation_id  
Unique identifier of the conversation.

## conversation_hash

Stable identifier used to group dataset rows belonging to the same conversation.

Typical implementations:

WhatsApp  
conversation_hash = wa_number

Email  
conversation_hash = email thread ID

Instagram  
conversation_hash = platform conversation ID

## channel

Indicates the communication platform.

Allowed values:

whatsapp_B  
instagram  
email

Additional channels may be added in future versions of the dataset.

wa_number  
Customer WhatsApp phone number when applicable.

order_id  
Order number detected in the conversation.

message_id_external  
Sequential number of the sender block inside the conversation.

## message_direction

Indicates whether the message was sent by the customer or the store.

Allowed values:

inbound  
outbound

Definitions:

inbound = message from customer  
outbound = message from store agent or system

message_source  
Automation that generated the dataset row.

---

# Message Content

message_text  
Merged text of all messages inside the sender block.

Contains the merged message text of a sender block.

Multiple messages from the same sender may be merged into a single text block.

Recommended maximum length:

5000 characters

## Sender Block Rule

Dataset rows represent sender blocks rather than individual messages.

A sender block is defined as:

One or more consecutive messages from the same sender.

Rules:

• Consecutive customer messages are merged into one inbound dataset row  
• Consecutive store messages are merged into one outbound dataset row  
• A new dataset row starts whenever the sender changes


# timestamp_utc  
Timestamp of the first message in the sender block.

# conversation_started_at  
Timestamp of the first message in the conversation.

# conversation_closed_at  
Timestamp of the last message when the conversation is closed.

---

# Classification Fields

broad_category  

High-level classification of the message.

Allowed values:

support  
supplier_prospect  
b2b_sales  
marketing  
spam  
other

### issue_category  

Detailed classification of support issues.

Definitions are maintained in:

docs/ai-dataset/taxonomy.md

Issue Category Constraint

issue_category may only contain a value when:

broad_category = support

If broad_category is not support, issue_category must be:

null

Examples:

supplier_prospect → issue_category = null  
marketing → issue_category = null  
spam → issue_category = null

## priority

Indicates the operational urgency of the message.

Allowed values:

high  
normal  
low  

Priority is determined by classification rules.

Typical mapping:

| broad_category / issue | priority |
|-----------------------|----------|
| b2b_sales | high |
| support complaint | high |
| support refund | high |
| supplier_prospect | normal |
| support other | normal |
| marketing | low |
| spam | low |
| other | low |

## confidence_score

Represents the model's confidence in the assigned classification.

Type: float  
Range: 0.0 – 1.0

Guidelines:

0.90 – 1.00  
Intent extremely clear

0.75 – 0.89  
Very likely correct

0.50 – 0.74  
Ambiguous message

Below 0.50  
Highly uncertain classification

## customer_sentiment

Represents the emotional tone of the customer message.

Allowed values:

positive  
neutral  
negative

Examples:

positive  
"Thank you very much"

neutral  
"Do you deliver to JVC?"

negative  
"This delivery is very late"

---

# Operational Fields

agent_name  

Name of the responding agent when available.

intent_tag  

Short derived intent label used for analytics.

## resolution_status

Represents whether the issue has been resolved.

Allowed values:

unresolved  
resolved  
auto_resolved

Rules:

If conversation_status = closed  
then resolution_status must be resolved.

## conversation_status

Represents the operational state of the conversation.

Allowed values:

open  
waiting_customer  
waiting_agent  
resolved  
auto_resolved  
escalated  
closed

Definitions:

open  
Conversation started but next action not yet determined.

waiting_agent  
Customer sent a message and agent response is expected.

waiting_customer  
Agent requested information or action from the customer.

resolved  
Issue solved by the agent but conversation not yet closed.

auto_resolved  
Issue solved automatically by system or automation.

escalated  
Conversation requires human escalation or manual handling.

closed  
Conversation fully finished and no further action expected.


response_time_seconds  

Response time if calculated externally.

## escalation_flag

Boolean.

Indicates whether the message triggered escalation to a human agent.

Allowed values:

true  
false

---

# AI Assistance Fields

ai_suggested_reply  

Reply proposed by AI.

## ai_reply_used

Boolean.

Indicates whether the AI-generated reply was actually sent to the customer.

Allowed values:

true  
false

---

# Governance Fields

## label_source

Indicates how the classification label was generated.

Allowed values:

| value | description |
|------|-------------|
| ai_prediction | Label generated automatically by AI during live automation. |
| human_labeled | Label assigned manually during dataset preparation or review. |
| simulation | Label produced during test scenarios or simulation runs. |
| system_default | Label assigned automatically by system fallback rules when classification cannot be determined. |

Example:

label_source = ai_prediction

---





