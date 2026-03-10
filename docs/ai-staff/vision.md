# AI Staff – Scope & Vision

**Created:** 2026-02-21  
**Project:** AI Digital Operations & Customer Support Assistant

---

# 1. Vision Statement

AI Staff is a **semi-autonomous digital operations assistant** designed to support customer communication, operational monitoring, stock awareness, supplier coordination, and digital store management.

The long-term objective is to operate **24/7 with human supervision for edge cases, commercial decisions, and strategic actions**.

AI Staff is not intended to replace humans.  
It is intended to:

- classify incoming communication
- draft helpful responses
- escalate sensitive cases
- reduce repetitive manual work
- improve over time through structured feedback and knowledge updates

---

# 2. Core Principles

- Semi-autonomous, **not uncontrolled**
- Every AI decision **must be logged**
- **Confidence score is required** for AI-generated classifications
- **Escalation must always be available**
- **No financial, refund, or legal decisions** without human approval
- Capability expansion must be **gradual and governed**
- AI should **draft first, automate later**
- Knowledge must be **versioned and traceable**

---

# 3. Capability Roadmap

## Phase 1 – Communication Assistant (Months 1–2)

- Email classification and routing
- Instagram message classification
- Structured WhatsApp dataset creation
- Airtable logging of communication data
- Telegram escalation for selected categories
- AI-drafted reply suggestions for human review
- Manual sending by support team

### Phase 1 operating model

At this stage, AI does **not automatically send most replies**.

The workflow is:

1. Message is received
2. AI classifies it
3. AI may draft a suggested reply
4. Human reviews / edits / sends
5. Conversation is logged for learning

This phase is designed to create a **safe human-in-the-loop support model**.

---

## Phase 2 – Assisted Knowledge-Based Support (Months 2–4)

- GitHub Markdown knowledge base integration
- AI-generated replies grounded in approved knowledge
- Safer handling of repeated customer questions
- Initial automation for low-risk, knowledge-stable topics

Typical low-risk topics may include:

- delivery coverage
- payment method questions
- store availability / working status
- product availability when data is reliable

---

## Phase 3 – Stock & Order Awareness (Months 3–5)

- Shopify API integration
- Real-time stock checks
- Product detail retrieval
- Order lookup support
- Improved support replies based on live operational data

Examples:

- checking whether a product is in stock
- checking order status
- retrieving basic delivery/order details
- confirming payment and fulfillment state

---

## Phase 4 – Supply Chain Assistant (Months 4–6)

- Supplier inquiry classification
- Supplier priority region detection
- Supplier response tracking
- Escalation of relevant supplier opportunities
- Structured follow-up support for sourcing workflows

---

## Phase 5 – Monitoring & Growth Assistant (Months 6–12)

- Website monitoring
- Broken page detection
- Missing image detection
- Drafting product or content suggestions
- SEO support automation
- Catalog consistency analysis

---

# 4. Decision Control Model

AI decisions in the current system are **classification-driven** and scenario-specific.

AI outputs are not abstract freeform reasoning objects.  
They are structured fields used by Make.com, Airtable, Telegram, and future support workflows.

## Current structured outputs

Depending on the scenario, AI returns structured values such as:

### Email classification
```text
broad_category|issue_category|priority|confidence|customer_email|supplier_region|escalation_flag
```

Instagram classification
```text
broad_category|issue_category|priority|confidence|escalation_flag
```

WhatsApp historical dataset generation

Structured JSON rows following the dataset schema.

Confidence model

Confidence is stored as a decimal between:
```text
0.0 – 1.0
```
Example:
```text
0.92
```
Confidence is used as a quality signal, but escalation is not determined by confidence alone.

Escalation model

Escalation is determined by scenario rules, which may include:

complaint

refund

cancellation

b2b_sales

supplier_prospect

explicit request for human support

other business-defined manual handling cases

Confidence may support escalation decisions, but the system is not governed by a single threshold alone.

# 5. Governance & Risk Control

AI must NOT autonomously:

Approve refunds

Offer compensation

Modify pricing

Commit to supplier financial agreements

Make legal commitments

Confirm sensitive order decisions without verified data

Send autonomous replies on high-risk topics unless explicitly approved by governance

Human oversight is required for:

complaints

refund-related requests

commercial negotiations

supplier relationship decisions

legal / reputational issues

ambiguous or sensitive customer situations

# 6. Human-in-the-Loop Operating Model

AI Staff follows a draft-first, automate-later strategy.

Stage 1 – Drafting mode

AI:

classifies the message

suggests a reply

flags cases needing escalation

Human:

reviews the reply

edits if needed

sends the final answer

Stage 2 – Controlled automation

Once a topic is proven safe and stable, AI may automatically respond to selected categories.

Examples of possible future safe topics:

delivery area availability

simple payment method questions

working status / service availability

basic product availability questions

Stage 3 – Broader operational support

As knowledge and APIs improve, AI may assist with:

order lookup

stock lookup

product detail retrieval

more precise customer support drafting

Automation must expand only after measured reliability.

# 7. End-State Objective

AI Staff becomes a 24/7 semi-autonomous digital assistant capable of:

managing first-line customer communication

drafting replies grounded in business knowledge

escalating cases requiring human review

monitoring operations

assisting supply chain decisions

supporting digital growth

All while remaining under human strategic and operational supervision.

