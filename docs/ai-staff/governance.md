
# AI Staff – Governance Model

This document defines the governance and safety model for the AI Staff system.

The goal is to ensure automation improves operational efficiency while maintaining **human control over critical business decisions**.

---

# Core Governance Principles

1. **AI is semi-autonomous, not independent.**
2. **Every AI classification must be logged.**
3. **Confidence scores must accompany all AI outputs.**
4. **Humans must remain in control of financial, legal, or supplier commitments.**
5. **AI acts as an operational assistant, not a decision-maker.**

---

# AI Classification Output Contract

All AI classification modules must return a structured output using the following pipe-separated format:

```
broad_category|issue_category|priority|confidence|customer_email|supplier_region
```

This format is used by the automation system to:

- route messages
- trigger alerts
- populate Airtable records
- build the AI training dataset.

---

# Field Definitions

## broad_category

High-level classification of the message.

Possible values:

support  
b2b_sales  
supplier_prospect  
marketing  
spam  
other

---

## issue_category

Subtype of support inquiries.

Used only when `broad_category = support`.

Possible values:

order_status  
payment  
product_question  
complaint  
refund  
cancellation  
address_change  
other

If the message is not classified as **support**, this value must return:

```
null
```

---

## priority

Indicates operational urgency.

Possible values:

low  
normal  
high

Typical rules:

- **complaint / refund → high**
- **b2b_sales → high**
- **supplier_prospect → normal**
- **marketing / spam → low**

---

## confidence

AI confidence score between:

```
0.0 – 1.0
```

Example:

```
0.83
```

Confidence reflects the model’s probability estimate that the classification is correct.

Confidence is recorded for **analysis and monitoring**, but does not solely determine escalation.

---

## customer_email

Used primarily for email scenarios.

If the sender email is embedded inside the message body (for example Shopify contact forms), the AI should extract and return the actual customer email.

If none exists:

```
null
```

---

## supplier_region

Used only when:

```
broad_category = supplier_prospect
```

Possible values:

priority_region  
other_region  
unknown  

Priority region includes suppliers located in:

Russia  
Armenia  
Georgia  
Uzbekistan  
Kazakhstan  
Belarus  

or UAE-based suppliers producing CIS-style products.

If the message is not a supplier inquiry:

```
null
```

---

# Escalation Model

AI does not directly respond to customers automatically.

Instead, the automation system determines when **human attention is required**.

Messages escalate to the internal support team when:

```
broad_category = support
broad_category = b2b_sales
broad_category = supplier_prospect
```

Escalation channel:

**Telegram internal support group**

Messages classified as:

```
marketing
spam
other
```

are logged but do not trigger alerts.

---

# Actions AI Must NOT Perform

AI is prohibited from automatically:

- Approving refunds
- Offering compensation
- Modifying product pricing
- Committing to supplier financial agreements
- Issuing legal commitments

These actions require explicit human approval.

---

# Logging & Audit Requirements

All interactions must be stored in:

```
Airtable Base: AI Staff – Conversation Engine
Table: conversation_log
```

Each message is stored as a separate record.

The dataset provides:

- operational history
- AI training data
- classification audit trail
- escalation monitoring

Every AI classification is logged.

Every human reply is logged.

This ensures complete traceability of AI-assisted decisions.

---

# Human Oversight Model

AI supports operations but does not replace human supervision.

Humans remain responsible for:

- financial decisions
- supplier agreements
- customer dispute resolution
- strategic communications.

AI’s role is limited to:

- message classification
- operational routing
- support intelligence
- dataset generation.

---

# Continuous Improvement Model

AI Staff improves through **supervised learning**.

Process:

1. Conversations are logged
2. Humans respond to complex cases
3. Patterns are structured
4. Dataset quality improves
5. AI classification accuracy increases

AI does not retrain itself automatically.

Learning is controlled through **dataset improvements and prompt refinement**.

---

# Governance Objective

Maintain a balance between:

- automation efficiency
- operational safety
- business accountability.

AI Staff acts as a **semi-autonomous operational assistant** designed to improve communication handling while keeping **humans in control of critical decisions**.
