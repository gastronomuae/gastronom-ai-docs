
# Scenario 2 — Morning WhatsApp Sender

## Objective

Send follow‑up WhatsApp messages in the morning for orders placed during late‑night hours.

Orders placed overnight (after configured business hours) receive an acknowledgement but require a proactive follow‑up message the next morning.

---

## Business Purpose

This scenario ensures customers who placed orders late at night receive a confirmation message in the morning confirming that the team is processing their order.

The scenario:

- scans the `orders_acknowledged` Google Sheet
- finds orders marked for morning follow‑up
- sends the **Night Follow‑Up template**
- updates the record to prevent duplicate messages

---

## Systems / Dependencies

- Make.com
- Google Sheets
- WhatsApp Business Cloud API

### Infrastructure Stack

- Make.com (automation engine)
- Google Sheets (order log + follow‑up tracking)
- WhatsApp Cloud API (message sending)

---

## Trigger

### Schedule

Runs daily at:

10:00 AM Dubai Time

This corresponds to the configuration value stored in:

`automation_config → night_message_time`

---

## Configuration Sources

### Google Sheet: `orders_acknowledged`

Purpose:

Stores acknowledgement logs and tracks whether a morning follow‑up message still needs to be sent.

Columns:

| Column | Field |
|------|------|
| A | order timestamp |
| B | order id |
| C | customer name |
| D | phone |
| E | acknowledgement time |
| F | acknowledgement type |
| G | follow-up status |

Possible follow‑up values:

- `pending`
- `completed`

---

## Processing Flow

### High‑level flow

1. Scheduler (daily trigger)
2. Google Sheets — Search Rows
3. Filter rows where follow_up_status = pending
4. WhatsApp Cloud API — Send Template Message
5. Google Sheets — Update Row

---

## Module Details

### Module 1 — Scheduler

Purpose:

Trigger the scenario once per day.

Configuration:

- Run time: **10:00**
- Timezone: **Asia/Dubai**

---

### Module 2 — Google Sheets: Search Rows

Sheet:

`orders_acknowledged`

Purpose:

Retrieve all rows where a follow‑up message still needs to be sent.

Example filter:

```
follow_up_status = pending
```

Example output:

```json
[
  {
    "order_id": "4109",
    "customer_name": "Alex",
    "phone": "971525970815",
    "follow_up_status": "pending"
  }
]
```

---

### Module 3 — WhatsApp Business Cloud: Send Template Message

Purpose:

Send the **Night Follow‑Up template** to the customer.

Variables passed:

- customer_name_en
- customer_name_ru
- order_id_en
- order_id_ru

Example API response:

```json
{
  "messaging_product": "whatsapp",
  "messages": [
    {
      "id": "wamid.EXAMPLE123456",
      "message_status": "accepted"
    }
  ]
}
```

---

### Module 4 — Google Sheets: Update Row

Purpose:

Mark the follow‑up message as completed to prevent duplicate sends.

Updated field:

```
follow_up_status = completed
```

Example output:

```json
{
  "updatedRows": 1,
  "updatedColumns": 1
}
```

---

## Technical Snapshot

Scenario Name in Make:

Morning WhatsApp Sender

Schedule:

Daily at 10:00 (Asia/Dubai)

Key fields used:

- order_id
- customer_name
- phone
- follow_up_status

---

## Open Items

Future improvements:

- move morning send time fully to configuration sheet
- add retry logic for failed WhatsApp sends
- add logging for failed message attempts
