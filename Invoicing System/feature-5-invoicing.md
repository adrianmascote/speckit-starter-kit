# Feature 5: Invoicing 

## User Stories 

### US-5.1: Generate a Draft Client Invoice
 
**As a** Accounting Clerk
**I want to** generate a draft invoice for a project over a selected billing period 
**So that** all approved billable hours and approved billable expenses are automatically compiled and totaled for the client. 

**Priority:** P1 
**Independent test:** Verify an Accounting Clerk can generate a `Draft` invoice for a project and date range that includes all uninvoiced `Approved` time entries (calculated using `ProjectRole.hourly_rate`) and uninvoiced `Approved` billable expenses. 
**Acceptance scenarios:** see ### US-5.1 under Gherkin AC 

### US-5.2: Review and Finalize (Send) an Invoice
**As a** Accounting Clerk
**I want to** review a draft invoice's line items and mark it as `Sent`
**So that** the invoice is locked for billing and ready to be delivered to the client. 

**Priority:** P1
**Independent test:** Verify an Accounting Clerk can transition a `Draft` invoice to `Sent` and lock its line items from further modification.
**Acceptance scenarios:** see ### US-5.2 under Gherkin AC 

### US-5.3: Record Invoice Payment 
**As an** Accounting Clerk
**I want to** mark a `Sent` invoice as `Paid` with the payment date
**So that** the firm can accurately track which client invoices have been settled and which remain outstanding

**Priority:** P1
**Independent test:** Verify an Accounting Clerk can update a `Sent` invoice's status to `Paid` with a `paid_date`. 
**Acceptance scenarios:** see ### US-5.3 under Gherkin AC 

### US-5.4: View Invoice History and Details
**As an** Accounting Clerk
**I want to** view a list of all invoices filtered by client, project, or status and inspect any invoice's full line-item breakdown
**So that** I can answer client billing questions and track accounts receivable

**Priority:** P1
**Independent test:** Verify an Accounting Clerk can filter invoices by `client_id`, `project_id`, or `status` and view the complete line-item details of the invoice. 
**Acceptance scenarios:** see ### US-5.4 under Gherkin AC 



## Functional Requirements 
* **FR-1:** The system must allow Accounting Clerks (and Admin/Owners) to generate a `Draft` invoice for a specific `Project` and date range (`billing_start_date` to `billing_end_date`)
* **FR-2:** When generating an invoice, the system must pull only `TimeEntry` rows with `status = 'Approved'` that have not already been billed on another invoice, calculating each line item as `hours * ProjectRole.hourly_rate`.
* **FR-3:** When generating an invoice, the system must pull only `Expense` rows with `status = 'Approved'` and `billable = true`that have not already been billed on another invoice.
* **FR-4:** The system must snapshot each line item's description, quantity/ hours, unit rate, and line total in `InvoiceLineItem` so future rate changes on a `ProjectRole` never change past historical invoices. 
* **FR-5:** The system must track each `Invoice` through a defined status lifecycle: `Draft`, `Sent`, `Paid`, and `Void`, and prevent double-billing of a time entry or expense linked to an active (`Draft`, `Sent`, or `Paid`) invoice


## Assumptions 
* Each invoice belongs to one `Project` and automatically pulls billing contact details (`client_name`, `billing_info`, `email`) from that project's linked `Client` (Feature 2).
* All monetary calculations use exact decimal math to avoid flating point rounding errors. 
* If a `Draft` invoice is voided or deleted before being sent, its linked `TimeEntry` and `Expense` records become available for future invoices. 
* Invoices are generated and viewed in-app (printable view); automated email delivery or payment gateway processing is out of scope. 



## Edge Cases 
* **No Billable Items Found:** If an Accounting Clerk attempts to generate an invoice for a project and date range that has zero uninvoiced `Approved` time entries and zero uninvoiced `Approved` billable expenses, the system must block creation and display a clear message ("No approved uninvoiced time or expenses found for this period").
* **Invalid Date Range:** If `billing_end_date` is earlier than `billing_start_date`, or `due_date` is earlier than `issue_date` the system must reject the request with a validation error
* **Double-billing Prevention:** If an Accounting Clerk generates a second invoice for an overlapping date range on the same project, the system must exclude any `TimeEntry` or `Expense` already attached to an existing `Draft`, `Sent`, or `Paid` invoice. 
* **Modifying a Locked Invoice:** If a user attempts to edit line items or delete an invoice whose status is `Sent` or `Paid`, the system must block the action 
* **Unapproved or Non-Billable Records:** `Draft`, `Submitted`, or `Rejected` time entries and expenses-as well as `Approved` expenses where `billable = false` - must never be included on an invoice.

## Success Criteria 
* An Accounting Clerk can generate, verify, and finalize an accurate client invoice in under 2 minutes. 
* 100% of invoice totals exactly equal the sum of their `InvoiceLineItem` rows (`hours * hourly_rate` for time + `amount` for billable expenses)
* 0 approved time entries or expenses are ever billed twice across multiple invoices. 


