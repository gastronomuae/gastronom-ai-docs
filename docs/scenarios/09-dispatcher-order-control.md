
# Architecture

Dispatcher Telegram command
        ↓
Regex validation
        ↓
Extract:
order_number
command
        ↓
Airtable – Search order
        ↓
Router
        ↓
Update Airtable status
        ↓
Telegram confirmation

# Command format

4198-ready
4198-out
4198-outbuy
4198-buy
4198-issue
4198-cancel
4198-delivered
4198-status
