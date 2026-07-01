# Automated-Sales-Agent-Performance-Tracker
Real-Time Multi-State Sales &amp; Commission Tracking System for External Sales Agents


  **Click on the[Looker Studio dashboard link](https://datastudio.google.com/u/0/reporting/9412be12-7f6c-498b-8989-8d115f9481cf/page/p_i74ejzmurd/edit) to view the Dashboard**


# Business Problem

MCG works with a network of external sales agents spread across six Nigerian states (Ogun, Oyo, Ondo, Osun, Ekiti, Kwara) plus Lagos, who sell health insurance and related products on an indirect, commission-earning basis. Before this system existed, several operational gaps limited visibility and control:

1. **Fragmented, Untraceable Sales Data:** Each state's agents recorded sales independently with no standard mechanism for consolidating records into one central, trustworthy dataset.

2. **Manual Commission Computation:** Commissions were calculated by hand across a matrix of product types, purchase types (new vs. renewal), and payment durations. Each product carries its own commission tiers, making manual calculation slow and error-prone at scale.

3. **No Real-Time Performance Visibility:** Management had no way to see current sales volume, commissions payable, or agent performance without waiting on manual compilation, which delayed decision-making.

4. **Data Integrity Risk:** Without access controls, an agent in one state could accidentally (or deliberately) alter another agent's entries, undermining trust in the underlying data.

5. **No Standardized Reporting Structure:** There was no consistent way to compare performance across states, track trends over time, or identify top-performing agents.

# Rationale for the Project

1. **Real-Time Decision-Making:** A live, auto-refreshing dashboard lets leadership track sales and commission payouts as they happen, rather than reconstructing them after the fact.

2. **Elimination of Manual Error:** A dynamic, product-aware formula automates commission calculation across 15+ product and plan combinations, removing a major source of human error and rework.

3. **Data Governance & Security:** Restricting each state's data entry to only that state's agents protects the integrity of the dataset at the point of entry, not after the fact.

4. **Scalability:** The architecture (per-state sheet → automated merge → master dataset → dashboard) is built to onboard new states or agents without a structural redesign.

5. **Transparency & Accountability:** Giving both agents and management visibility into individual and aggregate performance - including a leaderboard - supports faster commission reconciliation and healthy performance competition.

# Aim of Project

1. **Secure Data Collection:** Build a multi-state data entry system where each state's agents can only view and edit their own state's records.

2. **Data Integration:** Automatically merge and deduplicate records from every state sheet into a single, reliable master dataset.

3. **Commission Automation:** Replace manual commission math with a dynamic formula that accounts for product type, purchase type, and payment duration in a single calculation.

4. **Live Visualization:** Feed a clean, analysis-ready view into Looker Studio for a real-time sales and commission dashboard.

5. **Operational Monitoring:** Maintain an audit trail of every data merge and trigger automated alerts so pipeline failures are caught immediately, not discovered days later.

# System Architecture - How I Orchestrated It

This isn't a static dataset; it's an end-to-end pipeline with five layers.

### 1. Data Collection Layer (State-Level Google Sheets)
Each state (Ogun, Oyo, Ondo, Osun, Ekiti, Kwara, Lagos) has its own protected tab within the workbook. Sheet-level permissions ensures an agent working in, say, Oyo State cannot see or edit entries in the Ogun State tab, and vice versa. Every tab captures the same structured fields, so downstream automation can treat all seven tabs identically.

### 2. Dynamic Commission Engine
I placed every sales row to calculate its own commission through a single nested formula that branches on three variables at once:
- **Product** - All Health product variants, Life Insurance, motor comprehensive/third-party, Fire & Burglary, GIT, Marine, Credit Life, and more
- **Purchase Type** - New Purchase vs. Renewals (new business earns a materially higher rate)
- **Payment Duration** - Monthly, Quarterly, Bi-annual, or Annual

For annual, bi-annual,and quarterly **New Purchases**, the formula blends the first billing period at the "new business" rate with the remaining periods at the "renewal" rate - because only the first cycle represents genuinely new business, even though the customer paid for a full year upfront. This is the piece that used to be done manually per policy. Although business subscribing just for 1 month have a filter that can help them indicate if its a renewal or new purchase

### 3. Data Integration Layer (Google Apps Script)
A scheduled script (`setupAndMergeSheetsWithEnhancements`) runs on an automated trigger to:
- Pull new rows from every state tab
- Deduplicate using a composite key (Serial Number + Date + Product + Agent) so re-running the merge never creates duplicate entries
- Enrich each row with derived **Year**, **Month**, **Quarter**, and **ISO Week** fields for trend analysis
- Append clean rows into a single **Master** sheet
- Log every run - timestamp, sheet name, rows merged - to a dedicated **Merge Log** tab for auditability
- Send a success/failure email notification after every run

A companion script (`setDatePickersInColumnB`) enforces date-picker validation on every state tab, so agents can only submit correctly formatted dates - preventing the kind of malformed input that silently breaks downstream date logic.

### 4. Reporting Layer
A **Filtered View** sheet formats the Master data (currency formatting, cleaned columns) for consumption by external tools. A **Pivot Table** tab provides a spreadsheet-native cross-check of premium generated and commission by state and agent, independent of the dashboard.

### 5. Visualization Layer (Looker Studio)
The live dashboard connects directly to the Filtered View and surfaces:
- Total Sales, Current Month Sales, Month-over-Month Growth
- Aggregate Commission Paid
- Sales by State and an Agent Leaderboard
- Quarterly and month-on-month sales trend lines
- Filters for Date Range, Product, Agent, and State

# Data Description

Each state sheet - and the resulting Master sheet - captures the following fields:

**S/N (Number):** A sequential record identifier within the state tab.

**Date (Date):** The date the sale was recorded.

**Agent Name (Text):** The external sales agent responsible for the sale.

**Phone Number (Text):** The agent's contact number.

**State (Text):** The state the agent operates in (Ogun, Oyo, Ondo, Osun, Ekiti, Kwara, Lagos).

**Metropolis (Text):** The city/local area within the state (e.g., Ibadan, Abeokuta, Akure).

**Product Category (Text):** The broad product line (currently Health).

**Product Name (Text):** The specific plan sold (Flexicare, Zencare Plus, PrimeCare, etc.).

**Price (Currency):** The base price of the plan.

**Qty Sold (Number):** Units/policies sold in the transaction.

**Duration (Text):** The billing cycle purchased - Monthly, Quarterly, Bi-annual, or Annual.

**Total Amount (Currency):** The full amount paid by the customer.

**Purchase Type (Text):** New Purchase or Renewal.

**HMO ID (Text):** The enrollee's HMO identification number.

**Enrollee's Name (Text):** The name of the insured customer.

**Commission (Currency):** The commission owed to the agent, calculated automatically by the dynamic formula described above.

**Year / Month / Quarter / Week (Derived):** Added automatically during the merge into Master, enabling time-based analysis without manual date parsing.
<img width="1600" height="51" alt="WhatsApp Image 2026-07-01 at 8 58 00 AM (1)" src="https://github.com/user-attachments/assets/8e38fb18-5bed-4d3b-a08e-bcdfd98e3b4a" />

# Automation Scripts

### Merge & Enrich (runs on a scheduled trigger)
```javascript
function setupAndMergeSheetsWithEnhancements() {
  // Pulls new rows from every state sheet, deduplicates using a
  // composite key (S/N + Date + Product + Agent), enriches each row
  // with Year/Month/Quarter/Week, appends to Master, and logs the run.
  // Full script available in /apps-script in this repo.
}
```

### Commission Formula (per row, in the Master/state sheets)
```
=IF(H27="", "", IFS(
  (H27="Flexicare") * (M27="New Purchase") * (K27="Annual"),
    ROUND(L27/12*0.343, -2) + ROUND(L27*11/12*0.286, -2),
  (H27="Flexicare") * (M27="New Purchase") * (K27="Bi-annual"),
    ROUND(L27/6*0.343, -2) + ROUND(L27*5/6*0.286, -2),
  (H27="Flexicare") * (M27="New Purchase") * (K27="Quarterly"),
    ROUND(L27/3*0.343, -2) + ROUND(L27*2/3*0.286, -2),
  (H27="Flexicare") * (M27="New Purchase"), ROUND(L27*0.343, -2),
  (H27="Flexicare") * (M27="Renewal"), ROUND(L27*0.286, -2),
  ... [additional branches per product]
))
```
*(Full formula covers 15+ products; request for the workbook for the complete logic.)*

### Date Validation (runs on state tabs)
```javascript
function setDatePickersInColumnB() {
  // Applies a date-picker data validation rule and dd/mm/yyyy
  // formatting to column B on every state tab, preventing malformed
  // date entries from breaking downstream logic.
}
```

# Project Scope

1. **Data Collection Design** - structuring state-level tabs with restricted, agent-specific permissions.
2. **Data Integration** - building the Apps Script merge pipeline with deduplication and audit logging.
3. **Commission Logic Design** - translating the company's tiered, product-specific commission structure into a single dynamic formula.
4. **Data Visualization** - connecting the cleaned dataset to Looker Studio and designing a real-time performance dashboard.
5. **Monitoring & Reliability** - automated email alerts and a merge log to catch pipeline failures early.

# This screenshot captures highlights of the following:
- Sales Agents Performance Dashboard - overview KPIs and month-on-month trend
<img width="1360" height="1022" alt="WhatsApp Image 2026-07-01 at 9 16 11 AM" src="https://github.com/user-attachments/assets/942e33f5-0fcc-4356-86ed-ec664798919c" />


- Agent Leaderboard and Sales by State
<img width="1338" height="1002" alt="WhatsApp Image 2026-07-01 at 10 01 22 AM" src="https://github.com/user-attachments/assets/80dc4c7e-dfe4-4bfd-b010-2141f006216b" />


- Google Sheets → Looker Studio connection setup
- <img width="1600" height="667" alt="WhatsApp Image 2026-07-01 at 9 55 08 AM" src="https://github.com/user-attachments/assets/00a171c4-33d5-4d62-9cff-1475315381b4" />

- Raw data entry structure (state tab)
- <img width="1600" height="944" alt="WhatsApp Image 2026-07-01 at 8 58 01 AM" src="https://github.com/user-attachments/assets/a599f7ac-a9da-41ef-b701-cd438e41acaf" />

- Commission formula in the sheet
<img width="1574" height="1102" alt="WhatsApp Image 2026-07-01 at 10 06 40 AM" src="https://github.com/user-attachments/assets/e39580fa-ff6d-4048-b995-8d5821f0abc2" />


# Conclusion

This project moved MCG's external agent network from manual, after-the-fact commission tracking to a real-time, automated pipeline. Sales entered by agents in any of the seven state sheets flow - without manual intervention; into a deduplicated master dataset, get commission-calculated automatically against the correct product and tier, and surface within minutes on a live dashboard. The state-level permissioning also closes a data integrity gap that existed when commissions were reconciled manually.

# Strategic Recommendations Implemeneted on Phase 2 of this Project

**Extend Anomaly Detection:** Setting up conditional flags for commission outliers (e.g., a single row generating an unusually high commission) so they can be reviewed before payout rather than after.

**Commission Reconciliation Trail:** Added a simple "paid vs. pending" status column so finance can track which calculated commissions have actually been disbursed to agents.

**Agent-Facing Notifications:** Extended the existing email alert system to notify agents directly when their sales are merged and their commission is calculated, improving transparency and reducing dispute volume.



# The Way Forward

As MCG's agent network grows, this system is designed to scale without redesign - new states slot into the existing merge logic, and new products can be added to the commission formula as additional branches. The next phase of this project focuses on tightening the feedback loop between the dashboard and the agents themselves, so performance data drives faster, more transparent commission conversations.

# How to Use

1. Each state's sales agents record their sales directly into their permissioned state tab in the Google Sheet.
2. The Apps Script trigger (`setupAndMergeSheetsWithEnhancements`) runs automatically on a schedule, merging and deduplicating new entries into the Master sheet.
3. The commission formula calculates each row's commission automatically at the point of entry - no manual computation required.
4. The Filtered View sheet feeds the Looker Studio dashboard, which updates as new data lands in Master.
5. Management reviews live performance via the [Looker Studio dashboard link](https://datastudio.google.com/u/0/reporting/9412be12-7f6c-498b-8989-8d115f9481cf/page/p_i74ejzmurd/edit) - no manual report compilation needed.
