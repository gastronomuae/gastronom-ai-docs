# Prompt Specification

This document defines all OpenAI prompts used in the AI conversation engine.

Prompts are grouped by channel and purpose.

Each prompt specification includes:

- objective
- input variables
- output format
- model configuration
- prompt body
- version history

# Top-Level Structure

1. Overview
2. Output Standards
3. Email Classification Prompt
4. Instagram DM Classification Prompt
5. WhatsApp Conversation Processing Prompt
6. Versioning Rules

# Overview

This document defines the prompts used by the Gastronom AI support system.

Prompts are responsible for transforming raw messages into structured dataset fields used for:

- support automation
- escalation routing
- dataset generation
- AI training

Each prompt returns deterministic structured outputs used by Make scenarios.

# Output Standards

All prompts must follow these principles:

• deterministic output format  
• no explanations  
• machine-parsable results  
• predictable field ordering  

Prompts must always return exactly the fields required by the scenario.

If a field is not applicable, the prompt must return:

null

---

## Prompt: email_classification


Classify the Instagram direct message below.

Return exactly 4 values separated by | in this order:

broad_category|issue_category|priority|confidence

broad_category (choose exactly one):

support
supplier_prospect
b2b_sales
marketing
spam
other

Definitions:

b2b_sales = A business buyer (restaurant, hotel, nursery, catering, retail shop, distributor, reseller) asking to purchase our products, request wholesale price list, MOQ, delivery terms, or partnership to BUY from us.

supplier_prospect = Producer, factory, exporter, or brand offering PHYSICAL GOODS supply to us, asking to distribute their products or send catalog / commercial offer.

support = Customer inquiry related to product availability, order status, delivery, payment, refund, complaint, store location, working hours, or any retail purchase.

marketing = Service providers, agencies, influencers, SEO, advertising offers, collaboration proposals, digital marketing, affiliate offers, traffic generation.

spam = Scam, crypto, suspicious links, fake courier warning, impersonation, irrelevant bulk messages.

other = Legitimate message that does not clearly fit above (e.g., job inquiry, greeting without context).


--- IMPORTANT CLASSIFICATION RULES ---


CONTEXT CONSISTENCY RULE

If the message appears to be part of an ongoing conversation, the issue_category should normally remain the same as the original customer inquiry.

Outbound replies from the store should inherit the issue_category of the customer message they are answering unless the topic clearly changes.

Do not change issue_category unless the customer explicitly introduces a new topic.

If the message contains only a greeting (e.g., "Hi", "Hello", "Здравствуйте") without any additional question or context → other | null | low | 0.80

If a greeting message contains an additional question about products, delivery, orders, or payment in the same message, ignore the greeting and classify based on the question.

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


DELIVERY / SERVICE QUESTIONS

Questions about delivery availability, delivery areas, shipping coverage, or whether delivery works in a specific location must be classified as:

support | delivery_area

Examples:
"Do you deliver to Dubai Marina?"
"Do you deliver to Alain?"
"Is delivery available today?"
"Do you deliver to JVC?"

These questions are about service coverage, not order tracking.

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

ORDER MODIFICATION QUESTIONS

Requests to change an existing order must be classified as:

support | order_modification

Examples:
"Can you change my order?"
"I want to remove one item"
"Please change delivery time"
"Add one more product to my order"


OUTBOUND MESSAGE CLASSIFICATION

If the message is written by the store (agent reply), use the same issue_category as the customer's message that triggered the reply unless the reply introduces a completely different topic.

Example:

Customer: "Do you deliver to Dubai Marina?"
Store: "Yes, delivery is available."

Both messages should use:
support | order_status


priority rules:

If broad_category = b2b_sales → high.
If broad_category = supplier_prospect → normal (high only if clearly urgent/large-scale).
If support AND issue_category in (complaint, refund) → high.
If support other → normal.
If marketing → low.
If spam → low.
If other → low.

confidence:
Return a number between 0.0 and 1.0.

Return only the pipe-separated line. No explanation.

Message:
{{1.entry[].messaging[].message.text}}

---

## Prompt: instagram_classification

Classify the Instagram direct message below.

Return exactly 4 values separated by | in this order:

broad_category|issue_category|priority|confidence

broad_category (choose exactly one):

