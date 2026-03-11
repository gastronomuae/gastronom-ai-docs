# Gastronom Customer Support Knowledgebase

This knowledgebase defines official Gastronom customer support policies.

All rules are written in English, but **replies must always be generated in the customer's language**.

If the customer writes in Russian → reply in Russian.  
If the customer writes in English → reply in English.

Configuration variables such as phone numbers, delivery cutoffs, exchange rates and locations are provided dynamically via the configuration system and may override static information.

---

# DELIVERY POLICY

## Do you deliver to my address?

Yes, we deliver across all areas of Dubai, regardless of the district.

Our warehouse is located at:  
{{warehouse_location}}

If you have any questions about delivery, please contact us via WhatsApp:  
**{{support_whatsapp}}**

---

## When will my order be delivered?

We aim to deliver orders on the **same day** if they are placed **before {{delivery_same_day_cutoff}}**.

Orders placed after this time may be scheduled for the **next available delivery window** depending on courier availability and order volume.

If an order is placed **between {{delivery_same_day_cutoff}} and {{delivery_chat_cutoff}}**, same-day delivery **might still be possible**, but it requires **manual confirmation from our team**.

In such cases, customers should contact us via WhatsApp:

**{{support_whatsapp}}**

---

## What if I am not satisfied with the product quality?

If you are not satisfied with the quality of any product, please contact us by phone or WhatsApp:

**{{support_whatsapp}}**

We will conduct an internal review. If the issue is confirmed, we will either issue a refund or replace the product.

---

## What happens if a product is unavailable?

If a product is unavailable, we may suggest a **similar alternative**.

All substitutions are always discussed and confirmed with the customer via phone or WhatsApp before being made.

---

## Replacement products or weighted items

When replacing products or when orders include weighted items, the final order total may differ slightly from the amount shown at checkout.

---

## What if I cannot receive the delivery?

If a customer cannot receive the delivery personally, they can leave a note in the order comments asking the courier to **leave the order at the door**.

The delivery can also be cancelled or rescheduled by contacting:

**{{support_whatsapp}}**

---

## How do you store products and prepare orders?

All products are stored according to proper temperature conditions and manufacturer recommendations.

Special attention is paid to packaging quality and product freshness.

---

## Why do product substitutions happen?

Our assortment is constantly updated and product availability may change quickly.

Even if an item was available when the order was placed, it may be sold out before the order is packed.

In such cases we suggest a similar product and always confirm the replacement with the customer.

---

## I received the wrong item or something is missing

If an error occurred, customers should contact us via WhatsApp or phone:

**{{support_whatsapp}}**

We will investigate the issue and resolve it as quickly as possible.

---

# PAYMENT METHODS

We offer several convenient payment options for your order.

---

## Online Payment (Credit Card)

The easiest way to pay is directly on our website during checkout.

Customers can pay online using a **credit or debit card** when placing their order.

Currently **Apple Pay is not available**.

---

## Cash on Delivery

Customers can request **cash payment when the order is delivered**.

Typical UAE banknotes include:

- 10 AED  
- 20 AED  
- 50 AED  
- 100 AED  
- 500 AED  
- 1000 AED  

---

## Card Payment on Delivery (POS Terminal)

In some cases we can send a **POS terminal with the courier** so the customer can pay by card upon delivery.

However:

- this option requires additional logistics  
- it may **increase delivery time**  
- it may **not always be available**

Sometimes deliveries are dispatched using **external courier services** during busy periods.

External couriers **cannot accept POS terminal payments**, so in those cases card payment on delivery may not be possible.

Customers should confirm POS availability with the team in advance.

---

## Payment Link

If needed, we can send a **secure online payment link**.

This option is useful when:

- the customer is **not at home**
- there is an issue with **cash payment**
- a **POS terminal is not available**
- payment needs to be completed **after the order is placed**

Once payment is completed, we proceed with delivery.

---

## Payment in Russian Rubles (Bank Transfer)

We also support **payment in Russian rubles via bank transfer**.

The process:

1. Order total is calculated in AED.
2. The amount is converted to RUB using the exchange rate.
3. The customer transfers the RUB amount to our bank.
4. The customer sends the transfer receipt.
5. After confirmation we proceed with delivery.

Exchange rate used for conversion:

**{{rub_aed_exchange_rate}}**

Bank details:

**Bank:** {{rub_payment_bank_name}}  
**Transfer phone / account:** {{rub_payment_phone}}

The RUB amount may be **rounded up slightly to a convenient amount**.

---

## Refunds

If a refund is required, we process it using the same payment method whenever possible.

For **online payments or payment links**, refunds are processed through our payment provider.

Refunds may take approximately:

**3–7 business days**

to appear on the customer's bank card depending on the issuing bank.

Refunds may be issued:

- full refunds (for cancelled orders)
- partial refunds (for missing or replaced items)

For refund questions customers should contact:

**{{support_whatsapp}}**

---

# INTERNAL AI INSTRUCTIONS

## Delivery escalation rule

If a customer asks for **same-day delivery after {{delivery_same_day_cutoff}} but before {{delivery_chat_cutoff}}**:

The AI should:

1. Respond politely that same-day delivery **might still be possible**
2. Ask the customer to contact **{{support_whatsapp}}**
3. Flag the conversation for **human escalation if automation allows**

---

## Payment explanation rule

When customers ask about payment options, list clearly:

- Online credit card payment on the website
- Cash on delivery
- Card payment on delivery (POS terminal, limited availability)
- Payment link
- RUB bank transfer

---

## RUB payment rule

If a customer prefers RUB payment, calculate the final RUB amount using the configured exchange rate, round it up to the nearest convenient hundred RUB, and provide the bank details. Do not disclose the exchange rate or calculation.

---

## Refund rule

If a customer asks about refunds after online payment:

Explain that refunds are processed through the payment provider and usually take **3–7 business days**.

---

## Cash change rule (internal)

For cash-on-delivery orders, calculate possible change based on common UAE banknotes:

10, 20, 50, 100, 500, 1000 AED.

Example:

If order total is **820 AED**, the courier should be prepared to provide change from a **1000 AED banknote**.
