# Setu BillPay - WhatsApp Conversational Product Specification

**Version:** 1.0
**Platform:** Meta WhatsApp Cloud API
**Market:** India

---

## Executive Summary

Setu BillPay is a WhatsApp-native bill payment and management experience that leverages Meta's interactive messaging features (Lists, Reply Buttons, Flows) to deliver a seamless, trust-first payment experience for Indian consumers.

**Core Features:**
1. Pay bills across top categories (Electricity, FASTag, Loans, etc.)
2. One-tap saved bill payments
3. Smart reminders and AutoPay/SmartPay
4. Payment tracking and receipts
5. Contextual help and human escalation

**Key Constraints:**
- 24-hour customer service window for free-form messages
- Outside window: only approved message templates allowed
- Max 10 rows per interactive list
- WhatsApp Flows for multi-field structured input
- Task-focused: redirect off-topic queries to menu

---

## 1. DATA MODELS

### 1.1 UserProfile
```typescript
{
  wa_id: string (primary key, E.164 format)
  language: string (default: 'en', options: 'en'|'hi'|'kn')
  consent_save_billers: boolean
  notification_time_pref: string (HH:MM format, default: '10:00')
  onboarding_completed: boolean
  created_at: timestamp
  updated_at: timestamp
}
```

### 1.2 SavedBiller
```typescript
{
  saved_biller_id: uuid (primary key)
  wa_id: string (foreign key to UserProfile)
  category: string (ELECTRICITY|FASTAG|LOAN|MOBILE_POSTPAID|LPG_GAS|CREDIT_CARD|INSURANCE|DTH|WATER|BROADBAND)
  biller_id: string
  biller_name: string
  consumer_params: jsonb (encrypted - store as {"param1": "value1"})
  consumer_masked: string (display version - e.g., "XXXX1234")
  nickname: string (optional, user-defined)
  last_fetch_status: string (SUCCESS|FAILED|PENDING)
  last_due_date: date
  last_amount: decimal
  last_fetched_at: timestamp
  is_autopay_enabled: boolean
  autopay_type: string (null|AUTOPAY|SMARTPAY)
  autopay_cap_amount: decimal (max auto-pay limit)
  autopay_mandate_id: string (optional)
  created_at: timestamp
  updated_at: timestamp
}
```

### 1.3 Reminder
```typescript
{
  reminder_id: uuid (primary key)
  wa_id: string (foreign key)
  saved_biller_id: uuid (foreign key)
  type: string (DUE_DATE|MONTHLY_DATE)
  offset_days: integer (for DUE_DATE: e.g., -3 means 3 days before due)
  monthly_date: integer (1-31, for MONTHLY_DATE type)
  preferred_time: string (HH:MM)
  is_enabled: boolean
  snooze_until: timestamp (null if not snoozed)
  last_sent_at: timestamp
  next_run_at: timestamp
  created_at: timestamp
}
```

### 1.4 PaymentTx
```typescript
{
  tx_id: uuid (primary key)
  wa_id: string (foreign key)
  saved_biller_id: uuid (nullable - null for one-time payments)
  category: string
  biller_id: string
  biller_name: string
  consumer_params_masked: string
  amount: decimal
  status: string (INITIATED|PENDING|SUCCESS|FAILED|REFUNDED)
  payment_method: string (UPI_INTENT|WHATSAPP_PAY|UPI_COLLECT)
  bbps_ref: string (nullable)
  utr: string (nullable)
  failure_reason: string (nullable)
  receipt_url: string (nullable)
  bill_period: string (nullable - e.g., "Dec 2025")
  due_date: date (nullable)
  created_at: timestamp
  updated_at: timestamp
  paid_at: timestamp (nullable)
}
```

### 1.5 SupportTicket
```typescript
{
  ticket_id: string (primary key, e.g., "SETU20260104001")
  wa_id: string (foreign key)
  issue_type: string (BILL_NOT_FETCHED|PAYMENT_FAILED|PAYMENT_PENDING|WRONG_BILL|REFUND|AUTOPAY_ISSUE|OTHER)
  tx_id: uuid (nullable)
  description: text
  attachments: jsonb (array of media URLs)
  status: string (OPEN|IN_PROGRESS|RESOLVED|CLOSED)
  assigned_to: string (nullable)
  resolution_notes: text (nullable)
  created_at: timestamp
  updated_at: timestamp
  resolved_at: timestamp (nullable)
}
```

### 1.6 ConversationState (for session management)
```typescript
{
  wa_id: string (primary key)
  current_state: string (HOME|PAY_BILL_SELECT_CATEGORY|etc.)
  context: jsonb (session data like selected_biller, entered_params)
  last_interaction_at: timestamp
  updated_at: timestamp
}
```

---

## 2. CONVERSATION ARCHITECTURE

### 2.1 State Machine Overview

```
┌─────────────────────────────────────────────────────┐
│                    HOME MENU                        │
│  (Interactive List - 5 sections)                    │
└──┬──────────┬──────────┬──────────┬────────────────┘
   │          │          │          │          │
   v          v          v          v          v
PAY_BILL  SAVED_BILLS REMINDERS  TRACK    HELP
```

### 2.2 Message Type Legend
- **Text**: Plain text message
- **Reply Buttons**: Up to 3 quick reply buttons
- **Interactive List**: Menu with sections and rows (max 10 rows)
- **Flow**: Multi-screen form with validation
- **Template**: Pre-approved message template (for >24h window)

---

## 3. FEATURE FLOWS

## 3.1 HOME MENU

**Trigger:** User sends any message, OR user types "menu", OR session timeout

**Message Type:** Interactive List

**Content:**
```
Welcome to Setu BillPay!

Choose an option:

SECTIONS:
- Payments
  - Pay a bill / Recharge
  - Pay saved bills (1-tap)

- Manage
  - Reminders & AutoPay
  - Track payment / Receipt

- Support
  - Help & FAQs
```

**State:** `HOME`

**Buttons (as list rows):**
1. Pay a bill
2. Pay saved bills
3. Reminders & AutoPay
4. Track payment
5. Help

**Edge Cases:**
- First-time user: Add language selection before showing menu
- Off-topic query: "I can help with bill payments and support. Here's what I can do:" → show menu
- User types "menu" anytime: return to HOME

---

## 3.2 FLOW A: PAY A BILL (New Payment)

### A.1 Language Selection (First Time Only)

**State:** `ONBOARDING_LANGUAGE`

**Message Type:** Reply Buttons

**Content:**
```
Welcome to Setu BillPay!

Choose your preferred language:
```

**Buttons:**
1. English
2. हिंदी (Hindi)
3. ಕನ್ನಡ (Kannada)

**Next:** → `PAY_BILL_SELECT_CATEGORY`

---

### A.2 Category Selection

**State:** `PAY_BILL_SELECT_CATEGORY`

**Message Type:** Interactive List

**Content:**
```
Which bill would you like to pay?

Select category:
```

**Sections:**
- Popular
  1. Electricity
  2. FASTag
  3. Loan Repayment
  4. Mobile Postpaid
  5. LPG Gas
  6. Credit Card

- More Categories
  7. Insurance
  8. DTH
  9. Water
  10. Broadband

**Button:** "View all categories" → opens Flow with full list + search

**Next:** → `PAY_BILL_SELECT_BILLER`

**Edge Cases:**
- User types category name: fuzzy match and proceed
- User types "back": → HOME

---

### A.3 Biller Selection

**State:** `PAY_BILL_SELECT_BILLER`

**Message Type:** WhatsApp Flow

**Flow Name:** `biller_select_flow`

**Screens:**

**Screen 1: Search or Browse**
```
Fields:
- Search input (text, placeholder: "Search biller name or IFSC/CPID")
- Recent billers (if user has payment history)
- Popular billers list (top 10 for category)

Validation:
- Min 3 chars for search
- Show "No results" if empty

CTA:
- Select button per biller row
```

**Selected Output:**
```json
{
  "biller_id": "MSEDCL001",
  "biller_name": "Maharashtra State Electricity Distribution Co Ltd",
  "required_params": ["consumer_number", "mobile"]
}
```

**Next:** → `PAY_BILL_ENTER_DETAILS`

**Edge Cases:**
- Biller offline: Show "This biller is temporarily unavailable. Try another or check back later."
- User exits flow: → HOME