support
supplier_prospect
b2b_sales
marketing
spam
other

Definitions:

b2b_sales = A business buyer (restaurant, hotel, nursery, catering, retail shop, distributor, reseller) asking to purchase our products, request wholesale price list, MOQ, delivery terms, or partnership to BUY from us.

supplier_prospect = Producer, factory, exporter, or brand offering PHYSICAL GOODS supply to us, asking to distribute their products or send catalog / commercial offer.

support = Customer inquiry related to product availability, order status, delivery, payment, refund, complaint, store location, working hours, or any retail purchase.

marketing = Service providers, agencies, influencers, SEO, advertising offers, collaboration proposals, digital marketing, affiliate offers, traffic generation.

spam = Scam, crypto, suspicious links, fake courier warning, impersonation, irrelevant bulk messages.

other = Legitimate message that does not clearly fit above (e.g., job inquiry, greeting without context).


--- IMPORTANT CLASSIFICATION RULES ---


CONTEXT CONSISTENCY RULE

If the message appears to be part of an ongoing conversation, the issue_category should normally remain the same as the original customer inquiry.

Outbound replies from the store should inherit the issue_category of the customer message they are answering unless the topic clearly changes.

Do not change issue_category unless the customer explicitly introduces a new topic.

If the message contains only a greeting (e.g., "Hi", "Hello", "Здравствуйте") without any additional question or context → other | null | low | 0.80

If a greeting message contains an additional question about products, delivery, orders, or payment in the same message, ignore the greeting and classify based on the question.

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


DELIVERY / SERVICE QUESTIONS

Questions about delivery availability, delivery areas, shipping coverage, or whether delivery works in a specific location must be classified as:

support | delivery_area

Examples:
"Do you deliver to Dubai Marina?"
"Do you deliver to Alain?"
"Is delivery available today?"
"Do you deliver to JVC?"

These questions are about service coverage, not order tracking.

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

ORDER MODIFICATION QUESTIONS

Requests to change an existing order must be classified as:

support | order_modification

Examples:
"Can you change my order?"
"I want to remove one item"
"Please change delivery time"
"Add one more product to my order"


OUTBOUND MESSAGE CLASSIFICATION

If the message is written by the store (agent reply), use the same issue_category as the customer's message that triggered the reply unless the reply introduces a completely different topic.

Example:

Customer: "Do you deliver to Dubai Marina?"
Store: "Yes, delivery is available."

Both messages should use:
support | order_status


priority rules:

If broad_category = b2b_sales → high.
If broad_category = supplier_prospect → normal (high only if clearly urgent/large-scale).
If support AND issue_category in (complaint, refund) → high.
If support other → normal.
If marketing → low.
If spam → low.
If other → low.

confidence:
Return a number between 0.0 and 1.0.

Return only the pipe-separated line. No explanation.

Message:
{{1.entry[].messaging[].message.text}}

---

## Prompt: whatsapp_b_system

You convert WhatsApp conversations into structured dataset rows.

The dataset schema is locked and must never change.

Return valid JSON only.
Do not include explanations.
Do not wrap the response in markdown.
The output must start with [ and end with ].


SENDER BLOCK LOGIC

Each sender block must produce one dataset object.

A sender block means consecutive messages from the same sender.

Grouping rules:

• Merge consecutive customer messages into one inbound object.
• Merge consecutive store messages into one outbound object.
• Never merge messages from different senders.
• Never create more dataset objects than sender blocks.


FIELD DEFINITIONS (must always exist)

conversation_id
wa_number
order_id
message_id_external
message_direction
message_source
message_text
broad_category
issue_category
timestamp_utc
agent_name
intent_tag
resolution_status
response_time_seconds
conversation_status
ai_suggested_reply
ai_reply_used
escalation_flag
customer_sentiment
priority
confidence_score
channel
conversation_started_at
conversation_closed_at
conversation_hash
label_source


FIELD RULES

conversation_id = WhatsApp phone number
conversation_hash = WhatsApp phone number
wa_number = WhatsApp phone number

message_id_external must increment sequentially starting from 1 for each sender block.


MESSAGE DIRECTION

message_direction must be:

inbound = customer
outbound = store


MESSAGE SOURCE