# 8. Learning & Knowledge Architecture

AI Staff improves through structured knowledge + structured feedback, not through uncontrolled autonomous learning.

Its intelligence evolves from multiple governed data sources.

8.1 Airtable Conversation Engine

Airtable stores structured communication logs and acts as:

operational memory

training dataset

audit log

escalation history

Every relevant interaction becomes structured data.

8.2 GitHub Markdown Knowledge Base

Approved business knowledge is maintained in Markdown documents stored in GitHub.

This knowledge base may include:

delivery rules

payment methods

support playbooks

refund/cancellation policy summaries

supplier handling notes

store and service information

product and category guidance

GitHub is the preferred knowledge source because it is:

version-controlled

reviewable

easy to update

safe for AI retrieval

suitable for future automated updates

8.3 Future API-Based Knowledge

As the system matures, AI may also use live operational sources such as:

Shopify order data

Shopify inventory / stock data

product database details

future operational APIs

This enables grounded replies based on real-time data rather than prompt memory.

# 9. Learning Model Philosophy

AI Staff does NOT learn autonomously.

Instead, learning happens through supervised improvement:

Conversations are logged

Humans respond or approve replies

Good responses become reference patterns

Knowledge base is updated when needed

Prompt rules are refined

AI suggestions become smarter over time

This prevents uncontrolled AI drift.

# 10. How the Agent Improves Over Time

AI Staff improves through three main feedback loops.

10.1 Dataset Improvement Loop

Customer messages and structured labels in Airtable improve:

category recognition

escalation quality

support analytics

simulation quality

10.2 Knowledge Base Improvement Loop

When AI gives a weak or incomplete draft, the underlying knowledge can be improved in GitHub.

Examples:

update delivery coverage guidance

clarify payment methods

add policy wording

document better support phrasing

add new FAQ-like answer patterns

Once the knowledge base is updated, future replies improve without changing the core architecture.

10.3 Human Feedback Loop

Human replies and corrections help improve:

prompt instructions

escalation rules

support tone

safe automation boundaries

Over time, AI becomes more accurate because it is exposed to better structured guidance.

# 11. Conversation Logging Architecture

All inbound and outbound communication is written to:
```text
Airtable Base: AI Staff – Conversation Engine
Table: conversation_log
```
Each message or sender block is stored as a separate structured record depending on the channel workflow.

Inbound and outbound records are stored separately to preserve chronology and auditability.

# 12. Core Data Structure (Training Taxonomy)

## Identification Layer

| Field | Purpose |
|------|------|
| conversation_id | Unique internal identifier |
| message_id_external | Platform message ID |
| conversation_hash | Normalized phone/email/thread identifier used to group conversations |
| channel | Source channel (whatsapp_A / whatsapp_B / email / instagram) |
| message_source | Source platform |
| message_direction | inbound / outbound |
| from_email | Sender email address (email channel) |

---

## Content Layer

| Field | Purpose |
|------|------|
| message_text | Raw message text |
| subject | Email subject (email channel only) |
| timestamp_utc | Standardized timestamp |
| agent_name | Staff member name who responded |

---

## Classification Layer (AI Controlled)

| Field | Type | Purpose |
|------|------|------|
| broad_category | single select | High-level classification of message intent |
| issue_category | single select | Support subtype classification |
| priority | single select | Message urgency level |
| escalation_flag | boolean | Indicates message should be handled by human |
| confidence_score | decimal | AI classification confidence (0.0–1.0) |

### broad_category values

- support  
- b2b_sales  
- supplier_prospect  
- marketing  
- spam  
- other  

### issue_category values

- order_status  
- order_details  
- order_modification  
- delivery_time  
- delivery_area  
- payment  
- product_question  
- complaint  
- refund  
- cancellation  
- address_change  
- promotion  
- agent_reply  
- other  

---

## Operational Lifecycle Layer

| Field | Purpose |
|------|------|
| conversation_status | open / waiting_customer / waiting_agent / escalated / auto_resolved / closed |
| resolution_status | unresolved / resolved / auto_resolved |
| sla_status | Response time tracking |

---

## AI Control & Audit Layer

| Field | Purpose |
|------|------|
| classification_result_raw | Raw AI output stored for debugging |
| ai_confidence_score | AI confidence score |
| escalation_flag | Human escalation signal |
| ai_suggested_reply | AI-generated reply draft |
| ai_reply_used | Whether the AI reply was used by staff |
| label_source | ai_prediction / human_labeled / simulation / system_default |

---

# 13. Structural Philosophy

The dataset serves multiple operational purposes:

- **Operational memory** of customer interactions  
- **Training dataset** for improving AI classification and responses  
- **Governance log** ensuring traceability of AI decisions  
- **Escalation audit trail** for support management  
- **Reply quality improvement source** through human corrections  
- **Knowledge improvement signal** for updating the GitHub knowledge base  

### Governance Principles

Every AI classification must be logged.

Every human response or approval can be logged.

Every escalation must remain traceable.

Knowledge improvements must be version-controlled.

### Learning Model

AI Staff improves through:

1. Structured dataset growth (Airtable)
2. Human-reviewed responses
3. Knowledge base updates (GitHub Markdown)
4. Prompt refinement
5. Controlled automation expansion

AI Staff evolves through **structured supervision, curated knowledge, and controlled automation**, not through uncontrolled autonomous learning.
