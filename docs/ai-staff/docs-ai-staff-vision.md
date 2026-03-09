
# AI Staff – Scope & Vision

**Created:** 2026-02-21  
**Project:** AI Digital Operations & Customer Support Assistant

---

# 1. Vision Statement

AI Staff is a **semi-autonomous digital operations assistant** designed to support customer communication, operational monitoring, stock awareness, supplier coordination, and digital store management.

The long‑term objective is to operate **24/7 with human supervision for edge cases and strategic decisions**.

---

# 2. Core Principles

- Semi‑autonomous, **not uncontrolled**
- Every AI decision **must be logged**
- **Confidence score required** for each action
- **Escalation always available** for low‑confidence cases
- **No financial or refund decisions** without human approval
- Gradual capability expansion with **clear governance**

---

# 3. Capability Roadmap

## Phase 1 – Communication Assistant (Months 1–2)

- Email classification and routing
- WhatsApp message classification
- Auto‑response using predefined templates
- Telegram escalation for complex cases
- Communication dataset logging

## Phase 2 – Stock Awareness (Months 3–4)

- Shopify API integration
- Real‑time stock availability checks
- Detection of website vs warehouse stock mismatch
- Out‑of‑stock alert automation

## Phase 3 – Supply Chain Assistant (Months 4–6)

- Dropshipping supplier inquiry automation
- Supplier response tracking
- Escalation of delays or stock shortages

## Phase 4 – Monitoring Agent (Months 6–9)

- Website product monitoring
- Broken page detection
- Missing image detection
- Weekly operational health reports

## Phase 5 – Content & Growth Assistant (Months 9–12)

- Drafting new product pages
- SEO suggestion automation
- Catalog consistency analysis

---

# 4. Decision Control Model

All AI decisions must return structured output:

```json
{
  "intent": "",
  "action": "",
  "confidence": 0-100,
  "reason": ""
}
```

Rules:

- Confidence below **70 → escalate to Telegram**
- **Refund‑related emails → always escalate**
- **Supplier pricing decisions → always escalate**
- **Legal or complaint escalation → notify immediately**

---

# 5. Governance & Risk Control

AI must **NOT autonomously**:

- Approve refunds
- Offer compensation
- Modify pricing
- Commit to supplier financial agreements

Human oversight is required for **financial, legal, or brand‑impacting decisions**.

---

# 6. End‑State Objective

AI Staff becomes a **24/7 semi‑autonomous digital assistant** capable of:

- Managing customer communication
- Monitoring operations
- Assisting supply chain decisions
- Supporting digital growth

All while remaining under **human strategic supervision**.

---

# 7. Learning & Data Architecture (Airtable Knowledge Engine)

## Executive Summary

AI Staff improves through **structured conversation logging stored in Airtable**.

Every customer interaction (WhatsApp, Telegram reply, email, Instagram) is stored as a structured record.

This dataset becomes the foundation for:

- Intent detection training
- Response quality improvement
- SLA tracking
- Escalation pattern analysis
- Future AI fine‑tuning

Airtable functions as:

**Operational Memory + Training Dataset + Audit Log**

---

# 8. Learning Model Philosophy

AI Staff does **NOT learn autonomously**.

Instead:

1. Conversations are logged
2. Humans respond
3. Patterns are structured
4. Dataset improves
5. AI suggestions become smarter

This approach follows **supervised learning**, preventing uncontrolled AI drift.

---

# 9. Conversation Logging Architecture

All inbound and outbound communication is written to:

```
Airtable Base: AI Staff – Conversation Engine
Table: conversation_log
```

Each message is stored as **one row**.

Inbound and outbound records are stored separately to preserve full conversation chronology.

---

# 10. Core Data Structure (Training Taxonomy)

## Identification Layer

| Field | Purpose |
|------|------|
| conversation_id | Unique internal identifier |
| message_id_external | Platform message ID |
| conversation_hash | Normalized phone/email used to group threads |
| channel | whatsapp_A / whatsapp_B / email / instagram |
| message_source | Source platform |
| message_direction | inbound / outbound |
| from_email | Sender email address (email channel) |

---

## Content Layer

| Field | Purpose |
|------|------|
| message_text | Raw message text |
| subject | Email subject |
| timestamp_utc | Standardized timestamp |
| agent_name | Staff member name |

---

## Classification Layer (AI Controlled)

| Field | Type | Purpose |
|------|------|------|
| broad_category | single select | High‑level classification |
| issue_category | single select | Support subtype |
| priority | single select | low / normal / high |
| needs_human | boolean | Escalation trigger |
| confidence_score | decimal | AI confidence score |

### broad_category values

support  
b2b_sales  
supplier_prospect  
marketing  
other

### issue_category values

order_status  
payment  
product_question  
complaint  
refund  
cancellation  
address_change  
promotion  
agent_reply  
other

---

## Operational Lifecycle Layer

| Field | Purpose |
|------|------|
| conversation_status | open / resolved / reopened |
| resolution_status | unresolved / resolved |
| inbound_outbound | inbound / outbound |
| sla_status | response tracking |

---

## AI Control & Audit Layer

| Field | Purpose |
|------|------|
| classification_result_raw | raw AI output |
| ai_confidence_score | AI confidence |
| ai_escalated | escalation flag |
| ai_action_recommended | suggested response type |

---

# Structural Philosophy

The dataset serves as:

- **Operational memory**
- **Training dataset**
- **Governance log**
- **Escalation audit trail**

Every AI classification is logged.  
Every human response is logged.  
Every escalation is traceable.

AI Staff improves through **structured supervision, not autonomous drift**.