message_source must always be:
whatsapp_B


CHANNEL

channel must always be:
whatsapp_B


CONVERSATION STATUS

conversation_status must be one of:

open
waiting_customer
waiting_agent
resolved
escalated
auto_resolved
closed


RESOLUTION STATUS

resolution_status must be one of:

unresolved
resolved
auto_resolved


STATUS DEFINITIONS

unresolved = issue still open
resolved = issue solved by agent
auto_resolved = system handled automatically


CUSTOMER SENTIMENT

customer_sentiment must be one of:

positive
neutral
negative


CONFIDENCE SCORE

confidence_score must be between 0 and 1.

Use:

0.9 to 1.0 when intent is very clear
0.75 to 0.89 when likely but not perfectly explicit
0.5 to 0.74 when ambiguous
below 0.5 only when highly uncertain


TIMESTAMP RULES

timestamp_utc = timestamp of the first message in the sender block.

conversation_started_at = timestamp of the first message in the conversation.

Timestamp values must preserve the exact timestamp provided in the input message.

Do not convert time zones.
Do not change the time value.
Do not infer UTC.

Return the timestamp exactly as received from the conversation.


GREETING HANDLING

Greeting-only messages should not be removed.

Greeting-only messages must be merged with the following message from the same sender if they appear consecutively.


RESPONSE TIME

response_time_seconds must be empty unless it is explicitly calculated from the conversation.

Do not estimate or invent response times.


LABEL SOURCE

label_source must always be:

human_labeled


OUTPUT RULES

Return strictly valid JSON.

Do not include any text before or after the JSON array.


## Prompt: whatsapp_b_user

Process this WhatsApp conversation.

Phone:
{{1.phone}}

Messages:
{{1.messages}}

Operator instruction:
{{1.comments}}


Tasks:

1. Group messages into sender blocks.
   A sender block means one or more consecutive messages from the same sender.

2. Merge the text of messages within each sender block.

3. Extract timestamps from the first message in each sender block.

4. Extract order numbers if present.

5. Classify each sender block.


IMPORTANT STRUCTURE RULE

You must preserve the chronological structure of the conversation.

A new dataset row must start every time the sender changes.

Do NOT merge messages from the same sender if another sender message appears between them.

Each sender block must produce exactly one dataset row.


CONVERSATION STATUS RULES

conversation_status must reflect who is expected to send the next message.

open
Use when the conversation has started but it is not yet clear who is expected to respond next.

waiting_agent
Use when the next action is expected from the agent.

Examples:
• the customer asked a question
• the customer provided information (such as address or order details)
• the agent said they will process the order, check availability, or reply later


Also use waiting_agent when the agent explicitly says they will process the order or reply later.
waiting_customer
Use when the next action is expected from the customer.

Examples:
• the agent asked the customer a question
• the agent requested additional information
• the agent provided instructions and is waiting for the customer to proceed

closed
Use when the issue or request is clearly finished and no further action is expected from either side.

This can happen when:
• both sides end the conversation with thanks
• the agent provides final information and the customer acknowledges it

CUSTOMER ACKNOWLEDGEMENT RULE

If the agent has already provided the final information
(for example order confirmation, delivery schedule, or answer to a question)
and the customer replies only with acknowledgement phrases such as:

OK
Thanks
Thank you
Great thanks
👍
👌

then classify the message as:

issue_category = other
conversation_status = closed
resolution_status = resolved


RESOLUTION STATUS RULES

resolved
Use when the customer’s request or question has been clearly answered or completed.

unresolved
Use when further action or follow-up is still expected from either side.

If conversation_status = closed then resolution_status must be resolved.



BROAD CATEGORIES

support
supplier_prospect
b2b_sales
marketing
spam
other


b2b_sales

B2B buyer inquiry where a BUSINESS wants to purchase physical products from us for commercial use, resale, or supply.

Examples:
• restaurant asking for product supply or price list
• hotel asking to buy products
• grocery store asking wholesale price
• distributor asking MOQ or partnership
• catering company requesting samples
• retailer asking to stock our products

These inquiries typically come from businesses such as restaurants, cafes, hotels, catering companies, grocery stores, distributors, or retailers.

IMPORTANT DISTINCTION

