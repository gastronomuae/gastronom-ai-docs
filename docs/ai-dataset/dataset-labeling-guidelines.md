# Dataset Labeling Guidelines

This document defines how conversations are labeled in the AI dataset.

The goal is to ensure **consistent classification across all channels**, including:

- WhatsApp
- Instagram
- Email

AI performs the initial classification and labeling.  
Humans review dataset quality and correct errors when necessary.

All labeling must follow the rules defined in this document.

---

# Core Principle

Each message or sender block must be classified according to the **primary operational intent**.

If multiple topics appear in the same message, classification must reflect the **main action required from the store**.

Example:

Customer message:

"Hi, where is my order?"

Classification:

broad_category = support  
issue_category = order_status

The greeting is ignored because the operational intent is **order tracking**.

---

# Broad Category Definitions

Each message must first be assigned a **broad_category**.

Allowed values:

support  
supplier_prospect  
b2b_sales  
marketing  
spam  
other

---

# Support

support applies to **retail customer interactions related to store operations**.

Examples:

- product availability
- delivery questions
- order tracking
- payment questions
- complaints
- refunds
- store information

Typical examples:

"Do you have Borjomi?"

"Where is my order?"

"Do you deliver to Dubai Marina?"

Retail customers purchasing products for **personal use** must always be classified as:

broad_category = support

---
# Delivery Area vs Order Status

issue_category = delivery_area

Examples:

Do you deliver to Dubai Marina?

Do you deliver to JVC?

Is delivery available in Sharjah?

Can you deliver to Al Ain?

These questions relate to service coverage, not the status of an existing order.

Do NOT classify these messages as:

order_status

because the customer is not asking about an existing order.

---
# Order Modification Rule

Requests to change an existing order after it has been placed must be classified as:

issue_category = order_modification

Examples:

Add one more item

Remove one product

Change the quantity

Replace an item

If the customer only requests to change the delivery address, classify as:

address_change

---

# B2B Sales

b2b_sales applies **only when a business wants to purchase physical products from us**.

Typical examples include:

- restaurants
- cafés
- hotels
- catering companies
- grocery stores
- distributors
- retailers
- resellers

Examples:

"Please send your wholesale price list."

"We are a restaurant interested in purchasing your products."

"What is your MOQ?"

Priority rule:

b2b_sales → high priority


---

# Address Message Rule 

If a customer message contains an address but does not explicitly request a change, it must NOT be classified as address_change.

Examples:

Customer message:

"My address is Dubai Marina Tower A."

Classification:

issue_category = order_details

Only classify as address_change if the customer explicitly requests:

change address

send to another address

update delivery location

---

# IMPORTANT DISTINCTION  
## B2B vs Marketing

Many messages appear similar to B2B inquiries but are actually **marketing proposals**.

If the sender offers to:

- generate customers
- generate traffic
- promote our store
- provide affiliate programs
- run advertising campaigns
- provide SEO services
- promote products in exchange for commission

then the message must be classified as:

broad_category = marketing


Examples:

"We can bring more customers to your store."

"Join our affiliate platform."

"We can promote your products through influencers."

These messages **do not involve purchasing products from us**.

---

# Customer Acknowledgement Rule

When the customer replies with a short acknowledgement after receiving the final answer:

Examples:

OK
Thanks
Thank you
👍

The message must be classified as:

issue_category = other

This prevents incorrectly labeling customer acknowledgements as agent_reply.

---

#  Outbound Message Classification

When the store sends a reply, the message should normally inherit the issue_category of the customer's message that triggered the reply.

Example:

Customer:
"Do you deliver to Dubai Marina?"

Agent:
"Yes, delivery is available."

Both messages should use:

issue_category = delivery_area

Unless the agent introduces a new topic.

---

#  Mixed Message Rule

If a message contains politeness + operational information, classification must be based on the operational content.

Example:

Message:

"Sure, we will deliver tomorrow at 1 PM."

Correct classification:

issue_category = delivery_time

NOT:

agent_reply

---

# Sender Role Rule

agent_reply can only be used when the sender is the store agent.

If the sender is the customer, acknowledgements such as:

OK
Thanks
👍

must be classified as:

issue_category = other

---
# Conversation Closure Rule

If the agent provides the final information and the customer replies only with acknowledgement, the conversation should be considered complete.

Example:

Agent:
"Delivery will arrive at 6 PM."

Customer:
"Thanks."

Classification:

