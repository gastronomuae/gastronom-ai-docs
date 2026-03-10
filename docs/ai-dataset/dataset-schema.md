# Dataset Schema

Each dataset row represents a **sender block** in a conversation (one or more consecutive messages from the same sender).

The dataset is stored in Airtable and used to train and evaluate AI classification models.

All fields must exist in every dataset row.  
If a value is unknown or not applicable, leave the field empty.

---

# Core Identifiers

conversation_id  
Unique identifier of the conversation.

conversation_hash  
Identifier used to group all rows belonging to the same conversation.

channel  
Source channel of the conversation.

Possible values:

- whatsapp_B
- instagram
- email

wa_number  
Customer WhatsApp phone number when applicable.

order_id  
Order number detected in the conversation.

message_id_external  
Sequential number of the sender block inside the conversation.

message_direction  

- inbound = customer message  
- outbound = store/agent message

message_source  
Automation that generated the dataset row.

---

# Message Content

message_text  
Merged text of all messages inside the sender block.

timestamp_utc  
Timestamp of the first message in the sender block.

conversation_started_at  
Timestamp of the first message in the conversation.

conversation_closed_at  
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

issue_category  

Detailed classification of support issues.

Definitions are maintained in:

docs/ai-dataset/taxonomy.md

priority  

Operational priority level.

Allowed values:

high  
normal  
low

confidence_score  

AI classification confidence score between **0 and 1**.

customer_sentiment  

Estimated tone of the customer.

Allowed values:

positive  
neutral  
negative

---

# Operational Fields

agent_name  

Name of the responding agent when available.

intent_tag  

Short derived intent label used for analytics.

resolution_status  

Allowed values:

unresolved  
resolved  
auto_resolved

conversation_status  

Allowed values:

open  
waiting_customer  
waiting_agent  
resolved  
auto_resolved  
closed  
escalated

response_time_seconds  

Response time if calculated externally.

escalation_flag  

Indicates if the conversation was escalated.

---

# AI Assistance Fields

ai_suggested_reply  

Reply proposed by AI.

ai_reply_used  

Indicates if the AI reply was used.

---

# Governance Fields

label_source  

Indicates how the dataset row was labeled.

Typical value:

human_labeled