---

### A.4 Enter Consumer Details

**State:** `PAY_BILL_ENTER_DETAILS`

**Message Type:** WhatsApp Flow

**Flow Name:** `enter_consumer_details_flow`

**Screen 1: Consumer Details**
```
Dynamic fields based on biller:
- Consumer Number (text, required, pattern: /^\d{10,15}$/)
- Mobile Number (text, required, pattern: /^[6-9]\d{9}$/)
- [Other biller-specific params]

Help text: "This info is used to fetch your bill securely"

CTA: Fetch Bill
```

**Validation:**
- Inline validation on each field
- Show examples: "e.g., 1234567890"
- Required field markers

**Output:**
```json
{
  "consumer_number": "1234567890",
  "mobile": "9876543210"
}
```

**API Call:** `fetchBill(biller_id, consumer_params)`

**Loading State:** Show "Fetching your bill..." (text message)

**Next:** → `PAY_BILL_PREVIEW`

**Edge Cases:**
- Fetch failed (invalid consumer):
  ```
  Unable to fetch bill. Please check:
  - Consumer number is correct
  - Bill is generated by biller

  [Try Again] [Change Biller] [Menu]
  ```
- Fetch failed (biller down):
  ```
  This biller is temporarily down. We'll notify you when it's back.

  [Try Another Biller] [Menu]
  ```
- Timeout (>30s): "Taking longer than usual. We'll send you a notification when bill is ready."

---

### A.5 Bill Preview

**State:** `PAY_BILL_PREVIEW`

**Message Type:** Text + Reply Buttons

**Content:**
```
✅ Bill fetched successfully

Biller: Maharashtra State Electricity
Consumer: XXXX7890
Name: Raj K***mar

Bill Amount: ₹1,245
Due Date: 10 Jan 2026
Bill Period: Dec 2025

What would you like to do?
```

**Buttons:**
1. Pay Now
2. Edit Details
3. Back to Menu

**Next:**
- Pay Now → `PAY_BILL_PAYMENT`
- Edit → Go back to A.4
- Menu → HOME

**Edge Cases:**
- Bill already paid: Show "This bill appears to be paid. Amount: ₹0" + buttons [Fetch Again] [Menu]
- Overdue bill: Add warning emoji and text: "⚠️ This bill is overdue. Late fee may apply."

---

### A.6 Payment Execution

**State:** `PAY_BILL_PAYMENT`

**Message Type:** Text + Dynamic (UPI Intent / WhatsApp Pay)

**Content:**
```
Initiating payment...

Amount: ₹1,245
Biller: Maharashtra State Electricity
Consumer: XXXX7890

Payment methods:
```

**Buttons / Actions:**
1. Pay with UPI (opens UPI intent link: `upi://pay?pa=...`)
2. Pay with WhatsApp Pay (if available in region)

**Process:**
1. Create `PaymentTx` record with status=INITIATED
2. Generate payment link/intent
3. User completes payment in UPI app
4. Webhook receives payment status
5. Update `PaymentTx` status

**Loading State:**
```
⏳ Payment in progress...

We'll notify you once it's confirmed.
You can check status anytime by typing "track"
```

**Next:** → `PAY_BILL_SUCCESS` or `PAY_BILL_PENDING` or `PAY_BILL_FAILED`

---

### A.7 Payment Success

**State:** `PAY_BILL_SUCCESS`

**Message Type:** Text + Reply Buttons

**Content:**
```
✅ Payment Successful!

Amount: ₹1,245
Biller: Maharashtra State Electricity
Consumer: XXXX7890
UTR: 402912345678
Tx ID: SETU20260104001

Receipt: [Download Link]

Would you like to:
```

**Buttons:**
1. Save this biller (for 1-tap payments)
2. Set reminder
3. View receipt
4. Back to menu

**API Call:** `generateReceipt(tx_id)` → store receipt_url

**Next Actions:**
- Save biller → Flow `save_biller_flow` (optional nickname + autopay consent)
- Set reminder → Flow `reminder_setup_flow`
- View receipt → Send receipt as PDF attachment
- Menu → HOME

**Template Trigger:** If payment completes >24h after last user message, send template:
```
Template: payment_success_notification
Variables: {biller_name}, {amount}, {tx_id}, {receipt_url}
CTA Button: "View Receipt" (opens WhatsApp chat)
```

---

### A.8 Payment Pending

**State:** `PAY_BILL_PENDING`

**Message Type:** Text + Reply Buttons

**Content:**
```
⏳ Payment Pending

Amount: ₹1,245
Biller: Maharashtra State Electricity
Tx ID: SETU20260104001

Your payment is being processed. This usually takes 5-10 minutes.

We'll send you a notification once it's confirmed.
```

**Buttons:**
1. Check Status Again
2. Track Payment
3. Need Help

**Background Process:**
- Poll payment gateway every 2 mins (max 30 mins)
- When status updates, send template notification

**Template (when resolved):**
```
Template: payment_status_update
Variables: {status}, {biller_name}, {amount}, {tx_id}
CTA: "View Details"
```

**Edge Cases:**
- Pending >30 mins: Auto-escalate to support team, send template with ticket ID

---

### A.9 Payment Failed

**State:** `PAY_BILL_FAILED`

**Message Type:** Text + Reply Buttons

**Content:**
```
❌ Payment Failed

Amount: ₹1,245
Biller: Maharashtra State Electricity
Tx ID: SETU20260104001

Reason: {failure_reason}

Your money is safe. No amount was debited.
```

**Buttons:**
1. Retry Payment
2. Try Different UPI
3. Raise Ticket

**Common Failure Reasons:**
- User cancelled payment
- Insufficient balance
- UPI PIN incorrect
- Bank server down
- Transaction timeout

**Next:**
- Retry → Go back to A.6 with same bill details
- Raise Ticket → Help flow E.7

---

## 3.3 FLOW B: PAY SAVED BILLS (1-Tap)

### B.1 Saved Billers List

**State:** `SAVED_BILLS_LIST`

**Message Type:** Interactive List

**Pre-check:**
- If no saved billers: "You don't have any saved billers yet. Let's add one!" → redirect to Flow A

**Content:**
```
Your saved billers

Choose a biller to pay:
```

**Sorting Logic:**
1. Bills with due dates in next 7 days (ascending)
2. Bills last fetched >30 days ago
3. Alphabetical by nickname/biller name

**List Rows (max 10):**
```
Row format:
Title: {nickname or biller_name}
Description: {consumer_masked} • Due: {due_date or "Fetch bill"}
```

**Footer Button:** "Manage saved billers" → opens Flow

**Next:** → `SAVED_BILL_FETCH`

**Edge Cases:**
- User has >10 saved billers: Show top 10 + "View all" button that opens Flow with search
- No recent bills: Show message "Fetch fresh bills to see latest dues"

---

### B.2 Auto-Fetch Selected Bill

**State:** `SAVED_BILL_FETCH`

**Message Type:** Text (auto-triggered)

**Content:**
```
Fetching bill for {biller_name}...
```

**API Call:** `fetchBill(biller_id, decrypted_consumer_params)`

**Process:**
1. Retrieve SavedBiller record
2. Decrypt consumer_params
3. Call fetchBill API
4. Update last_fetch fields in SavedBiller
5. Show preview

**Next:** → `SAVED_BILL_PREVIEW`

**Edge Cases:**
- Fetch failed (invalid consumer):
  ```
  ❌ Unable to fetch bill

  This consumer number may have changed or is invalid.

  [Update Details] [Try Again] [Menu]
  ```
  - Update Details → Flow `edit_saved_biller_flow`

- Fetch failed (biller down): Same as A.4 edge case

- Consumer no longer valid (moved/closed):
  ```
  This consumer account is no longer active.

  [Delete Biller] [Update Details] [Menu]
  ```

---

### B.3 Saved Bill Preview

**State:** `SAVED_BILL_PREVIEW`

**Message Type:** Text + Reply Buttons

**Content:**
```
✅ Bill ready

Biller: {biller_name} ({nickname})
Consumer: {consumer_masked}

Bill Amount: ₹1,245
Due Date: 10 Jan 2026
Bill Period: Dec 2025

Quick actions:
```

