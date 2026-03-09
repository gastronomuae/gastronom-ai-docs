# Gastronom AI Support System

This repository documents the **AI-powered customer support automation system** used for the Gastronom online store.

The system combines **Make.com automation, AI message classification, and a structured conversation dataset** to support customer communication across multiple channels.

Supported channels:

- WhatsApp
- Instagram Direct Messages
- Email

The long-term goal is to operate a **semi-autonomous AI support assistant** capable of handling customer conversations, assisting operations, and escalating complex issues to human operators.

---

# System Overview

Customer messages from different platforms are processed through automation pipelines that classify, log, and route conversations.

High-level flow:

```
Customer Messages
      ↓
Communication Channel
(WhatsApp / Instagram / Email)
      ↓
Make.com Automation
      ↓
AI Classification
(OpenAI)
      ↓
Routing Decision
├─ Auto Response
└─ Telegram Escalation
      ↓
Routing Decision
├─ Auto Response
└─ Telegram Escalation
```

This architecture allows the system to:

- classify incoming messages
- automate common responses
- escalate complex cases
- build a structured dataset for AI improvement

---

## Repository Structure

```
docs
├─ ai-dataset
│  ├─ index.md
│  ├─ overview.md
│  ├─ system-architecture.md
│  ├─ dataset-schema.md
│  ├─ dataset-labeling-guidelines.md
│  ├─ prompt-specification.md
│  └─ make-automation.md
│
├─ ai-staff
│  ├─ ai-staff-vision.md
│  ├─ roadmap.md
│  └─ governance.md
│
└─ scenarios
   ├─ 01-shopify-whatsapp-order-automation.md
   ├─ 02-morning-whatsapp-sender.md
   ├─ 03-whatsapp-number-a-telegram-routing.md
   ├─ 04-telegram-whatsapp-reply-routing.md
   ├─ 05-email-ai-classification-router.md
   └─ 06-instagram-dm-watcher.md
```
---

# Documentation Sections

## AI Staff

Defines the **long-term vision and governance model** of the AI assistant.

Location:
docs/ai-staff/


Documents:

- Vision of the AI assistant
- Capability roadmap
- Governance and safety rules

---

## AI Dataset

Defines the **conversation dataset architecture** used to train and improve AI message classification.

Location:
docs/ai-dataset/



Topics covered:

- dataset schema
- classification taxonomy
- labeling guidelines
- prompt specification
- automation pipeline

The dataset is stored in **Airtable** and populated through automation scenarios.

---

## Automation Scenarios

Technical documentation of each **Make.com automation scenario**.

Location:
docs/scenarios/


Each scenario describes:

- automation purpose
- trigger
- processing logic
- routing behavior
- dataset interactions

---

# Key Technologies

| Technology | Role |
|------|------|
| Make.com | automation orchestration |
| OpenAI API | message classification |
| Airtable | conversation dataset storage |
| WhatsApp Cloud API | customer messaging |
| Instagram API | DM monitoring |
| Telegram Bot | internal escalation channel |
| Shopify | order data and store backend |

---

# Design Principles

The AI system follows several core principles:

**Semi-Autonomous Operation**

The AI assists operations but does not operate without oversight.

**Traceable Decisions**

All AI actions are logged with confidence scores.

**Human Escalation**

Low-confidence or sensitive cases escalate to human operators.

**Structured Learning**

The system improves through a structured dataset of past conversations.

---

# Long-Term Objective

The long-term objective is to build a **24/7 AI operations assistant** capable of supporting:

- customer communication
- order inquiries
- operational monitoring
- supplier coordination
- catalog growth

while maintaining **human governance and decision control**.

---

# Status

Current focus:

Phase 1 – Communication Automation

Implemented:

- WhatsApp routing
- Telegram escalation
- Email classification
- Instagram DM capture
- Airtable conversation dataset

Future phases will expand into:

- operational monitoring
- supply chain awareness
- AI-assisted content generation

  


