# Gastronom Dispatcher Guide

## Telegram Order Control

You will use **two Telegram group chats**.

------------------------------------------------------------------------

# 1️⃣ Support Chat

Example name:

**Gastronom Support AI**

This chat shows:

-   customer messages\
-   AI reply suggestions

Messages are handled by the **gastronom.support Telegram bot**.

You can respond to customers in **two ways**.

------------------------------------------------------------------------

## Method 1 --- Send AI message (recommended)

If the AI reply is correct, send it exactly as suggested.

Reply to the AI message and type:

send

Example:

Customer message: Where is my order 4178?

AI suggestion: Your order #4178 was received. We will check the status
and update you shortly.

Dispatcher reply: send

The **gastronom.support bot** sends the AI message to the customer.

------------------------------------------------------------------------

## Method 2 --- Modify AI reply

If you want to change the message, reply to the AI suggestion and type
your own message.

Example:

Your order is already with the courier and should arrive soon.

The **gastronom.support bot** will send your modified message to the
customer.

No command is needed.

------------------------------------------------------------------------

# 2️⃣ Logistics Chat

Example name:

**Gastronom Logistics**

This chat is used to **update delivery status**.

⚠️ Only **XXXX-command format** is allowed.

Format:

ORDERNUMBER-status

Example:

4178-out\
4198-ready

⚠️ No spaces allowed.

------------------------------------------------------------------------

# Available Commands

ready\
out\
outbuy\
buy\
issue\
cancel\
delivered\
status

------------------------------------------------------------------------

# When to Use Each Command

## Order ready for courier

Send:

4178-ready

Use when: - order packed - waiting for driver

------------------------------------------------------------------------

## Courier left warehouse

Send:

4178-out

Use when: - driver left warehouse - driver going to customer

------------------------------------------------------------------------

## Courier left warehouse and will buy missing items

Send:

4178-outbuy

Use when: - driver left warehouse - driver will buy missing items on the
way

------------------------------------------------------------------------

## Missing items must be purchased before delivery

Send:

4178-buy

Use when: - order cannot be completed - driver or dispatcher must buy
items

------------------------------------------------------------------------

## Delivery problem / need to contact customer

Send:

4178-issue

Use when: - wrong address - customer not answering - delivery problem

------------------------------------------------------------------------

## Order cancelled

Send:

4178-cancel

Use when: - customer cancelled the order - order cannot be delivered

------------------------------------------------------------------------

## Order delivered

Send:

4178-delivered

Use when: - order successfully delivered

------------------------------------------------------------------------

## Check order status

Send:

4178-status

Use when: - you want to check the current order status

------------------------------------------------------------------------

# Important

Always update the order when:

-   order is ready
-   courier leaves warehouse
-   courier is delivering
-   order delivered
-   delivery delayed or there is a problem

------------------------------------------------------------------------

# Quick Examples

Order ready:

4201-ready

Courier delivering:

4201-out

Courier buying items:

4201-outbuy

Order delivered:

4201-delivered

Customer not answering:

4201-issue
