# Scenario 1 — Shopify → WhatsApp Order Notification

## Objective

Automatically send WhatsApp order confirmation messages after a successful Shopify order, using time-based routing (Work Hours / After Hours / Late Night), and log activity in Google Sheets.

---

## Business Purpose

This scenario ensures that customers receive an acknowledgement on WhatsApp shortly after placing a paid Shopify order.

The logic adapts the acknowledgement message depending on the order time in Dubai:

- during work hours → immediate confirmation
- after hours → confirmation with next-morning expectation
- late night → defer proactive confirmation to the Morning WhatsApp Sender scenario

It also logs each acknowledgement into Google Sheets for traceability and follow-up control. 

---

## Systems / Dependencies

- Shopify
- Make.com
- Google Sheets
- WhatsApp Business Cloud API
- Telegram (support escalation exists as separate scenario)

### Infrastructure Stack

- Shopify (order trigger)
- Make.com (automation engine)
- Google Sheets (configuration + logging)
- WhatsApp Cloud API (Meta Business)
- Telegram (support escalation – separate scenario) 

---

## Trigger

### Module
**Shopify – Watch Orders**

### Trigger Condition
- New paid order

### Captured Data
- Order ID
- Customer name
- Phone number
- Order items
- Order total
- Created at timestamp 

---

## Configuration Sources

### Google Sheet: `automation_config`

Purpose:
Store business hour configuration dynamically.

The document lists these settings:

| setting | value (Dubai time) | UTC time |
|---|---:|---:|
| work_start | 9 | 5 |
| work_end | 18 | 14 |
| after_hours_end | 21 | 17 |
| night_message_time | 10 | — |

Used via:
- Google Sheets → Search Rows
- values referenced inside router filters

### Important note
Current implementation is **partially dynamic and partially hardcoded**.  
Backlog item: move all hour boundaries fully into Google Sheets so business hours can be adjusted centrally without editing Make router filters. 

### Google Sheets used

#### 1. `automation_config`
Stores business-hour boundaries. 

#### 2. `orders_acknowledged`
Stores acknowledgement logging and morning follow-up status. 

---

## Processing Flow

### High-level flow

1. Shopify – Watch Orders
2. Google Sheets – Search Rows
3. Tools – Set Multiple Variables (Block 1)
4. Tools – Set Multiple Variables (Block 2)
5. Airtable (creaste in orders table) - currnelty disabled
6. Router
   - Work Hours
   - After Hours
   - Late Night
7. WhatsApp Business Cloud – Send Template Message
8. Google Sheets – Add Row 

---

## Module Details

### Module 1 — Shopify: Watch Orders

Purpose:
Receive order data from Shopify.

Fields used:
- Order ID
- Customer name
- `shippingAddress.phone`
- Order total
- Created at timestamp 

---

### Module 2 — Google Sheets: Search Rows

Sheet:
`automation_config`

Purpose:
- retrieve business hour configuration dynamically
- provide `work_start`, `work_end`, `after_hours_end` values for router

Example filter:
- `setting = work_start`

Example output:

```json
[
  {
    "0": "work_start",
    "1": "9",
    "2": "5",
    "__ROW_NUMBER__": 2,
    "__SPREADSHEET_ID__": "1dJYebt3xJz8L0OO_P692HEL6DYC5964CBa_5qE4Vcf4",
    "__SHEET__": "Sheet1",
    "__IMTLENGTH__": 1,
    "__IMTINDEX__": 1
  }
]
```

---

### Module 3 — Tools: Set Multiple Variables (Block 1)

Purpose:

Normalize the customer phone number and extract key fields from the Shopify order payload.

Typical variables created:

* `clean_phone`
* `order_id`
* `customer_name`

Phone cleaning formula used in Make:

```make
replace(
  replace(
    replace(
      replace(
        replace(1.shippingAddress.phone; "+"; "");
      " "; "");
    "-"; "");
  "("; "");
")"; "")
```

Example output:

```json
[
  {
    "clean_phone": "525970815"
  }
]
```

Explanation:

This block removes formatting characters from the Shopify phone number:

