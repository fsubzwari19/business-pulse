# Business Pulse — Cowork task specification

Use this document as folder instructions or the primary spec when running this work in Claude Cowork.

## Goal

Create a separate folder under `Analytics` named **Business Pulse**. Build a **daily dashboard AI agent** that surfaces the metrics below with **WTD** (week-to-date) and **MTD** (month-to-date) running totals **by city**.

## Dashboard tabs

1. **High-level summary** — Counts for each KPI; add overall business analysis from trends, sentiment, and stats.
2. **Daily summary** — Daily totals with WTD and MTD columns.
3. **WoW** — Up to four weeks visible, with a filter to choose which weeks to show.
4. **MoM** — Up to four months visible (align with your data cadence), with a filter to select the month to display.
5. **Issue resolution** — Dedicated tab for resolution-focused views (see complaint / care views below).
6. **Post Dispatch** — (Can be separate tab or subsection of Issue Resolution)
   - Customer issue funnel showing: Dispatched Orders, Service Issues Total, Service Issues %, Ops Error, Commercial Error, Customer Error, Care Error, In-Progress Issues
   - Breakdown by error type with daily (DoD) view
   - **Service Issues Breakup** — SKU quality Issue, Short items delivered, Wrong items delivered, Order not delivered, Short Expiry, Item is damaged, Expired Item
   - **Views:** Daily (DoD), Week-over-Week (WoW), Month-over-Month (MoM)
7. **Deep Dive** — (Can be separate tab or subsection of Issue Resolution)
   - Two sections: (1) AI-generated analysis of red flags, correlation patterns, and trends; (2) Insights based on user queries and manual inputs
   - Surfaces actionable intelligence on root causes of issues
   - Updates dynamically based on query results

## Filter behavior

**All filters (Month, Segment, City) apply across all tabs:**
- **Month filters** — All, Jan, Feb, Mar, Apr (and future months as data grows)
- **Segment filters** — All Seg, B2B, B2C, ROB
- **City filters** — All Cities, Karachi, Lahore, Islamabad, Gujranwala, Faisalabad, Multan, ISB Outskirts, Sheikhupura, Sialkot
- When a filter is applied, all tabs update to show only data matching that filter
- Filters are cumulative (e.g., selecting "Feb" and "B2B" shows Feb data for B2B stores only)

## Authentication & Access Control

- **Viewer access** — No login required; dashboard displays read-only data
- **Admin access** — Clicking the Admin button opens a popup login window
- Admin features include: data refresh controls, filter override, export functions

## Store type mapping (ignore other store types)

| Segment | Store types |
|--------|--------------|
| **B2B** | `GENERAL_STORE`, `BUYER`, `SELF_SERVICE_STORE`, `WHOLE_SELLER`, `KARACHI_EAST_GT` |
| **B2C** | `HOUSEHOLD`, `DIGITAL_BUYER` |

(Include **Pro** and **Prime** in order breakdowns where applicable per your data model.)

## Metrics (definitions)

1. **Total orders** — All orders in the system for the date or range.
2. **Total conversations** — All conversations where `status_marked_by_ai` is **not** in `'Bot was not initialized'`.
3. **Total tickets** — All tickets in the system for the date or range.
4. **AI tickets** — `CUSTOMER_SUPPORT_ISSUE` where `ticket_created_by_name` is `'BazaarAI'`. Track volume and whether each **issue category** is increasing or decreasing.
5. **Human tickets** — Everything not counted as AI tickets above.
6. **Resolved by AI** — Auto-resolved: `ticket_created_by` and `updated_by` are the same **and** `ticket_created_by_name` is `'BazaarAI'` **OR** `'58375766'`.
7. **Resolution rate** — `(abandoned + resolved) / total conversations`, excluding bot-not-initialized conversations.
8. **Handover** — Count conversations where status was handover; use user resolution marked by AI to extract **themes** for why handovers occur.
9. **Handover %** — `total handover / total conversations`.
10. **AI error rate** — Tickets created by AI that are marked **invalid** / total tickets created by AI.

## Required views

1. **Orders by city** — Daily, weekly, and MTD totals.
2. **Orders by store-type category** — Breakdown: B2B, B2C, Pro, Prime.
3. **Complaint tickets (care_biz silver)**  
   - Overall: `issue_categories`, `issue_reason`, resolution per issue, auto-resolve per issue, handover per issue, error rate per issue.  
   - **Team-wise** breakdown of issues.  
   - Surface high-level counts on the **high-level summary** tab.  
   - **3a** — Issue category and reasons: daily, WTD, MTD, **by city** (under daily summary).  
   - **3b** — Second view: 7-day running total, WoW (four weeks), MoM (last three months).

