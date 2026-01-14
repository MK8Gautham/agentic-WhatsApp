# BillPay on WhatsApp — README

A production-ready blueprint for delivering BBPS bill payments on WhatsApp using Meta **WhatsApp Cloud API** primitives:
- **Interactive List / Reply Buttons** for navigation
- **WhatsApp Flows** for structured forms (biller select, consumer input, reminders, ticketing)
- **Message Templates** for reminders + async payment updates outside the 24-hour window

This repo/asset set includes:
- Conversation UX + edge-case handling
- Flow screen specs (“JSON-ish”)
- Excalidraw-ready Mermaid diagram for the full journey

---

## What users can do (V1)
### ✅ Core journeys
1. **Pay a bill / Recharge**
   - Category → Biller → Consumer details → Bill preview → Pay → Receipt
2. **Pay saved bills (1-tap)**
   - Saved bill list → Auto-fetch → Pay → Receipt
3. **Reminders & AutoPay**
   - Reminders (T-3/T-2/T-1) via templates
   - AutoPay (mandate) / SmartPay fallback (auto-fetch + 1-tap pay template)
4. **Track payment / Receipt**
   - Recent txns / Txn ID / UTR → Status → Receipt → Retry / Complaint
5. **Help**
   - Guided troubleshooting + ticket creation + human escalation + privacy controls

---

## Why this is “app-like” on WhatsApp
- Minimal typing: most actions are **lists**, **buttons**, or **Flow forms**
- Repeat pay in 2–3 taps via **Saved Bills**
- Trust UX: masked identifiers, clear status (Pending/Deemed/Success/Failed), instant receipts
- Handles async outcomes via **templates** (status updates / reminders)

---

## High-level architecture

### Components
- **WhatsApp Business Account (WABA) + Verified Number**
- **Cloud API**
- **Bot Orchestrator** (state machine + copy)
- **Flow Service**
  - Serves Flow JSON
  - Handles `data_exchange` callbacks
- **BBPS layer** (Setu BBPS APIs)
  - bill fetch, payment, status inquiry, receipt generation
- **Scheduler**
  - reminder triggers
  - pending/deemed status updates
- **Support system**
  - ticketing (internal/helpdesk) + human handoff

### Typical request path
1. User message / Flow submit → Cloud API Webhook → Bot Orchestrator
2. Orchestrator calls BBPS (fetch/pay/status) + Flow Service (screens)
3. Orchestrator replies with interactive msg / Flow / template

---

## Message types used
- **Interactive List**: Home menu, category list, saved bills, recent transactions
- **Reply Buttons**: Yes/No, Pay now/Edit/Back
- **Flows**: Enter consumer details, reminders setup, autopay setup, track search, ticket creation
- **Templates**: Reminders, pending resolved updates, deemed confirmation updates, receipts (when outside 24h)

---

## State machine (payment)
Supported payment states:
- `INITIATED` → `PENDING` → `SUCCESS` | `FAILED`
- `DEEMED` → `SUCCESS_CONFIRMED` (later)
Rules:
- If user is outside the 24-hour session window, send **templates** for updates.
- Always provide self-serve: **Track payment** and **Raise complaint**.

---

## Excalidraw-ready Mermaid diagram
Paste this directly into Excalidraw (Mermaid import) or render in Markdown.

```mermaid
flowchart TD
  U([User]) --> HI["User sends: Hi / Menu"]
  HI --> LANG{"Language set?"}
  LANG -- No --> LANGSEL["Reply Buttons:\nEnglish | हिंदी | ಕನ್ನಡ | More"]
  LANGSEL --> HOME
  LANG -- Yes --> HOME

  HOME["HOME (Interactive List)\n1) Pay a bill\n2) Pay saved bills (1-tap)\n3) Reminders & AutoPay\n4) Track payment/Receipt\n5) Help"] --> CHOICE{"User selects?"}

  CHOICE -- "Pay a bill" --> PAYCAT["Interactive List:\nTop categories + All categories"]
  PAYCAT --> BILLERSEL["FLOW: BILLER_SELECT"]
  BILLERSEL --> ENTERCONS["FLOW: ENTER_CONSUMER\n(validations)"]
  ENTERCONS --> FETCHBILL["Backend: fetchBill"]
  FETCHBILL --> FETCHOUT{"Fetch outcome"}
  FETCHOUT -- OK --> PREVIEW["FLOW: BILL_PREVIEW\nPay now | Edit"]
  FETCHOUT -- "No due" --> NODUE["FLOW: NO_DUE\nSet reminder | Menu"]
  FETCHOUT -- "Biller down" --> BDWN["FLOW: ERROR_BILLER_DOWN\nTry again | Help"]
  FETCHOUT -- "Invalid params" --> CONSERR["Inline field error"]

  PREVIEW --> PAYINIT["Pay (WhatsApp Pay / UPI intent)"]
  PAYINIT --> PAYSTAT{"Payment status"}
  PAYSTAT -- PENDING --> PENDMSG["Processing ⏳\nUpdate later"]
  PAYSTAT -- DEEMED --> DEEMMSG["Deemed accepted\nConfirm later"]
  PAYSTAT -- SUCCESS --> SUCCESSMSG["Success ✅\nReceipt + actions"]
  PAYSTAT -- FAILED --> FAILMSG["Failed ❌\nRetry | Help | Track"]

  PENDMSG --> CSW{"Within 24h window?"}
  DEEMMSG --> CSW
  CSW -- Yes --> FINALINSESSION["Session update"]
  CSW -- No --> FINALTMPL["Template update"]

  CHOICE -- "Pay saved bills" --> SAVEDLIST["Saved bills list"]
  SAVEDLIST --> SFETCH["Auto fetch bill"]
  SFETCH --> SFOUT{"Outcome"}
  SFOUT -- OK --> SPREVIEW["Preview\nPay now | Edit"]
  SFOUT -- "Biller down" --> SBDWN["Biller down\nHelp"]
  SFOUT -- "No due" --> SNODUE["No due\nSet reminder"]
  SFOUT -- "Invalid saved params" --> SEDIT["FLOW: EDIT_SAVED_BILLER"]

  CHOICE -- "Reminders & AutoPay" --> RA_MENU["Reminders & AutoPay menu"]
  RA_MENU --> R_SETUP["FLOW: REMINDER_SETUP"]
  R_SETUP --> R_TMPL["Reminder Template\nPay now CTA"]
  RA_MENU --> A_SETUP["FLOW: AUTOPAY_SETUP"]
  A_SETUP --> A_ELIG{"Eligible?"}
  A_ELIG -- Yes --> MANDATE["Mandate authorize"]
  A_ELIG -- No --> SMARTPAY["SmartPay fallback"]
  RA_MENU --> A_MGMT["FLOW: AUTOPAY_MANAGE"]

  CHOICE -- "Track payment/Receipt" --> T_MENU["Track options"]
  T_MENU --> T_FLOW["FLOW: TRACK_SEARCH"]
  T_MENU --> T_RECENT["Recent txns list"]
  T_FLOW --> T_DETAIL["Txn detail + actions"]
  T_RECENT --> T_DETAIL

  CHOICE -- "Help" --> H_MENU["Help menu"]
  H_MENU --> TICKET["FLOW: TICKET_CREATE"]
  H_MENU --> HUMAN["Human handoff"]
