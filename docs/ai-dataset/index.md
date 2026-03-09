# AI Dataset Documentation

This section documents the **conversation dataset architecture** used by the Gastronom AI Customer Support system.

The dataset powers the AI assistant responsible for handling customer communication across multiple channels including:

- WhatsApp
- Instagram Direct Messages
- Email

All conversations are logged, structured, and classified to support:

- AI message classification
- Response automation
- Escalation logic
- Dataset improvement and supervised learning

The dataset is stored in **Airtable** and populated through **Make.com automation scenarios**.

---

# Documentation

## Overview
High-level explanation of the dataset purpose and how it supports the AI support assistant.

- [Overview](overview.md)

## System Architecture
Explains how messages flow from communication channels through automation pipelines into the dataset.

- [System Architecture](system-architecture.md)

## Dataset Structure

Detailed description of the fields used to store conversations and AI classification results.

- [Dataset Schema](dataset-schema.md)

## Labeling & Classification

Guidelines used to label messages for training and improving AI classification.

- [Dataset Labeling Guidelines](dataset-labeling-guidelines.md)

## Automation Implementation

Documentation of the Make.com scenarios that populate and maintain the dataset.

- [Make Automation](make-automation.md)

---

# Related Documentation

Additional documentation describing the wider AI system:

- `docs/ai-staff/` — AI assistant vision, governance, and roadmap  
- `docs/scenarios/` — Individual automation scenario documentation
