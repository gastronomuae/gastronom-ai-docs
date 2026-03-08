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

It also logs each acknowledgement into Google Sheets for traceability and follow-up control. :contentReference[oaicite:2]{index=2}

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
- Telegram (support escalation – separate scenario) :contentReference[oaicite:3]{index=3}

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
- Created at timestamp :contentReference[oaicite:4]{index=4}

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
Backlog item: move all hour boundaries fully into Google Sheets so business hours can be adjusted centrally without editing Make router filters. :contentReference[oaicite:5]{index=5}

### Google Sheets used

#### 1. `automation_config`
Stores business-hour boundaries. :contentReference[oaicite:6]{index=6}

#### 2. `orders_acknowledged`
Stores acknowledgement logging and morning follow-up status. :contentReference[oaicite:7]{index=7}

---

## Processing Flow

### High-level flow

1. Shopify – Watch Orders
2. Google Sheets – Search Rows
3. Tools – Set Multiple Variables (Block 1)
4. Tools – Set Multiple Variables (Block 2)
5. Router
   - Work Hours
   - After Hours
   - Late Night
6. WhatsApp Business Cloud – Send Template Message
7. Google Sheets – Add Row :contentReference[oaicite:8]{index=8}

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
- Created at timestamp :contentReference[oaicite:9]{index=9}

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

### Module 3 — Tools: Set Multiple Variables (Block 1)

Purpose:
- normalize phone number
- store order fields into variables

Variables typically set here:
- normalized_phone
- order_id
- customer_name

Phone cleaning formula:
{{replace(replace(replace(replace(replace(1.shippingAddress.phone; "+"; ""); ; ""); "-"; ""); "("; ""); ")"; "")}}

Output sample:
[
  {
    "clean_phone": "525970815"
  }
]

### Module 4 — Tools: Set Multiple Variables (Block 2)

Purpose:
Normalize UAE phone format for WhatsApp API compatibility.

Formula:
{{if(substring(toString(31.clean_phone); 0; 1) = 5; substring(971; 0; 3) + 31.clean_phone;
if(substring(toString(31.clean_phone); 0; 1) = 0; substring(971; 0; 3) + substring(toString(31.clean_phone); 1;
length(toString(31.clean_phone))); 31.clean_phone))}}

Output sample:
[
  {
    "normalized_phone": "971525970815"
  }
]