## Key Entities 
* **Invoice** 
* **InvoiceLineItem** 
* **Client** *(referenced from Feature 2)*
* **Project** *(referenced from Feature 2)*
* **ProjectRole** *(referenced from Feature 2)*
* **TimeEntry** *(referenced from Feature 3)*
* **Expense** *(referenced from Feature 4)*

## Initial Data Model 
                      
### Invoice

| Field Name | Data Type | Description | 
|---|---|---| 
| `invoice_id` | Integer | Primary key for the invoice | 
| `invoice_number` | String | Unique invoice code  | 
| `client_id` | Integer | Foreign key referencing `Client.client_id` |
| `project_id` | Integer | Foreign key referencing `Project.project_id` |
| `billing_start_date` | Date | Start date of the billing period (`YYYY-MM-DD`) | 
| `billing_end_date` | Date | End date of the billing period (`YYYY-MM-DD`) | 
| `issue_date` | Date | Date the invoice was issued (`YYYY-MM-DD`) | 
| `due_date` | Date | Payment due date (`YYYY-MM-DD`) | 
| `subtotal_time` | Decimal | Sum of all time-based line items on this invoice | 
| `subtotal_expenses` | Decimal | Sum of all billable expense line items on this invoice | 
| `total_amount` | Decimal | Grand total (`subtotal_time + subtotal_expenses`) |
| `status` | Enum | Current invoice state (`Draft`, `Sent`, `Paid`, `Void`) | 
| `paid_date` | Date | Date payment was received (nullable; set when status becomes `Paid`) | 
| `created_by` | Integer | Foreign key referencing `Employee.employee_id` of the Accounting Clerk/Admin who generated it | 

### InvoiceLineItem

| Field Name | Data Type | Description | 
|---|---|---|
| `line_item_id` | Integer | Primary key for the invoice line item | 
| `invoice_id` | Integer | Foreign key referencing `Invoice.invoice_id` |
| `item_type` | Enum | Type of charge (`Time` or `Expense`) | 
| `time_entry_id` | Integer | Foreign key referencing `TimeEntry.time_entry_id` (nullable; unique across non-void invoices) |
| `expense_id` | Integer | Foreign key referencing `Expense.expense_id` (nullable; unique across non-void invoices) | 
| `description` | String | Snapshot description (e.g., `"Adrian Mascote - Developer (2026-11-18): API setup"` or expense description) |
| `quantity` | Decimal | Hours worked (for `Time`) or `1`(for `Expense`) |
| `unit_rate` | Decimal | Snapshot `hourly_rate` (for `Time`) or expense `amount` (for `Expense`) |
| `line_total` | Decimal | Calculated total for this line (`quantity * unit_rate`) |


## Gherkin AC 

### US-5.1 
**Scenario: Accounting Clerk generates a draft invoice from approved time and billable expenses**
**Given:** "Cloud Migration" has an approved uninvoiced TimeEntry of "6.5" hours at "$150.00/hr" and an approved billable Expense of "$49.99" within "2026-11-01" to "2026-11-30"
**When:** an Accounting Clerk selects "Cloud Migration", enters billing period "2026-11-01" to "2026-11-30" 
**And** clicks "Generate Invoice" 
**Then:** the system creates a "Draft" Invoice with subtotal_time "$975.00", subtotal_expenses "$49.99" and total_amount "$1024.99"
**And:** both records are linked via InvoiceLineItem so they cannot be billed again

### US-5.2 
**Scenario: Accounting Clerk finalizes and marks a draft invoice as Sent** 
**Given** a "Draft" Invoice "INV-2026-0001" exists for "Acme Corp" 
**When** an Accounting Clerk opens "INV-2026-0001" and clicks "Mark as Sent" 
**Then** the Invoice status updates to "Sent"
**And** the invoice line items are locked from further editing

### US-5.3
**Scenario: Accounting Clerk records payment on a Sent invoice**
**Given** Invoice "INV-2026-0001" has status "Sent"
**When** an Accounting Clerk clicks "Record Payment", enters paid_date "2026-12-05", and clicks "Confirm"
**Then** the invoice status updates to "Paid" with paid date "2026-12-05"
**And** the expense is locked and available for client invoicing

### US-5.4
**Scenario: Accounting Clerk filters invoices by7 status and views invoice details**
**Given** Invoice "INV-2026-0001" has status "Sent" and Invoice "INV-2026-0002" has status "Paid"
**When** an Accounting Clerk filters the Invoice list by status "Sent" and clicks "INV-2026-0001"
**Then** only "INV-2026-0001" appears in the filtered list
**And** the detail view displays the client billing info, all line items, and total_amount

# Data Ownership & Isolation 

* Accounting Clerks and Admin/Owners manage and generate all `Invoice` and `InvoiceLineItem` records; Project Managers have read-only access to invoices for thier projects; Team Members have no access to client invoices. 

