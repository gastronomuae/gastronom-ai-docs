
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

09:00 AM Dubai Time

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

Retrieve orders that were placed during late-night hours and require a morning follow-up message.

This module scans the acknowledgement log and identifies orders that:

- received the **night delay acknowledgement**
- are still waiting for the **morning confirmation**

---

### Filter Logic

The following conditions are applied:

```
follow_up_status (G) = waiting_morning
AND
ack_type (F) = ack_night_delay
```

Explanation:

- `ack_type = ack_night_delay` ensures the order was placed during late-night hours
- `follow_up_status = waiting_morning` ensures the follow-up message has not yet been sent

This prevents duplicate morning messages.

---

### Example Output — Google Sheets Search Rows

```json
[
  {
    "0": "04 February, 23:01",
    "1": "#4084",
    "2": "Egor Larin",
    "3": "503889141",
    "4": "",
    "5": "ack_night_delay",
    "6": "waiting_morning",
    "__ROW_NUMBER__": 5,
    "__SPREADSHEET_ID__": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
    "__SHEET__": "Sheet1",
    "__IMTLENGTH__": 1,
    "__IMTINDEX__": 1
  }
]
```

Column mapping:

| Column | Field |
|------|------|
| 0 | order timestamp |
| 1 | order id |
| 2 | customer name |
| 3 | phone |
| 4 | acknowledgement time |
| 5 | ack_type |
| 6 | follow_up_status |

---

### Business Purpose

Ensure customers who placed orders overnight receive a business-hours confirmation message and follow-up communication in the morning.

---
### Module 3 — Tools: Set Multiple Variables (Phone Normalization)

Purpose:

Normalize the customer phone number for WhatsApp API compatibility.

This step ensures the phone number is converted into UAE international format without symbols.

---

### Variables Set

Removes:

- `+`
- spaces
- `-`
- `(`
- `)`

Variable created:

- `normalized_phone`

Normalization formula:

```make
{{if(substring(toString(10.clean_phone); 0; 1) = 5; substring(971; 0; 3) + 10.clean_phone; if(substring(toString(10.clean_phone); 0; 1) = 0; substring(971; 0; 3) + substring(toString(10.clean_phone); 1; length(toString(10.clean_phone))); 10.clean_phone))}}
```

---

### Logic

- If phone starts with `5` → prefix `971`
- If phone starts with `0` → remove `0` and prefix `971`
- Otherwise → keep as is

This produces a WhatsApp-compatible phone number such as:

```text
971XXXXXXXXX
```

---

### Module 4 — Shopify: Search Orders

Purpose:

Retrieve the latest order record from Shopify using the order number stored in the logging sheet, and validate the order status before sending the morning follow-up message.

This prevents follow-up messages from being sent for cancelled or invalid orders.

---

### Fields Retrieved

- `name` (order number)
- `display_financial_status`
- `display_fulfillment_status`
- `shippingAddress.phone`
- `createdAt`
- `lineItems` (if needed)

---

### Filter — Valid Orders Only

**Label:** `Not Cancelled Orders`

Condition:

```text
display_financial_status ≠ VOIDED
```

---

### Control Logic

- Morning follow-up message is sent only if the order is **not voided**
- Prevents sending confirmation for cancelled or invalidated orders

---

### Module 5 — WhatsApp Business Cloud: Send Template Message

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

### Module 6 — Google Sheets: Update Row

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