4. **Sentiment and error analysis**  
   - **4a** — From issue description and resolver comment, classify customer sentiment: **satisfied**, **dissatisfied**, **neutral**.  
   - **4b** — Identify where AI **could not** handle the concern, tied to **issue reason** (and conversation context as available).

## Issue Resolution Tab — Data Definitions (AUTHORITATIVE)

> These definitions override any earlier descriptions of the Issue Resolution tab.

### Source table
All issue/ticket metrics come from **`hive.bazaar_care_silver.issues`** — NOT from the conversations table.

### Mandatory filters (apply before any aggregation)

| Filter | Rule |
|--------|------|
| `issue_type` | **Only** include rows where `issue_type = 'CUSTOMER_SUPPORT_ISSUE'` |
| `issue_category` | **Exclude** rows where `issue_category = 'Chat Drop/Spam Messages'` |

These two filters must be applied in every query against `bazaar_care_silver.issues`, regardless of which metric or breakdown is being calculated.

### Key columns

| Column | Purpose |
|--------|---------|
| `issue_status` | Ticket outcome: `RESOLVED`, `INVALID`, `INPROGRESS` |
| `issue_category` | Category of the complaint (e.g. Order Cancellation, Delivery Issues, etc.) |
| `issue_reason` | More granular reason within a category |
| `escalated_team` | Team responsible for the ticket (used for team-wise breakdown) |
| `ticket_created_by_name` | Who/what created the ticket (`'BazaarAI'` for AI-created) |
| `resolver_user_name` | Who resolved it |

### Required KPIs on the Issue Resolution tab

1. **Total Issues** — COUNT of all rows in `bazaar_care_silver.issues` for the period
2. **Resolved** — COUNT where `issue_status = 'RESOLVED'`
3. **Invalid** — COUNT where `issue_status = 'INVALID'`
4. **In Progress** — COUNT where `issue_status = 'INPROGRESS'`
5. **Total Conversations** — from `bazaar_care_silver.bird_crm_messages` where `status_marked_by_ai != 'Bot was not initialized'` (shown for context alongside issue counts)

### Required breakdowns

- **By issue_category** — table showing: Category | Total | Resolved | Invalid | In Progress | Resolution %
- **By escalated_team** — table showing: Team | Total | Resolved | Invalid | In Progress | Resolution % | Invalid %

### Donut / status chart
Show ticket status split: Resolved vs In Progress vs Invalid (sourced from `issue_status`).

### Important constraints
- **Never mix** issue counts (from `issues` table) with conversation counts (from `bird_crm_messages`). They measure different things.
- Resolved count **must always be ≤ Total Issues**.
- Invalid % = Invalid / Total Issues × 100.
- Resolution % = Resolved / Total Issues × 100.

---

## Critical: Quality Assurance Before Publishing

**ALWAYS perform the following before publishing any dashboard updates:**

1. **Review all data completeness** — Verify that all required metrics are present and calculated
2. **Validate calculations** — Confirm WTD, MTD, and running totals are correct
3. **Check filter accuracy** — Ensure all city, segment, and month filters work as specified
4. **Verify missing data** — If any metric or dimension is missing or incomplete:
   - **STOP** — Do not publish
   - **ASK FOR CLARITY** — Explicitly ask what should be done with missing data
   - **Document issues** — List what's missing and why
5. **Review accuracy** — Sample check data against source queries to ensure accuracy

**Common issues to check:**
- WTD/MTD calculations are correct (not just daily totals)
- Store type mapping follows the exact spec (BUYER, GENERAL_STORE, WHOLE_SELLER for B2B; HOUSEHOLD, DIGITAL_BUYER for B2C)
- Conversations exclude "Bot was not initialized"
- Resolution rate = (abandoned + resolved) / total conversations
- All city-level aggregations are present

## Deliverables checklist for Cowork

- [ ] Folder `Analytics/Business Pulse/` created and organized (data connections, docs, dashboard assets as you define).
- [ ] Agent or pipeline implements the tabs and filters described above.
- [ ] All metrics use the definitions in this spec consistently.
- [ ] City, WTD, and MTD dimensions applied where specified.
- [ ] Dashboard reviewed for accuracy and completeness before publishing.

---

*Cowork project / folder instruction spec (evolved from original `prompt.txt`).*