Retail customers asking about delivery, placing an order, confirming an address, or buying products for personal consumption must always be classified as:

support

Only classify as b2b_sales when the sender clearly represents a business purchasing products for commercial use.



ISSUE CATEGORIES

order_status
order_details
order_modification
delivery_time
delivery_area
payment
product_question
complaint
refund
cancellation
address_change
promotion
agent_reply
other


ISSUE CATEGORY DEFINITIONS

order_status
Customer asking about order progress or delivery status.

Examples:
• Where is my order?
• When will my order arrive?
• Has my order been shipped?


order_details
Operational coordination of an order after it is placed.

Examples:
• confirming delivery time
• confirming delivery address
• order total confirmation
• rider arrival message
• delivery instructions
• customer confirming availability for delivery

order_details also includes questions about:
• minimum order amount
• delivery area
• delivery requirements
• ordering conditions

order_modification
Customer asking to change an existing order after it has already been placed.

Examples:
• Can I add one more item?
• Remove the cake from my order
• Change the quantity
• Replace the product with another one

ORDER MODIFICATION RULE

If the customer asks to modify the contents of an existing order
(add/remove/change products or quantities),
classify as:

order_modification

If the customer asks only to change the delivery address,
classify as:

address_change


delivery_time
Specific discussion about delivery schedule or time window.

Examples:
• Can you deliver at 6 PM?
• I will be home at 1 PM
• Please deliver tomorrow morning

Use delivery_time only when the conversation is primarily about choosing or negotiating a delivery time.

If the time is simply being confirmed as part of order coordination, use order_details instead.


delivery_area
Customer asking whether delivery is available in a specific area or location.

Examples:
• Do you deliver to Dubai Marina?
• Do you deliver to JVC?
• Is delivery available in Sharjah?
• Can you deliver to Al Ain?

These questions relate to delivery coverage rather than a specific order.

ADDRESS MESSAGE RULE

If a customer message contains an address but does NOT explicitly request a change, do NOT classify it as address_change.

If the address is being confirmed or provided for delivery, classify the issue as order_details.

Only classify address_change when the customer explicitly asks to change the delivery address.

Examples:
• Change my address
• Send to another address
• My address changed


AGENT_REPLY SENDER RULE

issue_category = agent_reply can ONLY be used when the sender is the agent.

If the sender is the customer, never use agent_reply,
even if the message only contains acknowledgements such as:

OK
Thanks
Thank you
👍
😊

Customer acknowledgements must be classified as:

issue_category = other

AGENT_REPLY RULE

Use issue_category = agent_reply ONLY when:

1. The sender is the agent
2. The message contains only acknowledgement or politeness
3. The message contains NO operational information
4. The message does not ask a question


Correct examples:

Agent: OK
→ agent_reply

Agent: Thank you 👍
→ agent_reply

Agent: You're welcome
→ agent_reply


Customer: OK thank you
→ other

Customer: Thanks
→ other

Agent replies must contain ONLY phrases such as:

Ok
Sure
Thanks
Thank you
You're welcome
👍
😊
👌

These messages acknowledge the conversation but do not add any new information.

If the message contains ANY operational information,
it must NOT be classified as agent_reply.

Operational information includes:

• delivery timing
• delivery confirmation
• delivery instructions
• product availability
• payment methods
• order confirmation
• order coordination
• requests for information
• links to the website
• asking questions about the order

If ANY of the above appear, classify using the appropriate category,
most commonly:

order_details
product_question
delivery_time
payment

Mixed messages rule:

If a message contains politeness AND operational information,
ignore the politeness and classify based on the operational content.

Example:

"Sure, we will deliver tomorrow at 1 PM."

Correct classification:
delivery_time

NOT agent_reply.

CUSTOMER SENTIMENT RULE

positive
Customer expresses satisfaction, thanks, or positive tone.

neutral
Neutral informational messages or simple questions.

negative
Customer expresses dissatisfaction, complaint, frustration, or problem.

Examples:

"Thanks a lot" → positive  
"Where is my order?" → neutral  
"This delivery is very late" → negative

PRIORITY RULES

b2b_sales = high
supplier_prospect = normal
support complaint or refund = high
support other = normal
marketing = low
spam = low
other = low


Return the structured dataset rows as JSON.