**Buttons:**
1. Pay Now (primary)
2. Fetch Again
3. More Options

**More Options (submenu):**
- Edit saved details
- Delete biller
- Set reminder
- Enable AutoPay
- Back

**Next:**
- Pay Now → Flow A.6 (Payment Execution)
- More Options → Show secondary menu

**Edge Cases:**
- Bill already paid: "✅ This bill is already paid (₹0 due)" + buttons [Fetch Again] [Menu]
- AutoPay enabled for this biller: Add note "🔄 AutoPay is ON for this biller"

---

### B.4 Manage Saved Billers

**State:** `SAVED_BILLS_MANAGE`

**Message Type:** WhatsApp Flow

**Flow Name:** `manage_saved_billers_flow`

**Screen 1: Billers List with Actions**
```
List of all saved billers with:
- Nickname (editable inline)
- Consumer masked
- Last fetched date
- Actions per row:
  - [Edit] [Delete] [AutoPay: ON/OFF toggle]

Search bar at top

CTA:
- Add New Biller (redirects to Flow A)
- Done
```

**Screen 2: Edit Biller Details** (opens on Edit click)
```
Fields:
- Nickname (text, optional)
- Consumer params (pre-filled, editable)
- AutoPay toggle
- AutoPay cap amount (if AutoPay ON)

CTA: Save Changes
```

**Screen 3: Confirm Delete**
```
Are you sure you want to delete {biller_name}?

This will also remove:
- Associated reminders
- AutoPay settings

[Cancel] [Yes, Delete]
```

**Edge Cases:**
- Delete biller with autopay: Show warning + confirm
- Edit consumer params: Re-validate with fetchBill before saving

---

## 3.4 FLOW C: REMINDERS & AUTOPAY

### C.1 Reminders Menu

**State:** `REMINDERS_MENU`

**Message Type:** Interactive List

**Content:**
```
Reminders & AutoPay

Manage your bill reminders and automatic payments:
```

**Sections:**
- Reminders
  1. Set a new reminder
  2. View & manage reminders ({count} active)

- AutoPay
  3. Enable AutoPay / SmartPay
  4. Manage AutoPay settings

- Settings
  5. Notification preferences

**Next:** Based on selection

---

### C.2 Set a New Reminder

**State:** `REMINDER_SETUP`

**Message Type:** WhatsApp Flow

**Flow Name:** `reminder_setup_flow`

**Screen 1: Select Biller**
```
Choose a saved biller for reminder:

[List of saved billers without active reminders]

If no saved billers: "Save a biller first to set reminders"
→ Redirect to Flow A or B
```

**Screen 2: Reminder Type**
```
Choose reminder type:

○ Before due date
  Set reminder N days before bill due date

○ Fixed monthly date
  Set reminder on same date every month

[Next]
```

**Screen 3a: Due Date Reminder** (if "Before due date" selected)
```
Fields:
- Days before due date: [Dropdown: 1,2,3,5,7,10]
- Preferred time: [Time picker, default: 10:00 AM]

Info: "We'll remind you {N} days before your {biller_name} bill is due"

[Set Reminder]
```

**Screen 3b: Monthly Reminder** (if "Fixed monthly date")
```
Fields:
- Date of month: [Dropdown: 1-31]
- Preferred time: [Time picker]

Info: "We'll remind you on the {date} of every month"

[Set Reminder]
```

**Screen 4: Consent**
```
✅ Reminder set successfully!

You'll receive reminders via WhatsApp.

Note: Reminders are sent as templates (outside 24hr window)

[Done] [Set Another]
```

**Database Actions:**
1. Create Reminder record
2. Schedule next_run_at timestamp
3. Setup background job to send template

**Template Used:**
```
Template: bill_reminder_due_date
Variables: {biller_nickname}, {days_before}, {estimated_amount}
CTA: "Pay Now" (opens Flow with prefilled biller)
```

**Edge Cases:**
- User selects biller that already has reminder: "You already have a reminder for this biller. Manage existing?"
- User has >10 saved billers: Show search + pagination in flow

---

### C.3 Manage Reminders

**State:** `REMINDER_MANAGE`

**Message Type:** WhatsApp Flow

**Flow Name:** `reminder_manage_flow`

**Screen 1: Active Reminders List**
```
Your active reminders:

[For each reminder:]
Card:
  Biller: {nickname}
  Type: {Due date - 3 days / Monthly - 5th}
  Time: {10:00 AM}
  Status: {Active / Snoozed until {date}}

  Actions:
  - [Edit] [Snooze] [Disable] [Delete]

If no reminders: "No active reminders. Set one now?"

CTA: [Add New Reminder] [Done]
```

**Screen 2: Edit Reminder**
```
Same as C.2 Screen 3 but pre-filled
```

**Screen 3: Snooze Reminder**
```
Snooze reminder until:

○ Next week
○ Next month
○ Custom date [Date picker]

[Snooze] [Cancel]
```

**Snooze Logic:**
- Update `snooze_until` timestamp
- Skip reminder execution until date passes
- Show "Snoozed" badge in list

**Edge Cases:**
- Delete reminder with AutoPay: Show warning "This biller has AutoPay. Reminder is independent."
- Disable vs Delete: Disable keeps record, Delete removes permanently

---

### C.4 Enable AutoPay / SmartPay

**State:** `AUTOPAY_SETUP`

**Message Type:** WhatsApp Flow

**Flow Name:** `autopay_setup_flow`

**Screen 1: Select Biller**
```
Choose a biller for AutoPay:

[List of saved billers without AutoPay]

If no saved billers: "Save a biller first"
```

**Screen 2: AutoPay Type**
```
Choose AutoPay mode:

○ Full AutoPay
  Automatically pay bills on due date
  Requires mandate setup (one-time)

○ SmartPay
  Get 1-tap payment notification on due date
  No mandate needed
  You approve each payment

[Next]
```

**Screen 3a: Full AutoPay Setup** (if selected)
```
Full AutoPay Setup

This will automatically pay your bills.

Fields:
- Maximum amount per payment: ₹[input, default: 5000]
- Payment method: [UPI Autopay Mandate]

Info:
"You'll set up a UPI mandate next. Bills exceeding cap will require manual approval."

[Setup Mandate] [Cancel]
```

**Mandate Setup Process:**
1. Generate UPI mandate link (mock: `upi://mandate?...`)
2. User approves in UPI app
3. Store mandate_id in SavedBiller
4. Confirm setup

**Screen 3b: SmartPay Setup**
```
SmartPay Setup

We'll notify you when bill is due.

Fields:
- Auto-fetch bill: [Toggle ON/OFF]
- Notification time: [Time picker, default: 10:00 AM]

Info:
"You'll get a 1-tap payment message on due date"

[Enable SmartPay]
```

**Screen 4: Confirmation**
```
✅ {AutoPay/SmartPay} enabled!

Biller: {biller_name}
Mode: {Full AutoPay / SmartPay}
Cap: {₹5000}

Your bills will be {auto-paid / sent for approval} automatically.

[Done] [Enable for Another Biller]
```

**Database Actions:**
1. Update SavedBiller: is_autopay_enabled=true, autopay_type, autopay_cap_amount
2. For SmartPay: schedule template on due date
3. For AutoPay: schedule payment execution job

**Template Used for SmartPay:**
```
Template: smartpay_bill_ready
Variables: {biller_name}, {amount}, {due_date}
CTA: "Pay Now" (1-tap, opens payment with prefilled details)
```

**Edge Cases:**
- Mandate setup fails: "Unable to setup mandate. Try again or use SmartPay"
- User already has AutoPay on biller: "AutoPay is already enabled. Manage settings?"
- Bill amount > cap: Send template for manual approval instead of auto-pay

---

### C.5 Manage AutoPay

**State:** `AUTOPAY_MANAGE`

**Message Type:** WhatsApp Flow

**Flow Name:** `autopay_manage_flow`

**Screen 1: AutoPay List**
```
Billers with AutoPay:

[For each autopay-enabled biller:]
Card:
  Biller: {nickname}
  Mode: {Full AutoPay / SmartPay}
  Cap: {₹5000}
  Status: {Active / Paused}
  Last payment: {date}

  Actions:
  - [Edit Cap] [Pause] [Disable] [History]

If none: "No AutoPay enabled yet"

CTA: [Enable AutoPay] [Done]
```

