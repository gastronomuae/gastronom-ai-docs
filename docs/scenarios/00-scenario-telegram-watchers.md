# Scenario 00 — Telegram Router & Handshake

## Overview

Scenario 00 acts as the central router for all incoming Telegram messages.  
It classifies messages and forwards them to the appropriate downstream scenarios via HTTP webhooks.

This scenario supports three main flows:

1. Config / system commands
2. Order dispatcher commands (Scenario 9)
3. Support replies (Scenario 8)

---

## Module Structure

1. Telegram Bot — Watch Updates
2. Router with 3 routes:
   - Route 1: Config updates
   - Route 2: Scenario 9 (Order dispatcher)
   - Route 3: Scenario 8 (Support replies)

---

## Route 1 — Config Updates

### Purpose
Handles system-level commands such as:
- `/commands`
- `test_key`
- callback queries

### Filter

1.message.text starts with "/"
OR
1.callback_query.data exists
OR
1.message.text starts with "test_key"

---

### HTTP Module

Webhook:
https://hook.eu1.make.com/vmhcxiwx9w454pg1jlj4yjglnce2w4r7

---

## Route 2 — Scenario 9 (Order Dispatcher)

### Filter

1.message.text exists
AND
lower(1.message.text) matches pattern
^\d{4}-(ready|out|outbuy|buy|issue|cancel|delivered|status)$

---

### Payload

{
  "order": {
    "number": "{{first(split(1.message.text; '-'))}}",
    "command": "{{last(split(1.message.text; '-'))}}"
  },
  "chat_id": "{{1.message.chat.id}}",
  "operator": {
    "username": "{{1.message.from.username}}"
  },
  "message_text": "{{1.message.text}}"
}

Webhook:
https://hook.eu1.make.com/tjhmg3perv5y4rdfhfp73vsznqb6hpom

---

## Route 3 — Scenario 8 (Support Replies)

### Filter

(Reply exists AND bot message AND support chat)
OR
(Reply exists AND bot message AND logistics chat)

---

### Payload

{
  "message": {
    "text": "{{1.message.text}}",
    "chat": {
      "id": "{{1.message.chat.id}}"
    },
    "from": {
      "username": "{{1.message.from.username}}"
    },
    "reply_to_message": {
      "message_id": "{{1.message.reply_to_message.message_id}}",
      "text": "{{1.message.reply_to_message.text}}",
      "from": {
        "username": "{{1.message.reply_to_message.from.username}}"
      }
    }
  }
}

Webhook:
https://hook.eu1.make.com/tx8ttg0cqgu9qiftle83gvrdadh7q38v

---

## Summary

Scenario 00 routes all Telegram inputs into structured downstream flows and ensures consistent data handling across the system.
