---
name: sure-finance-skills
description: Record, view, and delete financial transactions in Sure Finance. Use this skill when the user mentions spending money, receiving income, logging expenses, checking recent transactions, removing a transaction, or uploads a receipt or invoice image.
metadata:
  homepage: https://github.com/we-promise/sure
  require-secret: true
  require-secret-description: "Enter credentials as BASE_URL|API_KEY, e.g. http://192.168.1.100:3000|your-api-key"
---

# Sure Finance — Financial Recorder

## Instructions

Use this skill to record income or expense transactions, view recent transactions, or delete a transaction.

### Step 1 — Understand the user's intent

Classify the request into one of three actions:
- **record**: user is logging a purchase, payment, income, or any financial event
- **list**: user wants to see recent transactions or spending
- **delete**: user wants to remove or undo a transaction

If the user uploads an image (e.g. a receipt, invoice, or bank screenshot), treat it as a **record** request and extract the transaction fields from the image before proceeding to Step 2.

### Step 2 — Extract the required fields

**For "record":**
- `name` (required): a short description of the transaction (e.g. "Grocery shopping", "Salary", "Netflix subscription"). If extracting from a receipt image, use the merchant or store name.
- `amount` (required): the numeric amount, always positive (e.g. 42.50). If extracting from an image, use the total or grand total value.
- `nature` (required): `"expense"` for money going out, `"income"` for money coming in. Receipts and invoices are almost always `"expense"`.
- `date` (required): in YYYY-MM-DD format. Extract from the image if present; otherwise use today's date.
- `notes` (optional): any extra context — for images, you may include a brief summary of the items purchased.

If the image is unclear or a required field cannot be confidently read, ask the user to confirm the value before proceeding. Do not guess amounts.

**For "list":**
- `per_page` (optional, default 10): how many transactions to show
- `type` (optional): `"income"` or `"expense"` if the user wants to filter
- `start_date` / `end_date` (optional): YYYY-MM-DD range if the user specifies a period
- `search` (optional): keyword if the user asks about a specific merchant or category

**For "delete":**
- `id` (preferred): the transaction UUID if already known from a prior list response
- If `id` is unknown, provide `search` (keyword) and optionally `date` (YYYY-MM-DD) so the skill can find the transaction first

### Step 3 — Call run_js

Call the `run_js` tool with:
- script name: `index.html`
- data: a JSON string containing the fields above, always including `"action"`

#### For "record" — use a two-step flow

**Step 3a — fetch context first**

Always call `get_context` before recording. This returns the user's real account IDs and category IDs.

```json
{ "action": "get_context" }
```

The response contains:
- `accounts`: list of `{ id, name, type, currency }`
- `expense_categories`: list of `{ id, name, parent }`
- `income_categories`: list of `{ id, name, parent }`

Cache this result for the conversation. Only call `get_context` again if the user mentions adding a new account. Do not call it before every transaction.

**Step 3b — pick account and category, then record**

From the context response:
- Choose the most appropriate **account** based on the transaction (e.g. "Cash" for cash payments, "Visa" for card payments, or the first account if unclear).
- Choose the most appropriate **category** from `expense_categories` or `income_categories` based on the transaction name and nature. Use your semantic understanding — "Netflix" → "Subscriptions", "Grab" → "Transportation", "Salary" → "Employment Income".
- If no category is a reasonable match, omit `category_id`.

```json
{
  "action": "record",
  "account_id": "uuid-from-context",
  "category_id": "uuid-from-context",
  "name": "Grocery shopping",
  "amount": 52.80,
  "nature": "expense",
  "date": "2024-07-15",
  "notes": "Weekly groceries at Trader Joe's"
}
```

#### Example — list recent transactions
```json
{
  "action": "list",
  "per_page": 10
}
```

#### Example — delete by ID
```json
{
  "action": "delete",
  "id": "uuid-of-the-transaction"
}
```

#### Example — delete by search
```json
{
  "action": "delete",
  "search": "Grocery shopping",
  "date": "2024-07-15"
}
```

### Step 4 — Report results

After receiving the result from `run_js`:
- For **record**: confirm what was saved, including the name, amount, date, account, and category.
- For **list**: present the transactions in a readable format (date, name, amount, category).
- For **delete**: confirm the transaction that was removed, or explain if it could not be found.