**Screen 2: Edit Cap Amount**
```
Edit AutoPay limit

Biller: {biller_name}
Current cap: ₹{current}

New cap amount: ₹[input]

[Save] [Cancel]
```

**Screen 3: AutoPay History**
```
AutoPay history for {biller_name}

List of last 10 autopay transactions:
- Date
- Amount
- Status (Auto-paid / Manual approval needed)
- Receipt link

[Close]
```

**Pause vs Disable:**
- Pause: Temporary (can resume without mandate)
- Disable: Removes autopay, mandate stays in backend

**Edge Cases:**
- Disable AutoPay: "Are you sure? You'll need to setup mandate again if you re-enable."
- AutoPay failure: Show failure reason + option to retry or disable

---

### C.6 Notification Preferences

**State:** `NOTIFICATION_PREFS`

**Message Type:** WhatsApp Flow

**Flow Name:** `notification_prefs_flow`

**Screen 1: Preferences**
```
Notification Preferences

Choose what notifications you want:

Toggles:
☑ Bill reminders
☑ Payment confirmations
☑ Payment failures & pending updates
☑ AutoPay notifications
☑ Promotional offers

Preferred time for reminders:
[Time picker: HH:MM]

Language:
[Dropdown: English / Hindi / Kannada]

[Save Preferences]
```

**Database Actions:**
- Update UserProfile.notification_time_pref
- Update UserProfile.language

**Edge Cases:**
- User disables all notifications: Show warning "You won't receive any updates. Are you sure?"

---

## 3.5 FLOW D: TRACK PAYMENT / RECEIPT

### D.1 Track Menu

**State:** `TRACK_MENU`

**Message Type:** Interactive List

**Content:**
```
Track payments & receipts

How would you like to search?
```

**Options:**
1. Recent payments (last 10)
2. Search by biller
3. Enter transaction ID
4. Pending payments

**Next:** Based on selection

---

### D.2 Recent Payments

**State:** `TRACK_RECENT`

**Message Type:** Interactive List

**Query:** Fetch last 10 PaymentTx for wa_id, order by created_at DESC

**Content:**
```
Your recent payments:

[List rows:]
Title: {biller_name}
Description: ₹{amount} • {status} • {date}
```

**Row Click:** → `TRACK_DETAIL` for that tx_id

**Edge Cases:**
- No payment history: "No payments yet. Make your first payment!" → redirect to Flow A

---

### D.3 Search by Biller

**State:** `TRACK_SEARCH_BILLER`

**Message Type:** WhatsApp Flow

**Flow Name:** `track_search_biller_flow`

**Screen 1: Biller Search**
```
Search payments by biller

Fields:
- Search: [Text input with autocomplete from saved billers + past billers]

Results list:
[Show payments grouped by biller]

[Close]
```

---

### D.4 Enter Transaction ID

**State:** `TRACK_ENTER_TX_ID`

**Message Type:** Text (prompt)

**Content:**
```
Enter your transaction ID

Format: SETU20260104001
```

**User Input:** Plain text

**Validation:**
- Regex: `/^SETU\d{11}$/`
- Check if tx_id exists for this wa_id

**Next:** → `TRACK_DETAIL`