| Rule | Requirement | 
| --- | --- | 
| **Read scope** | Accounting Clerks and Admin/Owners see all invoices; Project Managers have read-only access to invoices; Team Members have no access (`404`) |
| **Write scope** | Only authenticated users with `system_role` in `('AccountingClerk', 'AdminOwner')`can generate, send, void, or mark invoices as paid | 
| **Create scope** | New `Invoice` and `InvoiceLineItem` rows created exclusively by an authenticated `AccountingClerk` or `AdminOwner` with `created_by = req.user_id`| 
| **Cross-user-access** | Team Member requesting any `/api/invoices` endpoint -> `404` (not `403`) | 
| **UI scope** | UI displays the invoicing menu and screens only to Accounting Clerks, Admin/Owners, and (read-only) Project Managers | 
| **Implementation** | Shared helper in `app/authorization/` - do not dulicate scope in every controller | 


## API Requirements 

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/invoices` | Yes | Retrieves list of invoices with optional `client_id`, `project_id` and `status` filters (AccountingCler/AdminOwner/PM)  |
| `GET` | `/api/invoices/{invoice_id}` | Yes | Retrieves full invoice header, client billing info, and all `InvoiceLineItem` rows (AccountingClerk/AdminOwner/PM) |
| `POST` | `/api/invoices/generate` | Yes | Generates a new `Draft` invoice and snapshots all eligible approved uninvoiced time and billable expenses (AccountingClerk/AdminOwner only) | 
| `PATCH` | `/api/invoices/{invoice_id}/send` | Yes | Transitions a `Draft` invoice to `Sent` and locks it (AccountingClerk/AdminOwner only) |
| `PATCH` | `/api/invoices/{invoice_id}/pay` | Yes | Transitions a `Sent` invoice to `Paid` and records `paid_date` (AccountingClerk/AdminOwner only) |
| `PATCH` | `/api/invoices/{invoice_id}/void` | Yes | Marks a `Draft` or `Sent` invoice as `Void` and releases its linked time/expenses for future billing (AccountingClerk/AdminOwner only) |



**Create request body(`POST /api/invoices/generate`):**
```json 
{
 "project_id" : 10,
 "billing_start_date" : "2026-11-01", 
 "billing_end_date" : "2026-11-30",
 "issue_date" : "2026-12-01",
 "due_date" : "2026-12-31",
}

```


## Screen Requirements

### [View: Invoices Dashboard Screen] - route name `invoices-list`
* Heading / purpose: Invoice management, generation, and payment tracking(Accounting Clerk/Admin)
* Primary action: **Generate Invoice** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icon actions with labels: Filter bar (`Client`, `Project` ,`status`); Generate Invoice dialog(`project_id` dropdown, `billing_start_date`, `billing_end_date`, `issue_date`, `due_date`, `Generate` button); Invoices table showing `invoice_number`, Client, Project, Billing Period, `total_amount`, `status` badge, and row-level view Details action
* **Empty state:** "No invoices found"
* Loading / error: skeleton/progress; `<v-alert type="error">` for failures 



### [View: Invoice Detail Screen] - route name `invoice-detail`
* Heading / purpose: Invoice management, generation, and payment tracking (Accounting Clerk/Admin)
* Primary action: **Mark as Sent** or **Record Payment** (When `Sent`)(`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icnn actions with labels: Client billing header (`client_name`, `billing_info`, `email`); Time Line Items table; Expense line Items table; Totals Summary (`subtotal_time`, `subtotal_expenses`, `total_amount`); **Void Invoice** secondary action; Record Payment dialog (`paid_date`)
* **Empty state:** "Invoice not found."
* Loading / error: skeleton/progress; `<v-alert type="error">` for failures

## Test Coverage Map 

| Story | Scenario | Test file | Test name | 
|-------|----------|-----------|-----------|
| US-5.1 | Accounting Clerk generates a draft invoice from approved tiem and billable expenses | `backend/tests/invoices.test.js` |  `Accounting clerk generates draft invoice with accurate totals` | 
| US-5.2 | Accounting Clerk finalizes and makrs a draft invoice as Sent | `backend/tests/invoices.test.js` | `Accounting clerk marks draft invoice as sent` |
| US-5.3 | Accounting Clerk records payment on a Sent invoice | `backend/tests/invoices.test.js` | `Accounting clerk filters invoices and views line items` |



## Agent Implementation request 

```text
Implement Feature 5 from @Invoicing System/feature-5-invoicing.md on branch `invoicing system`. 


Follow layer order in @features/framework.md (models -> routes -> backend tests -> frontend -> frontend tests).
Map every Gherkin scenario in the Test Coverage Mape; run `npm test` before finishing.
If API routes, payloads, schema, or product rules changed per this spec update @features/reference/api.md, @features/reference/data-model.md, and/or @features/reference/behavior.md in the same PR to match the code. 
```


## Definition of Done 

* [ ] Backend and frontend implemented per this spec (FRs satisfied)
* [ ] Success Criteria met
* [ ] All mapped tests pass (`npm test`)
* [ ] Test Coverage Map complete 
* [ ] `features/reference/data-model.md` updated (if schema changed)
* [ ] `features/reference/api.md` updated (if API changed)
* [ ] `features/ reference/behavior.md` updated (if product rules changed)

## Out of Scope 
* Automated email delivery of invoices or PDF attachment generation 
* Online credit card payment processing.
* Sales tax calculation, multi-currency conversion, or partial payment tracking