* `+`
* spaces
* `-`
* parentheses

The result is a numeric string that can be normalized into UAE format in the next step.

---

### Module 4 — Tools: Set Multiple Variables (Block 2)

Purpose:

Convert the cleaned phone number into a WhatsApp-compatible UAE format.

The logic ensures the phone number always begins with `971`.

Normalization formula:

```make
if(
  substring(toString(31.clean_phone); 0; 1) = 5;
  "971" & 31.clean_phone;
  if(
    substring(toString(31.clean_phone); 0; 1) = 0;
    "971" & substring(toString(31.clean_phone); 1; length(toString(31.clean_phone)));
    31.clean_phone
  )
)
```

Explanation:

Case handling:

| Input format   | Result         |
| -------------- | -------------- |
| `525970815`    | `971525970815` |
| `0525970815`   | `971525970815` |
| `971525970815` | unchanged      |

Example output:

```json
[
  {
    "normalized_phone": "971525970815"
  }
]
```

This normalized phone number is then used in the **WhatsApp Cloud API send message module**.



### Module 5 - Router Logic

The router determines which acknowledgement message should be sent based on the order time in Dubai.

Time is extracted using:

```make
formatDate(1.Created at; H; Asia/Dubai)
```

Three time buckets exist:

* Work Hours
* After Hours
* Late Night

---

### Branch 1 — Work Hours

Condition:

```make
formatDate(1.Created at; H; Asia/Dubai) >= 3.value (B)
AND
formatDate(1.Created at; H; Asia/Dubai) < 18
```

Explanation:

* `formatDate(1.Created at; H; Asia/Dubai)` extracts the order hour
* `3.value (B) represents the `work_start` value retrieved from Google Sheets.
   Module 3 (Google Sheets – Search Rows) returns the configuration row:
   setting = work_start
   value = 9
   Column B contains the hour value used in the router filter.
* `18` currently represents the hard-coded work_end boundary

If this branch matches:

1. Send WhatsApp **Work Hours template**
2. Log acknowledgement to Google Sheets

---

### Branch 2 — After Hours

Condition:

```make
formatDate(1.Created at; H; Asia/Dubai) >= 18
AND
formatDate(1.Created at; H; Asia/Dubai) < 21
```

Explanation:

* Orders placed between **18:00 and 21:00 Dubai time**
* Both boundaries are currently **hardcoded values**

If this branch matches:

1. Send WhatsApp **After Hours template**
2. Log acknowledgement to Google Sheets

---

### Branch 3 — Late Night

Default router fallback route.

Condition:

```make
formatDate(1.Created at; H; Asia/Dubai) >= 21
OR
formatDate(1.Created at; H; Asia/Dubai) < 3.value (B)
```

Explanation:

* Orders placed **after 21:00** or **before the work_start time**
* This covers the overnight window

If this branch matches:

1. Send **Night acknowledgement template**
2. Mark order for follow-up by the **Morning WhatsApp Sender scenario**

---

## WhatsApp Message Sending

### Module 6 — WhatsApp Business Cloud: Send Template Message

#### Purpose

Send order acknowledgement to the customer’s WhatsApp number using approved WhatsApp Business templates.

---

#### WhatsApp API Setup

WhatsApp Cloud API configuration for this number is managed in the Meta Developers Console:

https://developers.facebook.com/apps/25486959764339557/use_cases/customize/?use_case_enum=WHATSAPP_BUSINESS_MESSAGING&business_id=336841861276587

This console is used for:

- configuring the WhatsApp phone number
- managing API tokens
- testing message delivery
- webhook configuration

WhatsApp API Setup
```
Meta App: message automation
App ID: 25486959764339557
Business Portfolio ID: 336841861276587
WABA ID: 1321636813460403
Phone Number ID: 1192944337224419
Display Number: +971 54 277 8985
Platform: CLOUD_API
Throughput: STANDARD
```

Important production setup note
```
Production phone registration was completed via:
POST /v25.0/1192944337224419/register

WABA webhook subscription was completed via:
POST /v25.0/1321636813460403/subscribed_apps