conversation_status = closed
resolution_status = resolved

---

# Sender Block Rule

Dataset rows represent sender blocks, not individual messages.

A sender block is defined as:

One or more consecutive messages from the same sender.

Rules:

consecutive customer messages are merged into one inbound row

consecutive store messages are merged into one outbound row

a new row is created when the sender changes

---

# Timestamp Integrity

Timestamps must always match the original message timestamp.

The system must not:

convert time zones

modify timestamps

infer missing time values

---

# Supplier Prospect

supplier_prospect applies to **producers or exporters offering physical goods supply**.

Typical examples include:

- manufacturers
- food producers
- exporters
- brand owners
- agricultural suppliers
- wholesalers offering supply

Examples:

"We are a producer of frozen dumplings."

"Please find our catalog attached."

"We export dairy products."

"We would like to distribute our brand in the UAE."

These messages indicate **someone wanting to supply products to our store**.

Priority rule:

supplier_prospect → normal priority



Priority may be elevated if the supplier operates in a priority region or offers highly relevant products.

---

# Supplier Region Classification

When a message is classified as:

supplier_prospect

the system must also classify:


supplier_region


Allowed values:

priority_region  
other_region  
unknown

---

## Priority Region

The following countries belong to the **priority sourcing region**:

- Russia
- Armenia
- Georgia
- Uzbekistan
- Kazakhstan
- Belarus

Suppliers from these countries must be classified as:

supplier_region = priority_region


---

## CIS-style Products Produced in UAE

Suppliers located in the UAE may also be classified as priority region if they produce **CIS-style foods**.

Examples include:

- pelmeni
- vareniki
- manty
- Eastern dumplings
- Russian bakery products
- CIS traditional foods

These suppliers should also be labeled:

supplier_region = priority_region


---

## Other Region

Suppliers clearly located outside the priority region must be classified as:


supplier_region = other_region


Examples:

- Vietnam exporters
- Indian manufacturers
- Chinese wholesalers
- unrelated global exporters

---

## Unknown Region

If the supplier location cannot be determined:


supplier_region = unknown


---

# Language Handling

Supplier country names may appear in multiple languages or grammatical forms.

Examples:

Россия  
России  
Армения  
Armenia  
Georgia  
Грузия  

The classification system must interpret these variations correctly.

---

# Issue Category Assignment

issue_category should be assigned **only when**:


broad_category = support


Otherwise the field must remain empty.

Issue category definitions are maintained in:


docs/ai-dataset/taxonomy.md


---

# Conversation Context Rule

Messages belonging to the same conversation should normally keep the same **issue_category**.

Do not change issue_category unless the customer clearly introduces a new topic.

Example:

Customer:
"Do you deliver to Dubai Marina?"

Agent:
"Yes we do."

Both messages should use:


issue_category = delivery_area


---

# Greeting Handling

Greeting-only messages should be merged with the following message when consecutive.

Examples:

Hi  
Hello  
Здравствуйте

If the greeting contains a question, ignore the greeting and classify based on the question.

---

# Agent Reply Rule

Agent replies that contain **only acknowledgement or politeness** must be classified as:


issue_category = agent_reply


Examples:

OK  
Sure  
Thanks  
You're welcome  
👍

If the message contains **any operational information**, it must NOT be classified as agent_reply.

Examples of operational information:

- delivery timing
- order confirmation
- payment instructions
- product availability
- asking questions

---

# Customer Sentiment Guidelines

positive

Customer expresses satisfaction or gratitude.

Examples:

"Thank you so much."

---

neutral

Customer asks a question or provides information without emotional tone.

Examples:

"Do you deliver to JVC?"

---

negative

Customer expresses dissatisfaction or frustration.

Examples:

"This delivery is very late."

---

# Priority Rules

b2b_sales → high  

supplier_prospect → normal  

support complaint or refund → high  

support other → normal  

marketing → low  

spam → low  

other → low

---

# Confidence Score

confidence_score must be between **0 and 1**.

General guidance:

0.90 – 1.00  
Intent extremely clear

0.75 – 0.89  
Very likely correct

0.50 – 0.74  
Ambiguous message

Values below 0.50 should be used only when intent is highly uncertain.

---

# Dataset Governance

The dataset is designed to support **continuous improvement of AI support systems**.

Principles:

- AI generates initial labels
- humans review difficult cases
- taxonomy must remain consistent
- historical labels should not be retroactively modified without clear reason
