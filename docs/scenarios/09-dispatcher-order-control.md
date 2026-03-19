
# Scenario 09 — Dispatcher Order Control (Telegram → Airtable)

## Purpose

This scenario allows the dispatcher to **control order delivery statuses directly from a Telegram logistics group**.

Commands sent in Telegram update the **Airtable order record**, which becomes the **single source of truth** for delivery status used by the AI customer support agent.

The scenario also allows the dispatcher to **query order status** without modifying it.

---

# Architecture

```
Webhook Custom -> Linked to 00-Scenario Dispatcher Telegram message
↓
Regex validation filter
↓
Extract variables:

- order_number
- command
  ↓
  Airtable → Search order
  ↓
  Filter (record exists)
  ↓
  Router (command)
  ↓
  Airtable → Update record (if applicable)
  ↓
  Telegram → Confirmation message
```

# Command format
```
4198-ready
4198-out
4198-outbuy
4198-buy
4198-issue
4198-cancel
4198-delivered
4198-status
```
Where:

- `4198` = Shopify order number
- command = action to perform


# Regex Validation Filter

Telegram messages must by in XXXX-command format:
```
  {{lower(2.message_text)}}
Matches Pattern (text)
  ^\d{4}-(ready|out|outbuy|buy|issue|cancel|delivered|status)\s*$
```

This ensures that:

- Only valid commands trigger the scenario
- Random messages in the logistics group are ignored

---

# Variable Extraction

## Order Number
```
{{lower(2.message_text)}}
```

Example:

Input: 4198-ready
Output: order_number = 4198


---

## Command

```
{{last(split(2.order.command; "-"))}}
```

Example:

Input: 4198-ready
Output: command = ready

---

# Airtable

## Safety Mechanism

After searching Airtable, apply a filter: Bundles > 0
If no record is found, send dispatcher message:
⚠️ Order {{order_number}} not found

Table: `orders`

Key field used for lookup: order_number
Search formula: {order_number} = "{{order_number}}"
Search limit: 1


---

# Router Commands

Each command updates the order status in Airtable.

| Command | operational_status | customer_status |
|-------|-------------------|----------------|
| ready | ready_for_driver | preparing_order |
| out | out_for_delivery | on_the_way |
| outbuy | out_for_delivery_buying | on_the_way |
| buy | buying_missing_items | preparing_order |
| issue | issue_customer_contact | awaiting_customer_response |
| cancel | cancelled | cancelled |
| delivered | delivered | delivered |

---

# Telegram Confirmation Messages

## READY

✅ {{order_number}} → READY FOR DRIVER

## OUT

🚚 {{order_number}} → OUT FOR DELIVERY

## OUTBUY

🛒🚚 {{order_number}} → OUT FOR DELIVERY (BUYING ITEMS)

## BUY

🛒 {{order_number}} → BUYING MISSING ITEMS

## ISSUE

⚠️ {{order_number}} → CUSTOMER ISSUE – DISPATCHER ACTION NEEDED

## CANCEL

❌ {{order_number}} → ORDER CANCELLED

## DELIVERED

📦 {{order_number}} → DELIVERED

# Role in Overall System

Scenario 09 provides **manual operational control of order delivery status**.

It integrates with:

| Scenario | Purpose |
|--------|--------|
| Scenario 07 | Escalates complex customer cases to dispatcher |
| Scenario 08 | AI replies to customers using Airtable status |
| Scenario 01 | Shopify order notifications |

Airtable acts as the **central order status database** used by both dispatcher and AI.

---

# Future Improvements

Planned extensions:

4198-eta-18:30
4198-driver-ali
4198-note-missing-item

These will update additional Airtable fields:

- `eta`
- `assigned_driver`
- `dispatcher_notes`

---

# Result

Dispatcher commands update order delivery status instantly, enabling:

- accurate customer updates
- structured logistics workflow
- centralized delivery tracking
- AI-assisted customer communication