Without subscribed_apps, Meta test webhooks work but real inbound messages do not reach Make.
```


---

#### Template Management

All WhatsApp message templates and template performance statistics are managed in Meta Business Manager: 
WABA ID: 1321636813460403
https://business.facebook.com/latest/whatsapp_manager/message_templates?business_id=336841861276587&asset_id=1321636813460403 


Templates must be approved before they can be used by the WhatsApp Cloud API.

---

#### Templates Used in This Scenario

- Work Hours template
- After Hours template
- Late Night template

---

#### Template Variables

Variables passed from the automation scenario:

- `customer_name_en`
- `customer_name_ru`
- `order_id_en`
- `order_id_ru`

These variables are dynamically injected into the WhatsApp template when the message is sent.

---

### Example Response — Work Hours Send

```json
{
  "messaging_product": "whatsapp",
  "contacts": [
    {
      "input": "+971561345294",
      "wa_id": "971561345294"
    }
  ],
  "messages": [
    {
      "id": "wamid.HBgMOTcxNTYxMzQ1Mjk0FQIAERgSOEQxMkUxODhDRDAyNEFFOTFEAA==",
      "message_status": "accepted"
    }
  ]
}
```

---

### Example Response — After Hours Send

```json
{
  "messaging_product": "whatsapp",
  "contacts": [
    {
      "input": "+971561345294",
      "wa_id": "971561345294"
    }
  ],
  "messages": [
    {
      "id": "wamid.HBgMOTcxNTYxMzQ1Mjk0FQIAERgSMTdBODU4NTk2MUUzOTZFNTY3AA==",
      "message_status": "accepted"
    }
  ]
}
```

---

## Templates Used

1 = customer name EN
2 = order ID EN
3 = customer name RU
4 = order ID RU

### Template — Work Hours

English

Hello customer name EN 👋

We received your order order ID EN on gastronom.ae 🛒
We will contact you shortly with your delivery details.

If you need urgent assistance, please reply to this message.

Thank you 🙏

Здравствуйте customer name RU 👋

Мы получили ваш заказ order ID RU на gastronom.ae 🛒
Скоро свяжемся с вами для уточнения деталей доставки.

Если нужна срочная помощь, пожалуйста ответьте на это сообщение.

Спасибо 🙏

---

### Template — After Hours

Hello customer name EN 👋
We received your order order ID EN on gastronom.ae 🛒
We will contact you tomorrow morning to confirm delivery.

If you need urgent assistance:
https://wa.me/971523706376

Thank you 🙏

Здравствуйте customer name RU 👋
Мы получили ваш заказ order ID RU на gastronom.ae 🛒
Свяжемся с вами завтра утром для подтверждения доставки.

Если нужна срочная помощь:
https://wa.me/971523706376

Спасибо 🙏

---

### Template — Night Follow-Up

English

Hello customer name EN 👋
We received your order order ID EN on gastronom.ae 🛒
We will contact you tomorrow morning to confirm delivery.

If you need urgent assistance:
https://wa.me/971523706376

Thank you 🙏

Здравствуйте customer name RU 👋
Мы получили ваш заказ order ID RU на gastronom.ae 🛒
Свяжемся с вами завтра утром для подтверждения доставки.

Если нужна срочная помощь:
https://wa.me/971523706376

Спасибо 🙏

---

### Module 7 - Airtable - Create a record (for both routes where we ahve whastapp send message)

| Airtable field | Current / recommended value | Notes |
|---|---|---|
| `conversation_id` | `order_{{trim(replace(1.Name; "#"; ""))}}` | Better than WhatsApp Message ID. Use order as business thread reference. |
| `wa_number` | `{{32.normalized_phone}}` | Correct. Same phone format used by inbox. |
| `order_id` | `{{trim(replace(1.Name; "#"; ""))}}` | Correct. Removes `#`. |
| `message_id_external` | `{{21.Messages[].Message ID}}` / `{{26.Messages[].Message ID}}` | Correct. Use the WhatsApp `wamid...` from each branch. |
| `message_direction` | `outbound` | Correct. |
| `message_source` | `whatsapp_A` | Fine. Later you can use `automated_order_ack` if needed. |
| `message_text` | `Automated order acknowledgement sent for Order #{{1.Name}}. |
| `broad_category` | `support` | Correct. |
| `issue_category` | `order_status` | Recommended to add, if field exists. |
| `timestamp_utc` | `{{formatDate(now; "YYYY-MM-DDTHH:mm:ss.SSS[Z]"; "UTC")}}` | Better than Airtable Created Time. |
| `agent_name` | `make.com` | Correct. |
| `conversation_status` | `open` | Correct. |
| `customer_sentiment` | empty | Correct. |
| `Priority` | `normal` | Correct. |
| `channel` | `whatsapp_A` | Correct. |
| `conversation_started_at` | `{{formatDate(now; "YYYY-MM-DDTHH:mm:ss.SSS[Z]"; "UTC")}}` | Same as timestamp. |
| `conversation_closed_at` | empty | Correct. |
| `conversation_hash` | `{{32.normalized_phone}}` | Correct. Must be customer phone, even for outbound. |
| `label_source` | `system_default` | Fine. |


---

### Module 7 - Logging - Google Sheets — Add Row

Purpose:

Record acknowledgement events for auditing and follow-up.

Google Sheet 
https://docs.google.com/spreadsheets/d/1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4/edit?gid=0#gid=0

Sheet used:
`orders_acknowledged`

Columns:

| Column | Field                |
| ------ | -------------------- |
| A      | order timestamp      |
| B      | order id             |
| C      | customer name        |
| D      | phone                |
| E      | acknowledgement time |
| F      | acknowledgement type |
| G      | follow-up status     |

---

### Example Output — Add Row

#### Google Sheets Add Row — Work Hours (output sample)

```json
[
  {
    "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
    "tableRange": "Sheet1!A1:G25",
    "updates": {
      "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
      "updatedRange": "Sheet1!A26:G26",
      "updatedRows": 1,
      "updatedColumns": 7,
      "updatedCells": 7
    },
    "sheetName": "Sheet1",
    "rowNumber": 26
  }
]
```

---

#### Google Sheets Add Row — After Hours (output sample)

```json
[
  {
    "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
    "tableRange": "Sheet1!A1:G21",
    "updates": {
      "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
      "updatedRange": "Sheet1!A22:G22",
      "updatedRows": 1,
      "updatedColumns": 7,
      "updatedCells": 7
    },
    "sheetName": "Sheet1",
    "rowNumber": 22
  }
]
```

---

#### Google Sheets Add Row — Late Hours (output sample)

```json
[
  {
    "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
    "tableRange": "Sheet1!A1:G24",
    "updates": {
      "spreadsheetId": "1H6gflP7fJJy9Q8v7GUEUKIXCsbhwzPDu4Z7zZGuWbM4",
      "updatedRange": "Sheet1!A25:G25",
      "updatedRows": 1,
      "updatedColumns": 6,
      "updatedCells": 6
    },
    "sheetName": "Sheet1",
    "rowNumber": 25
  }
]
```

### Router Decision Snapshot

Store example of decision result for documentation: - order_hour: 14 - matched_branch: Work Hours - template_sent: work_hours_template
This ensures that later we can fully reconstruct logic without relying on memory.

---

## Scenario Technical Summary

Scenario Name in Make:

Integrations:
- Shopify
- Google Sheets
- WhatsApp Cloud API

Schedule:

Runs every 15 minutes.

Timezone logic:

```make
formatDate(1.Created at; H; Asia/Dubai)
```

Router branches:

1. Work Hours
2. After Hours
3. Late Night

Key Shopify fields used:

* name (order number)
* createdAt
* shippingAddress.phone
* totalPriceSet.amount
* lineItems[].sku
* lineItems[].quantity

---

## Open Items

The original document highlights several future improvements:

* remove hardcoded router hour values
* move all hour boundaries to Google Sheets configuration
* confirm duplicate prevention logic
* add better error handling
* validate router logic against real production scenario