**Edge Cases:**
- Invalid format: "Invalid transaction ID. Please check and try again. Example: SETU20260104001"
- TX not found: "Transaction not found. Please check the ID or contact support."
- TX belongs to different user: "Transaction not found" (security: don't reveal existence)

---

### D.5 Payment Detail View

**State:** `TRACK_DETAIL`

**Message Type:** Text + Reply Buttons

**Content:**
```
Transaction Details

Tx ID: {tx_id}
Status: {status_emoji} {status}

Biller: {biller_name}
Consumer: {consumer_masked}
Amount: ₹{amount}
Bill Period: {bill_period}
Due Date: {due_date}

Payment Method: {payment_method}
Paid On: {paid_at}
UTR: {utr}
BBPS Ref: {bbps_ref}

Receipt: {receipt_url}
```

**Buttons (dynamic based on status):**

**If SUCCESS:**
1. Download Receipt
2. Share Receipt
3. Back

**If PENDING:**
1. Check Status Again
2. Need Help
3. Back

**If FAILED:**
1. View Failure Reason
2. Retry Payment
3. Raise Ticket
4. Back

**Actions:**
- Download Receipt: Send PDF as attachment
- Share Receipt: Generate shareable link
- Check Status Again: Re-query payment gateway, update DB, refresh view

**Template Trigger:**
If status changes from PENDING → SUCCESS/FAILED outside 24h window:
```
Template: payment_status_update
Variables: {tx_id}, {status}, {biller_name}, {amount}
CTA: "View Details"
```

---

### D.6 Pending Payments

**State:** `TRACK_PENDING`

**Message Type:** Interactive List

**Query:** Fetch PaymentTx where status=PENDING for wa_id

**Content:**
```
Pending Payments

These payments are being processed:

[List rows:]
Title: {biller_name}
Description: ₹{amount} • Initiated {time_ago}
```

**Row Click:** → `TRACK_DETAIL`

**Auto-refresh logic:**
- Background job polls pending payments
- When status updates, send template notification

**Edge Cases:**
- No pending payments: "No pending payments. All clear!"
- Pending >24 hours: Show warning + "Raise Ticket" button

---

## 3.6 FLOW E: HELP

### E.1 Help Menu

**State:** `HELP_MENU`

**Message Type:** Interactive List

**Content:**
```
Help & Support

How can we help you?
```

**Sections:**
- Common Issues
  1. Bill not fetched
  2. Payment failed but amount debited
  3. Payment pending for long time
  4. Wrong bill amount
  5. Request refund

- AutoPay Issues
  6. AutoPay not working
  7. Disable AutoPay

- Other
  8. Raise a ticket
  9. Talk to agent
  10. FAQs

**Footer:**
"Working hours: 9 AM - 9 PM IST"

**Next:** Based on selection

---

### E.2 Bill Not Fetched

**State:** `HELP_BILL_NOT_FETCHED`

**Message Type:** Text + Reply Buttons

**Content:**
```
Bill Not Fetched?

This usually happens when:
1. Consumer number is incorrect
2. Bill not yet generated by biller
3. Biller's system is temporarily down

Try these steps:
✓ Double-check your consumer number
✓ Check if bill is generated on biller's website
✓ Try again after 1-2 hours

Still stuck?
```

**Buttons:**
1. Try Fetching Again
2. Raise Ticket
3. Back to Help

---

### E.3 Payment Failed But Amount Debited

**State:** `HELP_PAYMENT_FAILED_DEBITED`

**Message Type:** Text + Reply Buttons

**Content:**
```
Amount Debited but Payment Failed?

Don't worry! This happens rarely. Here's what we'll do:

1. Check your bank statement (can take 30 mins to reflect)
2. If amount is debited but payment failed in our system, it will auto-refund in 5-7 business days
3. We'll track this for you

Do you have a transaction ID?
```

**Buttons:**
1. Yes, I have Tx ID → Ask for TX ID, then auto-raise ticket with priority=HIGH
2. No Tx ID → Open ticket form
3. Back

**Auto-actions:**
- Query payment gateway for reconciliation
- Flag transaction for manual review
- Send confirmation with expected refund date

---

### E.4 Payment Pending

**State:** `HELP_PAYMENT_PENDING`

**Message Type:** Text + Reply Buttons

**Content:**
```
Payment Pending?

Pending payments usually complete within:
• 5-10 minutes (most cases)
• Up to 2 hours (some banks)
• Up to 24 hours (rare cases)

We'll notify you once it's confirmed.

Current pending duration: {time_since_payment}

Want to check status now?
```

**Buttons:**
1. Check Status → Fetch from gateway and show update
2. Raise Ticket (if >2 hours)
3. Back

---

### E.5 Wrong Bill Amount

**State:** `HELP_WRONG_BILL`

**Message Type:** Text + Reply Buttons

**Content:**
```
Wrong Bill Amount?

Bill amounts come directly from the biller's system (BBPS). We fetch the exact amount the biller has generated.

If you believe it's incorrect:
1. Check your bill on the biller's official website
2. Contact the biller directly to dispute the amount
3. Once corrected, fetch bill again here

Biller Customer Care: {biller_support_contact}

Need further help?
```

**Buttons:**
1. Fetch Bill Again
2. Raise Ticket
3. Back

---

### E.6 Request Refund

**State:** `HELP_REFUND`

**Message Type:** Text + Reply Buttons

**Content:**
```
Request a Refund

Common refund scenarios:
• Payment failed but debited
• Bill paid twice by mistake
• Biller rejected payment

Refunds typically take 5-7 business days.

Do you want to raise a refund request?
```

**Buttons:**
1. Yes, Request Refund → Open ticket form with issue_type=REFUND
2. Check Refund Status → Ask for TX ID, show refund status
3. Back

---

### E.7 Raise a Ticket

**State:** `HELP_RAISE_TICKET`

**Message Type:** WhatsApp Flow

**Flow Name:** `support_ticket_flow`

**Screen 1: Issue Details**
```
Raise a Support Ticket

Fields:
- Issue type: [Dropdown]
  • Bill not fetched
  • Payment failed
  • Payment pending >24h
  • Wrong bill
  • Refund request
  • AutoPay issue
  • Other

- Related transaction: [Optional, search recent TXs or enter TX ID]

- Description: [Text area, required, min 10 chars]
  Placeholder: "Describe your issue in detail"

- Attach screenshot (optional): [File upload, images only]

[Submit Ticket]
```

**Screen 2: Confirmation**
```
✅ Ticket Created

Ticket ID: {ticket_id}
Status: Open

Our team will review and respond within 4-6 hours.

You can check status anytime by typing:
"ticket {ticket_id}"

Need urgent help?
[Call Us] [Email Us] [Back to Menu]
```

**Database Actions:**
1. Create SupportTicket record
2. Send confirmation template (if >24h window)
3. Alert support team (internal system)

**Template:**
```
Template: ticket_created
Variables: {ticket_id}, {issue_type}
CTA: "Track Ticket"
```

**Edge Cases:**
- User uploads invalid file: "Please upload images only (JPG, PNG)"
- Description too short: "Please provide more details (min 10 characters)"

---

### E.8 Talk to Agent

**State:** `HELP_TALK_TO_AGENT`

**Message Type:** Text + Reply Buttons

**Content:**
```
Talk to Support Agent

Our agents are available:
🕒 Mon-Sat: 9 AM - 9 PM IST
🕒 Sun: 10 AM - 6 PM IST

Choose how to connect:
```

**Buttons:**
1. Call Us → Show phone number: +91-XXX-XXX-XXXX
2. Email Us → Show email: support@setu.co
3. Chat with Agent (if available) → Transfer to human agent in WhatsApp
4. Back

**Human Agent Handoff:**
- If within business hours + agents available: "Connecting you to an agent..."
- If outside hours: "Our agents are offline. Leave a message or raise a ticket?"

---

### E.9 FAQs

**State:** `HELP_FAQS`

**Message Type:** Interactive List

**Content:**
```
Frequently Asked Questions

Browse common questions:
```

**Categories:**
1. Getting Started
2. Payments & Refunds
3. Saved Billers & AutoPay
4. Reminders
5. Security & Privacy

**Each category:** Opens a flow with Q&A accordion

**Sample FAQs:**
- Is my data secure? → "Yes, we use bank-grade encryption..."
- How do reminders work? → "Reminders are sent via WhatsApp templates..."
- Can I delete my data? → "Yes, go to Help > Privacy > Delete all data"

---

### E.10 Privacy & Data Management

**State:** `HELP_PRIVACY`

**Message Type:** Interactive List

**Content:**
```
Privacy & Data Management

Manage your data:
```

**Options:**
1. View data stored → Show summary of saved billers, payments count
2. Delete saved billers → Confirm and delete all SavedBiller records
3. Delete payment history → Confirm and delete PaymentTx records
4. Delete all data & stop messages → Full account deletion + unsubscribe
5. Back

**Delete All Data Flow:**
```
Are you sure you want to delete all data?

This will permanently remove:
• All saved billers
• Payment history
• Reminders and AutoPay settings
• Support tickets

This cannot be undone.

Type "DELETE" to confirm:
[Text input]

[Cancel] [Confirm]
```

**Actions:**
- Soft delete: Mark user as inactive, stop all messages
- Hard delete: Comply with GDPR-style data deletion (7-day grace period)

---

## 4. MESSAGE TEMPLATES LIBRARY

### 4.1 Template Naming Convention
Format: `{event}_{context}_{action}`

### 4.2 Template Definitions

#### T1: bill_reminder_due_date
**Category:** Utility
**Language:** EN, HI, KN

**Content (EN):**
```
🔔 Bill Reminder

Your {{biller_nickname}} bill is due in {{days_before}} days.

Estimated amount: ₹{{estimated_amount}}
Due date: {{due_date}}

[Pay Now]
```

**Variables:**
- biller_nickname (text)
- days_before (number)
- estimated_amount (number)
- due_date (date)

**CTA Button:** "Pay Now" → Opens WhatsApp chat with prefilled context (biller_id in deep link)

---

#### T2: bill_reminder_monthly
**Category:** Utility

**Content (EN):**
```
🔔 Monthly Bill Reminder

Time to pay your {{biller_nickname}} bill!

Last amount: ₹{{last_amount}}

[Pay Now]
```

**Variables:**
- biller_nickname (text)
- last_amount (number)

**CTA:** "Pay Now"

---

#### T3: payment_success_notification
**Category:** Utility

**Content (EN):**
```
✅ Payment Successful

Amount: ₹{{amount}}
Biller: {{biller_name}}
Tx ID: {{tx_id}}

[View Receipt]
```

**Variables:**
- amount (number)
- biller_name (text)
- tx_id (text)

**CTA:** "View Receipt" → Opens chat, sends receipt as attachment

---

#### T4: payment_status_update
**Category:** Utility

**Content (EN):**
```
{{status_emoji}} Payment {{status_text}}

Amount: ₹{{amount}}
Biller: {{biller_name}}
Tx ID: {{tx_id}}

[View Details]
```

**Variables:**
- status_emoji (✅ or ❌ or ⏳)
- status_text (text: "Successful" / "Failed" / "Pending")
- amount (number)
- biller_name (text)
- tx_id (text)

**CTA:** "View Details"

---

#### T5: payment_pending_update
**Category:** Utility

**Content (EN):**
```
⏳ Payment Still Pending

Your payment of ₹{{amount}} to {{biller_name}} is taking longer than usual.

Tx ID: {{tx_id}}

We're tracking it. You'll be notified once confirmed.

[Check Status]
```

---

#### T6: smartpay_bill_ready
**Category:** Utility

**Content (EN):**
```
💡 SmartPay: Bill Ready

Your {{biller_name}} bill is due today.

Amount: ₹{{amount}}
Due date: {{due_date}}

Tap to pay instantly ↓

[Pay Now]
```

**CTA:** "Pay Now" → Opens payment with prefilled details (1-tap experience)

---

#### T7: autopay_success
**Category:** Utility

**Content (EN):**
```
✅ AutoPay Successful

Your {{biller_name}} bill was paid automatically.

Amount: ₹{{amount}}
Tx ID: {{tx_id}}

[View Receipt]
```

---

#### T8: autopay_failed
**Category:** Utility

**Content (EN):**
```
❌ AutoPay Failed

We couldn't auto-pay your {{biller_name}} bill.

Reason: {{failure_reason}}
Amount: ₹{{amount}}

Please pay manually to avoid late fees.

[Pay Now]
```

---

#### T9: autopay_approval_needed
**Category:** Utility

**Content (EN):**
```
⚠️ AutoPay: Approval Needed

Your {{biller_name}} bill exceeds AutoPay limit.

Bill amount: ₹{{amount}}
Your cap: ₹{{cap_amount}}

[Pay Now] [Skip]
```

---

#### T10: ticket_created
**Category:** Utility

**Content (EN):**
```
✅ Support Ticket Created

Ticket ID: {{ticket_id}}
Issue: {{issue_type}}

Our team will respond within 4-6 hours.

[Track Ticket]
```

---

#### T11: ticket_resolved
**Category:** Utility

**Content (EN):**
```
✅ Ticket Resolved

Ticket ID: {{ticket_id}}

Resolution: {{resolution_summary}}

[View Details] [Rate Support]
```

---

#### T12: biller_back_online
**Category:** Utility

**Content (EN):**
```
✅ Biller Back Online

{{biller_name}} is now available for payments.

[Pay Bill]
```

**Trigger:** When a previously offline biller comes back online and user had attempted to pay

---

## 5. WHATSAPP FLOWS SPECIFICATIONS

### 5.1 Flow Design Principles
- Max 3 screens per flow (WhatsApp best practice)
- Clear progress indicators
- Inline validation with helpful error messages
- Exit button on every screen
- Responsive layout for mobile

### 5.2 Flow Catalog

#### F1: biller_select_flow
**Purpose:** Search and select biller from large catalog

**Screens:**
1. Search/Browse (1 screen)
   - Search input (min 3 chars)
   - Popular billers grid (6 items)
   - Recent billers (if exists)
   - Category filter chips

**Data Model:**
```json
{
  "biller_id": "MSEDCL001",
  "biller_name": "Maharashtra State Electricity",
  "category": "ELECTRICITY",
  "required_params": ["consumer_number", "mobile"]
}
```

---

#### F2: enter_consumer_details_flow
**Purpose:** Collect consumer/account details for bill fetch

**Screens:**
1. Dynamic form based on biller requirements
   - Field types: text, number, dropdown
   - Validation patterns per field
   - Example hints
   - Help icon with tooltips

**Sample Fields:**
- Consumer Number (text, required, pattern varies by biller)
- Mobile Number (text, required, pattern: /^[6-9]\d{9}$/)
- Service Type (dropdown for some billers)

**Validation Examples:**
- Electricity consumer: 10-15 digits
- FASTag: Vehicle registration (e.g., MH01AB1234)
- Loan: Account number (alphanumeric)

---

#### F3: save_biller_flow
**Purpose:** Save biller for quick access + optional AutoPay consent

**Screens:**
1. Save Details
   - Nickname (optional, text, max 30 chars)
   - "Remember this biller for 1-tap payments?" (checkbox, default ON)
   - "Enable SmartPay notifications?" (checkbox, default OFF)

2. Confirmation
   - "Biller saved!" + next actions

---

#### F4: edit_saved_biller_flow
**Purpose:** Edit saved biller details

**Screens:**
1. Edit Form
   - Nickname (editable)
   - Consumer params (editable with validation)
   - AutoPay toggle
   - Last fetch status display

2. Confirm changes
   - Show diff if consumer params changed
   - "Re-fetch bill to verify?" option

---

#### F5: reminder_setup_flow
**Purpose:** Configure reminder for a biller

**Screens:**
1. Select biller (if not pre-selected)
2. Reminder type & schedule (see C.2)
3. Confirmation

---

#### F6: reminder_manage_flow
**Purpose:** View and manage all reminders

**Screens:**
1. Reminders list with actions (see C.3)
2. Edit/Snooze/Delete sub-screens

---

#### F7: autopay_setup_flow
**Purpose:** Enable AutoPay or SmartPay

**Screens:**
1. Select biller
2. Choose mode (AutoPay vs SmartPay)
3. Configure cap & mandate (see C.4)

---

#### F8: autopay_manage_flow
**Purpose:** Manage existing AutoPay settings

**Screens:**
1. AutoPay list with actions (see C.5)
2. Edit/Pause/Disable sub-screens

---

#### F9: track_search_flow
**Purpose:** Search payments by various criteria

**Screens:**
1. Search input with filters
   - By biller (dropdown)
   - By date range (date pickers)
   - By status (checkboxes)
2. Results list

---

#### F10: support_ticket_flow
**Purpose:** Create support ticket

**Screens:**
1. Issue details form (see E.7)
2. Confirmation with ticket ID

---

#### F11: manage_saved_billers_flow
**Purpose:** Bulk manage all saved billers

**Screens:**
1. Full list with search + actions (see B.4)
2. Edit/Delete sub-screens

---

#### F12: notification_prefs_flow
**Purpose:** Configure notification preferences

**Screens:**
1. Preferences form (see C.6)

---

## 6. EDGE CASES & FALLBACK HANDLING

### 6.1 User Input Edge Cases

| User Input | Bot Response | Next Action |
|------------|--------------|-------------|
| Random text (not a command) | "I can help with bill payments and support. Type 'menu' to see options." | Show HOME menu |
| Off-topic question ("Weather?") | "I specialize in bill payments. For other queries, please type 'menu' for options." | Show HOME menu |
| "Hello" / "Hi" | "Hi! Welcome to Setu BillPay. How can I help you today?" + Show HOME menu | HOME state |
| "Help" | Show Help menu | HELP_MENU state |
| "Menu" | Show HOME menu | HOME state |
| "Stop" / "Unsubscribe" | "Sorry to see you go. You'll stop receiving notifications. Type 'start' anytime to resume." | Disable notifications |
| Abuse / profanity | "I'm here to help with bill payments. Please use respectful language." | Continue current state |
| Multiple messages rapidly | Debounce: process last message only | Continue flow |
| Long message (>1000 chars) | "Please keep messages concise. What would you like to do?" + Show menu | HOME state |

---

### 6.2 API Failure Scenarios

#### A. fetchBill API Failures

| Error Type | User Message | Buttons | Background Action |
|------------|--------------|---------|-------------------|
| Invalid consumer | "Unable to fetch bill. Please check: ✓ Consumer number is correct ✓ Bill is generated by biller" | [Try Again] [Change Biller] [Menu] | Log error |
| Biller down | "{{biller_name}} is temporarily unavailable. We'll notify you when it's back." | [Try Another Biller] [Menu] | Track biller status, send template when up (T12) |
| Timeout (>30s) | "Taking longer than usual. We'll send a notification when bill is ready. Continue?" | [Wait] [Try Another] [Menu] | Async fetch, send template on completion |
| Network error | "Connection issue. Please try again." | [Retry] [Menu] | Retry with exponential backoff |
| Rate limit exceeded | "Too many requests. Please wait 2 minutes and try again." | [Menu] | Implement rate limiting per user |

---

#### B. Payment API Failures

| Error Type | User Message | Recovery Action |
|------------|--------------|-----------------|
| Payment gateway down | "Payment service temporarily unavailable. Please try again in 5 minutes." | Retry logic + notify when available |
| UPI app not responding | "UPI app not responding. Please complete payment in your UPI app or try again." | Show "Check Payment Status" button |
| Transaction timeout | "Payment timeout. Checking status..." → Show PENDING state | Auto-check status every 2 mins |
| Duplicate transaction | "A payment for this bill is already in progress (Tx ID: {{tx_id}})" | Show existing TX details |
| Insufficient balance (from bank) | "Payment failed: Insufficient balance. Please add funds and try again." | [Retry] [Menu] buttons |
| UPI PIN incorrect (3 attempts) | "Payment failed: Incorrect UPI PIN. Please try again carefully." | [Retry] button, lock after 3 attempts |
| Bank server down | "Your bank's server is down. Please try with a different bank or try later." | [Try Different UPI] [Menu] |

---

#### C. AutoPay Failures

| Scenario | User Notification (Template) | Action |
|----------|------------------------------|--------|
| Mandate expired | T8: autopay_failed (reason: "Mandate expired") | Ask user to renew mandate |
| Insufficient balance | T8: autopay_failed (reason: "Insufficient balance") | Send 1-tap payment link |
| Bill amount > cap | T9: autopay_approval_needed | Send approval request with [Pay] [Skip] |
| Biller down on due date | Retry for 24h, then send manual payment request | Track and retry |
| Mandate revoked by user | Disable AutoPay, send confirmation | Update SavedBiller.is_autopay_enabled=false |

---

### 6.3 Data Inconsistency Scenarios

| Issue | Detection | Resolution |
|-------|-----------|------------|
| SavedBiller with invalid consumer params | Fetch fails repeatedly | Show "Update details" prompt after 3 failures |
| Duplicate SavedBillers for same consumer | On save: check existing records | Ask "You have a saved biller for this. Update existing?" |
| Reminder with deleted SavedBiller | On reminder execution | Auto-delete reminder, don't send notification |
| PaymentTx stuck in INITIATED (>1 hour) | Background job checks | Auto-mark as FAILED, notify user to check bank statement |
| Orphan Reminder (no SavedBiller) | DB constraint / cleanup job | Delete reminder |

---

### 6.4 Race Conditions

| Scenario | Prevention | Handling |
|----------|------------|----------|
| User fetches same bill twice simultaneously | Lock on (wa_id, biller_id, consumer_params) | Show "Already fetching bill..." |
| User pays same bill twice | Lock on (wa_id, saved_biller_id) | Show "Payment in progress for this bill" |
| AutoPay and manual pay at same time | Check for existing INITIATED/PENDING tx before starting | Cancel autopay if manual pay succeeds |
| User deletes SavedBiller while AutoPay processing | Soft delete with grace period | Cancel scheduled autopay, show confirmation |

---

### 6.5 WhatsApp-Specific Edge Cases

| Issue | Handling |
|-------|----------|
| User sends message exactly at 24h boundary | Attempt free-form reply; if fails, fall back to template |
| Template delivery failed | Log error, retry after 1 hour, escalate to support if fails 3x |
| User blocks/unblocks bot | On unblock: send welcome back message, resume services |
| User changes phone number | Instruct to contact support for data migration |
| Flow interrupted (user exits) | Save partial state in ConversationState.context, offer to resume on next interaction |
| Media attachment fails to upload (in ticket) | Allow plain text submission, request media via follow-up |
| Interactive list doesn't render (old WhatsApp version) | Fall back to numbered text menu: "Reply with number: 1. Pay bill 2. Saved bills..." |

---

## 7. DATA MASKING & SECURITY

### 7.1 Data Masking Rules

| Field Type | Masking Pattern | Example |
|------------|-----------------|---------|
| Consumer Number (10-15 digits) | Show last 4 digits | XXXX1234 |
| Mobile (10 digits) | Show last 4 digits | XXXXX5678 |
| Account Number (alphanumeric) | Show first 2 + last 3 | AB***567 |
| FASTag (Vehicle Reg) | Show last 4 chars | ****1234 |
| Customer Name | Show first name + masked last | Raj K***mar |
| UPI VPA | Show first 3 + last 3 chars | raj***@okicici |
| Card Number | Show last 4 digits | **** **** **** 1234 |

### 7.2 Data Encryption

**Fields to Encrypt (at rest):**
- SavedBiller.consumer_params (AES-256)
- UserProfile.notification_time_pref (not critical but PII)
- SupportTicket.description (contains PII)

**Encryption Key Management:**
- Use Supabase encryption features or application-level encryption
- Rotate keys quarterly
- Store keys in secure vault (not in code)

### 7.3 Data Retention

| Data Type | Retention Period | Post-Retention Action |
|-----------|------------------|----------------------|
| PaymentTx | 7 years (regulatory) | Archive to cold storage |
| SupportTicket | 2 years | Archive |
| SavedBiller | Until user deletes or inactive >2 years | Soft delete |
| Reminder | Until user deletes or SavedBiller deleted | Hard delete |
| ConversationState | 30 days of inactivity | Hard delete |
| User consent logs | 7 years | Archive |

### 7.4 Compliance

- **PCI DSS:** Never store full card numbers, CVV, or UPI PINs
- **RBI Guidelines:** Follow BBPS security standards
- **GDPR-like (India DPDP Act):** Provide data export, deletion, and consent management
- **WhatsApp Policy:** Don't use WhatsApp data for cross-platform targeting

### 7.5 Access Control

- Supabase RLS policies on all tables (wa_id-based access)
- Support agents: limited read access to PaymentTx and SupportTicket (via admin panel, not direct DB)
- No raw consumer_params exposure in logs or analytics

---

## 8. DEMO TRANSCRIPT

**Scenario:** First-time user journey covering all major flows

---

**User:** Hi

**Bot:** (Interactive List - HOME MENU)
```
Welcome to Setu BillPay!

Choose an option:

PAYMENTS
1. Pay a bill / Recharge
2. Pay saved bills (1-tap)

MANAGE
3. Reminders & AutoPay
4. Track payment / Receipt

SUPPORT
5. Help & FAQs
```

---

**User:** [Clicks "Pay a bill"]

**Bot:** (Interactive List - Categories)
```
Which bill would you like to pay?

Select category:

POPULAR
1. Electricity
2. FASTag
3. Loan Repayment
4. Mobile Postpaid
5. LPG Gas
6. Credit Card

MORE CATEGORIES
7. Insurance
8. DTH
9. Water
10. Broadband
```

---

**User:** [Clicks "Electricity"]

**Bot:** (Opens WhatsApp Flow - biller_select_flow)
```
[FLOW SCREEN]
Search or select your electricity provider:

[Search: "Search biller..."]

POPULAR IN YOUR REGION
- Maharashtra State Electricity (MSEDCL)
- Tata Power - Mumbai
- Adani Electricity - Mumbai

[Select buttons per row]
```

---

**User:** [Selects "Maharashtra State Electricity"]

**Bot:** (Opens WhatsApp Flow - enter_consumer_details_flow)
```
[FLOW SCREEN]
Enter your MSEDCL details:

Consumer Number *
[____________]
e.g., 123456789012

Mobile Number *
[____________]
e.g., 9876543210

[Fetch Bill]
```

---

**User:** [Enters: 000012345678, 9876543210] [Clicks "Fetch Bill"]

**Bot:** (Text message - loading)
```
Fetching your bill...
```

**Bot:** (Text + Reply Buttons - Bill Preview)
```
✅ Bill fetched successfully

Biller: Maharashtra State Electricity
Consumer: XXXX5678
Name: Raj K***mar

Bill Amount: ₹1,245
Due Date: 10 Jan 2026
Bill Period: Dec 2025

What would you like to do?
```
**[Buttons: Pay Now | Edit Details | Back to Menu]**

---

**User:** [Clicks "Pay Now"]

**Bot:** (Text + UPI Intent)
```
Initiating payment...

Amount: ₹1,245
Biller: Maharashtra State Electricity
Consumer: XXXX5678

Tap below to pay via UPI:
```
**[Button: Pay with UPI]**

---

**User:** [Clicks "Pay with UPI", completes payment in UPI app]

**Bot:** (Text - loading)
```
⏳ Payment in progress...

We'll notify you once it's confirmed.
Tx ID: SETU20260104001
```

---

**[2 minutes later]**

**Bot:** (Text + Reply Buttons - Success)
```
✅ Payment Successful!

Amount: ₹1,245
Biller: Maharashtra State Electricity
Consumer: XXXX5678
UTR: 402912345678
Tx ID: SETU20260104001

Receipt: [Download Link]

Would you like to:
```
**[Buttons: Save this biller | Set reminder | View receipt | Back to menu]**

---

**User:** [Clicks "Save this biller"]

**Bot:** (Opens WhatsApp Flow - save_biller_flow)
```
[FLOW SCREEN]
Save biller for quick payments

Nickname (optional)
[____________]
e.g., "Home Electricity"

☑ Remember for 1-tap payments
☐ Enable SmartPay notifications

[Save Biller]
```

---

**User:** [Enters "Home Electricity", checks both boxes] [Clicks "Save Biller"]

**Bot:** (Text + Reply Buttons)
```
✅ Biller saved!

You can now pay this bill with 1-tap from "Pay saved bills"

Would you like to set a reminder?
```
**[Buttons: Yes, set reminder | No, back to menu]**

---

**User:** [Clicks "Yes, set reminder"]

**Bot:** (Opens WhatsApp Flow - reminder_setup_flow)
```
[FLOW SCREEN]
Set reminder for Home Electricity

Choose reminder type:

○ Before due date
  Remind me N days before bill is due

○ Fixed monthly date
  Remind me on same date every month

[Next]
```

---

**User:** [Selects "Before due date"] [Clicks "Next"]

**Bot:** (Flow - Screen 2)
```
[FLOW SCREEN]
Due Date Reminder

Days before due date:
[Dropdown: 3 days] ▼

Preferred time:
[Time: 10:00 AM] 🕒

Info: We'll remind you 3 days before your Home Electricity bill is due

[Set Reminder]
```

---

**User:** [Keeps defaults] [Clicks "Set Reminder"]

**Bot:** (Text)
```
✅ Reminder set successfully!

You'll receive reminders 3 days before due date at 10:00 AM.

Type 'menu' to continue.
```

---

**[One month later - Reminder Template sent]**

**Bot:** (Template Message - bill_reminder_due_date)
```
🔔 Bill Reminder

Your Home Electricity bill is due in 3 days.

Estimated amount: ₹1,280
Due date: 10 Feb 2026

[Pay Now]
```

---

**User:** [Clicks "Pay Now"]

**Bot:** (Text - auto-fetching)
```
Fetching bill for Home Electricity...
```

**Bot:** (Text + Reply Buttons - Bill Preview)
```
✅ Bill ready

Biller: Maharashtra State Electricity (Home Electricity)
Consumer: XXXX5678

Bill Amount: ₹1,280
Due Date: 10 Feb 2026
Bill Period: Jan 2026

Quick actions:
```
**[Buttons: Pay Now | Fetch Again | More Options]**

---

**User:** [Clicks "Pay Now"]

**Bot:** (Repeats payment flow as in first payment)

**[Payment completes successfully]**

**Bot:**
```
✅ Payment Successful!

Amount: ₹1,280
Biller: Maharashtra State Electricity
Consumer: XXXX5678
UTR: 402923456789
Tx ID: SETU20260207001

Receipt: [Download Link]
```
**[Buttons: View receipt | Set AutoPay | Back to menu]**

---

**User:** [Clicks "Back to menu"]

**Bot:** (Shows HOME menu)

---

**[Later that day]**

**User:** track

**Bot:** (Interactive List - Track Menu)
```
Track payments & receipts

How would you like to search?

1. Recent payments (last 10)
2. Search by biller
3. Enter transaction ID
4. Pending payments
```

---

**User:** [Clicks "Recent payments"]

**Bot:** (Interactive List - Recent Payments)
```
Your recent payments:

1. Maharashtra State Electricity
   ₹1,280 • SUCCESS • 07 Feb 2026

2. Maharashtra State Electricity
   ₹1,245 • SUCCESS • 04 Jan 2026
```

---

**User:** [Clicks first row]

**Bot:** (Text + Reply Buttons - Payment Detail)
```
Transaction Details

Tx ID: SETU20260207001
Status: ✅ Successful

Biller: Maharashtra State Electricity
Consumer: XXXX5678
Amount: ₹1,280
Bill Period: Jan 2026
Due Date: 10 Feb 2026

Payment Method: UPI
Paid On: 07 Feb 2026, 10:15 AM
UTR: 402923456789
BBPS Ref: BP20260207789

Receipt: [Download Link]
```
**[Buttons: Download Receipt | Share Receipt | Back]**

---

**User:** [Clicks "Back"]

**Bot:** (Shows Track Menu again)

**User:** menu

**Bot:** (Shows HOME menu)

---

**[Few days later - user has a payment issue]**

**User:** help

**Bot:** (Interactive List - Help Menu)
```
Help & Support

How can we help you?

COMMON ISSUES
1. Bill not fetched
2. Payment failed but amount debited
3. Payment pending for long time
4. Wrong bill amount
5. Request refund

AUTOPAY ISSUES
6. AutoPay not working
7. Disable AutoPay

OTHER
8. Raise a ticket
9. Talk to agent
10. FAQs
```

---

**User:** [Clicks "Raise a ticket"]

**Bot:** (Opens WhatsApp Flow - support_ticket_flow)
```
[FLOW SCREEN]
Raise a Support Ticket

Issue type:
[Dropdown: Other] ▼

Related transaction (optional):
[Search or enter TX ID...]

Description: *
[Text area]
Describe your issue...

Attach screenshot (optional):
[Upload]

[Submit Ticket]
```

---

**User:** [Fills form: Issue="Payment pending", TX ID="SETU20260207001", Description="Payment showing pending but completed in UPI app"]
[Clicks "Submit Ticket"]

**Bot:** (Text + Reply Buttons)
```
✅ Ticket Created

Ticket ID: SETU20260207001
Status: Open

Our team will review and respond within 4-6 hours.

You can check status anytime by typing:
"ticket SETU20260207001"

Need urgent help?
```
**[Buttons: Call Us | Email Us | Back to Menu]**

---

**User:** menu

**Bot:** (Shows HOME menu)

---

**[End of Demo Transcript]**

---

## 9. IMPLEMENTATION CHECKLIST

### 9.1 Phase 1: Core Payment Flow (MVP)
- [ ] Database schema setup (Supabase)
- [ ] Webhook endpoint for incoming messages
- [ ] Home menu (interactive list)
- [ ] Pay new bill flow (A.1 - A.9)
- [ ] Payment gateway integration (mock for MVP)
- [ ] Receipt generation
- [ ] Basic error handling
- [ ] fetchBill API integration (BBPS or mock)

### 9.2 Phase 2: Saved Billers & 1-Tap
- [ ] Save biller flow
- [ ] Saved billers list & management
- [ ] 1-tap payment flow
- [ ] Edit/delete saved billers
- [ ] Consumer params encryption

### 9.3 Phase 3: Reminders
- [ ] Reminder setup flow
- [ ] Reminder management flow
- [ ] Background job scheduler (cron/queue)
- [ ] Template message sending (outside 24h window)
- [ ] Snooze & disable functionality

### 9.4 Phase 4: AutoPay / SmartPay
- [ ] AutoPay setup flow
- [ ] UPI mandate integration (mock for MVP)
- [ ] SmartPay notifications
- [ ] AutoPay execution logic
- [ ] Cap amount & approval handling
- [ ] AutoPay management flow

### 9.5 Phase 5: Track & Support
- [ ] Payment tracking flows
- [ ] Receipt download/share
- [ ] Support ticket flow
- [ ] FAQ content
- [ ] Human agent handoff logic

### 9.6 Phase 6: Polish & Scale
- [ ] Multi-language support (Hindi, Kannada)
- [ ] Template message approvals (Meta Business)
- [ ] Analytics & logging
- [ ] Rate limiting & abuse prevention
- [ ] Load testing
- [ ] Data retention policies
- [ ] Privacy & compliance audit
- [ ] User onboarding optimization

---

## 10. METRICS & SUCCESS CRITERIA

### 10.1 Key Metrics

**Engagement:**
- Daily Active Users (DAU)
- Weekly Active Users (WAU)
- Session length
- Messages per session
- Drop-off points in flows

**Conversion:**
- Bill fetch success rate
- Payment initiation rate (after bill fetch)
- Payment success rate
- 1-tap payment adoption (saved billers usage)
- AutoPay adoption rate

**Retention:**
- D7, D30, D90 retention
- Repeat payment rate
- Saved billers per user
- Active reminders per user

**Support:**
- Ticket volume
- Ticket resolution time
- Human escalation rate
- User satisfaction (CSAT)

**Business:**
- GMV (Gross Merchandise Value)
- Take rate (if applicable)
- Cost per transaction
- Customer acquisition cost (CAC)

### 10.2 Success Criteria (6-month targets)

- 100K+ registered users (wa_id)
- 70%+ payment success rate
- 50%+ users with ≥1 saved biller
- 30%+ users with ≥1 active reminder
- 10%+ AutoPay/SmartPay adoption
- <5% support ticket rate (tickets/transactions)
- 4.5+ CSAT rating
- <2h avg support resolution time

---

## CONCLUSION

This specification defines a comprehensive WhatsApp-native bill payment experience optimized for Indian users. The conversational design leverages Meta's interactive features while respecting WhatsApp constraints, ensuring a seamless, trustworthy, and scalable product.

**Key Differentiators:**
1. **1-tap saved bill payments** (reduce friction)
2. **Intelligent reminders** (proactive due date notifications)
3. **AutoPay/SmartPay** (mandate-based or notification-based automation)
4. **Trust-first UX** (masked data, clear statuses, receipts)
5. **Robust help** (contextual FAQs, ticketing, human escalation)

**Next Steps:**
1. Review & approval from stakeholders
2. API contracts finalization (BBPS, payment gateway)
3. Meta WhatsApp Business account setup
4. Template message approvals
5. Begin Phase 1 development

---

**Document Version:** 1.0
**Last Updated:** 2026-01-04
**Owner:** Product Team
**Status:** Draft for Review
