# Scenario 4 — Telegram → WhatsApp A Reply Routing

## Objective

Allow support team members to reply inside the Telegram support group and automatically forward that reply back to the original WhatsApp customer using WhatsApp Number A.

This creates a controlled two‑way bridge:

WhatsApp → Telegram → WhatsApp

---

# Architecture Overview

Telegram Bot (Watch Updates)

↓

Filter — Only replies to bot

↓

Tools — Extract WhatsApp number

↓

WhatsApp Business Cloud — Send Message

↓

Airtable — Log outbound message

---

# Module 1 — Telegram Bot: Watch Updates

Name: Telegram → WA A routing

Trigger Type:
Webhook

Connection:
Telegram Reply Router

Purpose:

• Listen for new messages in the Telegram support group  
• Capture reply‑to structure  
• Extract message text and reply context  

Example Output

```json
[
{
"message": {
"message_id": 47,
"from": {
"id": 570449120,
"is_bot": false,
"first_name": "Vahag",
"username": "vohanjanyan"
},
"chat": {
"id": -5133624518,
"title": "Gastronom cusomer support",
"type": "group",
"all_members_are_administrators": true,
"accepted_gift_types": {
"unlimited_gifts": false,
"limited_gifts": false,
"unique_gifts": false,
"premium_subscription": false,
"gifts_from_channels": false
}
},
"date": "2026-02-19T16:48:52.000Z",
"reply_to_message": {
"message_id": 45,
"from": {
"id": 8135508887,
"is_bot": true,
"first_name": "gastronom.support",
"username": "gastronomae_bot"
},
"chat": {
"id": -5133624518,
"title": "Gastronom cusomer support",
"type": "group",
"all_members_are_administrators": true,
"accepted_gift_types": {
"unlimited_gifts": false,
"limited_gifts": false,
"unique_gifts": false,
"premium_subscription": false,
"gifts_from_channels": false
}
},
"date": 1771519562,
"text": "📩 New WhatsApp Message\n\n👤 WA A: 971561345294\n\n💬 Can you deliver tomorrow at 2pm?"
},
"text": "sure!",
"attachment": {
"type": "OtherOrNone"
}
},
"update_id": 763321569,
"channel_post": null,
"inline_query": null,
"callback_query": null,
"edited_message": null,
"my_chat_member": null,
"business_message": null,
"pre_checkout_query": null,
"edited_channel_post": null
}
]
```
---

# Filter — Only Replies to Bot

Label:
Only replies to bot

Conditions:

Message.reply_to_message.message_id exists
AND
Message.reply_to_message.from.is_bot = true
message.text = send
OR
message.text starts with /reply

Purpose:

• Ensure agent replies to a bot‑generated message  
• Prevent forwarding random Telegram messages  
• Ignore internal group chat messages  

---

# Module 2 — Tools: Set Variable

Variable Name:
wa_number

Purpose:

Extract the WhatsApp number from the original Telegram bot message.

Original Message Format

📩 New WhatsApp Message

👤 WA A: 971561345294

💬 Customer message here

Extraction rule:

Take number after

WA A:

Result Example

{
 "wa_number": "+971561345294"
}

---

# Module 3 — WhatsApp Business Cloud: Send Message

Sender ID:
1016151078247095

Receiver:
wa_number

Message Type:
Text

Body:
Telegram message text

Purpose:

Send the Telegram reply back to the original WhatsApp customer.

Example Output
```json
{
 "messaging_product": "whatsapp",
 "contacts": [
  {
   "input": "+971561345294",
   "wa_id": "971561345294"
  }
 ],
 "messages": [
  {
   "id": "wamid.examplemessageid"
  }
 ]
}
```
---

# Module 4 — Airtable: Create Record

Base:
AI Staff – Conversation Engine

Table:
conversation_log

Purpose:

Log outbound staff replies after successful WhatsApp delivery.

Record Mapping

conversation_id = Telegram message_id
wa_number = replace(wa_number, "+", "")
message_id_external = Telegram message_id
message_direction = outbound
message_source = telegram
message_text = Telegram text
timestamp_utc = current UTC timestamp
agent_name = Telegram sender first_name
resolution_status = unresolved
conversation_status = open
channel = whatsapp_A
conversation_hash = wa_number
broad_category = support
issue_category = agent_reply
priority = normal
confidence_score = 1
label_source = human_labeled

Example Record

```json
{
 "conversation_id": "106",
 "wa_number": "971561345294",
 "message_direction": "outbound",
 "message_source": "telegram",
 "message_text": "Sure",
 "agent_name": "Vahag",
 "conversation_status": "open"
}
```
---

# Logging Logic

Airtable logging occurs only when:

• Agent replies to a bot‑forwarded WhatsApp message  
• Reply contains text  
• WhatsApp send module executes successfully  

Prevents logging of:

• delivery receipts  
• system events  
• non‑text messages  

---

# System Behaviour

This is currently a manual relay system.

Requirements:

• Agent must reply to bot message  
• Filter must be satisfied  

Otherwise the message is ignored.

---

# Sample Flow

Customer sends:

Thanks. When expect delivery?

↓

Telegram receives

📩 New WhatsApp Message
👤 WA A: 971561345294
💬 Thanks. When expect delivery?

↓

Agent replies in Telegram

Delivery tomorrow between 12–2pm.

↓

Customer receives WhatsApp reply

Delivery tomorrow between 12–2pm.

---

# Risk Notes

• If agent replies without using reply function → ignored  
• If WA number extraction fails → message not sent  
• If WhatsApp API fails → no retry mechanism currently  

Future improvement:

Add retry logging and error alerts.
