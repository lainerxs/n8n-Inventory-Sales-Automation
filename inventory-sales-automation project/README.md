# Inventory & Sales Automation (n8n)

A production-style automation built in [n8n](https://n8n.io) that runs a small retail business's
order-to-inventory pipeline end to end: a sale comes in, stock gets validated and updated in
Google Sheets in real time, and the business owner gets daily low-stock digests, weekly sales
analytics, and instant alerts if anything breaks — all without writing a traditional backend.

This project was built to learn workflow automation hands-on, using a real small store (rice,
eggs, and cooking oil) as the use case.

## What it does

A sale placed through the demo storefront (or any HTTP client) triggers a workflow that:

1. Validates the order's shape (order ID, payment method, line items) and rejects malformed
   requests with a clear error — before touching any data.
2. Checks for duplicate submissions (e.g. a retried request) so the same sale is never recorded
   twice.
3. Looks up real-time stock per product, fulfills what it can, and flags anything it can't
   (unknown product, insufficient stock) as an exception instead of silently failing.
4. Updates inventory counts and logs the sale — calculating revenue from the product's current
   selling price, not the price at some earlier point in time.
5. Sends a Telegram alert immediately if a sale pushes a product's stock at or below its reorder
   threshold.

Independently of that, two scheduled jobs and one safety net run in the same workflow:

- **Daily Low-Stock Digest** (8 AM) — one consolidated Telegram + email message listing every
  product that needs reordering, instead of a separate ping per low-stock sale.
- **Weekly Sales Analytics Report** (Monday 9 AM) — top-selling products, total units, and total
  revenue for the past 7 days, sent the same way.
- **Global Error Alerts** — if any node in the workflow fails for any reason, a Telegram alert
  fires immediately with the error detail, so failures don't go unnoticed.

## Architecture

The whole thing lives in one workflow file, split into four independent branches that share a
single Google Sheet as their data store:

```
Branch A — Real-Time Sale Processing            (Webhook trigger)
  🛒 New Sale Order → Validate → Duplicate check → Match items against stock
    → Update Inventory Stock → Record Sale → low-stock check → Telegram alert (if needed)
    → respond to caller with a real result (not just "received")

Branch B — Daily Low-Stock Digest               (Schedule: every day, 8 AM)
  Read Inventory → build digest → Telegram + Email

Branch C — Weekly Sales Analytics Report        (Schedule: every Monday, 9 AM)
  Read SalesLog (last 7 days) → compute top products / totals → Telegram + Email

Branch D — Global Error Alerts                  (Error Trigger, bound to this workflow)
  Any node fails, anywhere → format error → Telegram alert

Branch E — Product Availability                 (Webhook trigger, GET, read-only)
  Read Inventory → map stock to a status (in stock / low stock / out of stock)
    → respond with status per product (never exact counts) — feeds the storefront's
    live availability badges
```

**Data store:** a Google Sheet with three tabs — `Inventory` (the product catalog and live stock
counts), `SalesLog` (append-only record of every completed sale), and `Exceptions` (append-only
record of every sale that couldn't be fulfilled, with why).

**Notifications:** Telegram (via a bot) for instant alerts, and SMTP email for the daily/weekly
digests.

## Tech stack

- **n8n** — workflow engine (self-hosted locally for this project)
- **Google Sheets** — data store (no traditional database)
- **Telegram Bot API** — instant notifications
- **SMTP (Email Send node)** — scheduled digest emails
- **Vanilla HTML/CSS/JS** — the demo storefront used to generate test orders, with no build step
  or dependencies

## Repo structure

```
.
├── workflow/
│   └── inventory-sales-automation.json   # Import this into n8n
├── google-sheet-template/
│   └── inventory-sales-sheet-template.xlsx   # Starter copy of the 3-tab Google Sheet
├── demo-storefront/
│   └── order-console.html                # Open directly in a browser — no server needed
└── docs/                                 # Screenshots / recordings (add your own)
```

## Setting it up yourself

1. **Import the workflow** — in n8n, open **Workflows → Import from File** and select
   `workflow/inventory-sales-automation.json`. Personal values were replaced with placeholders
   (`YOUR_GOOGLE_SHEET_ID`, `YOUR_TELEGRAM_CHAT_ID`, `your-sender@example.com`,
   `owner@example.com`) — swap in your own after importing.
2. **Set up the Google Sheet** — create a new Google Sheet and copy in the three tabs from
   `google-sheet-template/inventory-sales-sheet-template.xlsx` (headers and starter rows
   included — open the sheet's "Read Me" tab for what each column means). Connect it to every
   Google Sheets node in the workflow with your own Google Sheets OAuth credential.
3. **Connect Telegram and Email (optional but recommended)** — create a Telegram bot via
   [@BotFather](https://t.me/BotFather) and add its credential to the Telegram nodes; add SMTP
   credentials to the Email Send nodes. The workflow runs without these, but the alert/digest
   branches won't deliver anywhere.
4. **Also set Workflow Settings → Error Workflow** to this same workflow, so Branch D can catch
   failures from the other branches.
5. **Test Branch A** — click **Execute workflow** on the `🛒 New Sale Order (Webhook)` node to get
   a Test URL, paste it into `demo-storefront/order-console.html`'s settings panel (open the file
   directly in a browser), and place a test order.
6. **Activate the workflow** once everything checks out, and switch the storefront to the
   webhook's Production URL.
7. **(Optional) Wire up live stock badges** — click **Execute workflow** on the
   `📦 Get Product Availability (Webhook)` node the same way, paste its URL into the storefront's
   "Product Availability Webhook URL" field, and each product card will show In Stock / Low Stock
   / Out of Stock, refreshed automatically after every order.

## Engineering notes — problems hit and how they were solved

A few non-obvious issues came up building this that are worth calling out, since they're the
kind of thing that doesn't show up until you actually run the workflow against real data:

- **A Google Sheets node's output replaces the whole data payload with the row it just
  read/wrote**, using the *target sheet's* column names as keys — it doesn't pass through
  whatever came in from earlier nodes. Any node downstream of a sheet write that still needed
  the original order data (e.g. the low-stock check, the Telegram alert text) had to reference it
  explicitly with `$('Match Items & Calculate Stock').item.json.fieldName` rather than the bare
  `$json.fieldName` that worked everywhere else. Using `.item` (not `.first()`) matters here too,
  since an order with multiple line items runs through this node once per item in parallel —
  `.first()` would silently return item `[0]`'s data for every one of them.

- **n8n's Google Sheets node resource locator for a tab name needs the tab's actual numeric
  `gid`, not its display name**, once the mapping mode is set to manual. Using the plain name
  produces a `Sheet with ID <name> not found` error that looks like a permissions problem but
  isn't — the gid is visible in the tab's URL (`#gid=...`).

- **Reselecting the Document or Sheet dropdown in the n8n UI silently resets column mapping from
  "Map Each Column Manually" back to "Map Automatically."** This turned into a repeating bug
  every time a sheet reference needed to be fixed — the fix is to re-check (and reset) the
  mapping mode every time that dropdown gets touched, not just once.

- **Revenue needed to be calculated from the price the customer actually pays, not the store's
  cost.** The Inventory sheet originally had a single "Price" column, which worked until the real
  business model came up: buy at one price from a supplier, resell at a markup. The fix was
  splitting that into `Unit Price` (cost) and `Selling Price` (what's charged), with every
  revenue calculation reading from `Selling Price` — `Unit Price` is kept for a possible future
  profit-margin feature but isn't used in any calculation yet.

## Demo storefront

`demo-storefront/order-console.html` is a dependency-free HTML page built to make Branch A easy
to test and demo without a REST client: it renders the product catalog as a small storefront
(Rice, Eggs, Cooking Oil, each with size/packaging options where relevant), lets you build a cart,
and POSTs the order to the workflow's webhook — showing the real success/error response back,
including which items were fulfilled and which weren't. Prices shown are placeholders for the
demo; the real price, stock check, and revenue are always calculated server-side by the workflow
from the Inventory sheet.

Open the file directly in any browser — no install or server required. If the request fails with
a network/CORS error, add `*` under the Webhook node's **Options → Allowed Origins (CORS)**.

If an order can't be completed as submitted — a validation error, or an item that's out of
stock — the storefront shows why and clears the cart, since the order has already been decided
one way or another server-side. A fully successful order instead shows an itemized receipt and
waits for **Done** before clearing, so the buyer has a moment to see what they bought.

## License

MIT — feel free to adapt this for your own learning projects. Swap in your own product catalog,
credentials, and branding.
