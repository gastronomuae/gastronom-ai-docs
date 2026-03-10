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
