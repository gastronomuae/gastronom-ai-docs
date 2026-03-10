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
