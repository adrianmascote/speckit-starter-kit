# Feature 4: Expense Tracking

## User Stories 

### US-4.1: Log a Project Expense
 
**As a** Team Member 
**I want to** log a project-related expense with its category, amount, date, and description
**So that** reimbursable and client-billable costs are accurately tracked against the project.

**Priority:** P1 
**Independent test:** Verify a Team Member can create a new draft `Expense` linked to one of their assigned projects.
**Acceptance scenarios:** see ### US-4.1 under Gherkin AC 

### US-4.2: Edit and Submit Time Entries
**As a** Team Member 
**I want to** edit my draft or rejected expenses and submit them for approval
**So that** my project Manager can review and approve them for client billing

**Priority:** P1
**Independent test:** Verify a Team Member can update a draft/rejected `Expense` and change its status to `Submitted`.
**Acceptance scenarios:** see ### US-4.2 under Gherkin AC 

### US-4.3: Approve or Reject Submitted Expenses 
**As a** Project Manager
**I want to** review submitted project expenses and either approve them or reject them 
**So that** only verified expenses are included on the client invoices

**Priority:** P1
**Independent test:** Verify a Project Manager can change a `Submitted` expense to `Approved` or `Rejected` (with a required rejected reason)
**Acceptance scenarios:** see ### US-4.3 under Gherkin AC 


## Functional Requirements 
* **FR-1:** The system must allow Team Members to log expenses (`project_id`, `expense_date`, `category`, `amount`, `description`, `billable`) strictly against the active projects they're assigned to
* **FR-2:** The system must track each `Expense` through a defined status lifecycle: `Draft`, `Submitted`, `Approved`, and `Rejected`
* **FR-3:** The system must allow Team Members to edit or delete their own expenses only while the status is `Draft` or `Rejected`, and submit them to change the status to `Submitted`
* **FR-4:** The system must allow Project Managers to approve `Submitted` expenses (locking them from further edits) or reject them with a `rejection_note`
* **FR-5:** The system must allow Accounting Clerks to view all `Approved` expenses marked `billable = true` so they can be included in Client invoices (Feature 5).


## Assumptions 
* Expenses are tied to a `Project` (from Feature 2) and an `Employee` (from Feature 1), rather than a sepcific `ProjectRole`, because an expense like travel or software applies to the project as a whole.
* All monetary amounts are stored as exact decimals in USD (`amount > 0`).
* Once an `Expense` is `Approved`, it is locked so historical billing totals cannot be altered before or after invoicing. 
* Not every project expense is billed to the client; the `billable` boolean flag distringuishes client-billable expenses from internal company costs. 

## Edge Cases 
* **Invalid Amount:** If a Team Member enters `0` or a negative `amount`, or leaves required fields (`project_id`,`expense_date`, `category`, `amount`, `description`) blank, the system must reject the submission and display a validation error. 
* **Editing a locked Expense:** If a Team Member attempts to edit or delete an `Expense` whose status is `Submitted` or `Approved`, the system must block the action.
* **Rejecting Without a Reason:** If a Project Manager attempts to reject an `Expense` without providing a `rejection_note`, the system must reject the request and prompt for a reason. 
* **Logging Expense on an Unassigned or Inactive Project:** If a Team Member attempts to submit an `Expense` for a `project_id` they are not assigned to (or for a `Completed`/`Archived` project), the system must block the request
* **Self-Approval Prevention:** A Project Manager who logs an expense must not be able to approve their own `Expense` (an Admin/Owner must approve it).

## Success Criteria 
* A Team Member can log and submit a project expense in under 1 minute.
* 100% of `Approved` expenses are locked against modification by Team Members
* 100% of `Approved` expenses are `billable = true` are accurately surfaced for client invoicing in Feature 5.

## Key Entities 
* **Expense** 
* **Project** (referenced from Feature 2)
* **Employee** (referenced from Feature 1)

## Initial Data Model 
                      
### Expense

| Field Name | Data Type | Description | 
|---|---|---| 
| `expense_id` | Integer | Primary key for the expense | 
| `project_id` | Integer | Foreign key referencing `Project.project_id` | 
| `employee_id` | Integer | Foreign key referencing `Employee.employee_id` who incurred the expense |
| `expense_date` | Date | Calendar date the expense was made (`YYYY-MM-DD`) |
| `category` | Enum | Expense category (`Travel`, `Software`, `Equipment`, `Meals`, `Other`) | 
| `amount` | Decimal | Exact dollar amount of the expense (muust be `>0`) | 
| `description` | String | Brief explanation of the expense | 
| `billable` | Boolean | Whether this expense should be billed to the client (`true` = Billable, `false` = Internal) | 
| `status` | Enum | Current workflow state (`Draft`, `Submitted`, `Approved`, `Rejected`) | 
| `rejection_note` | String | Optional manager feedback explaining why an expense was rejected (nullable) | 
| `approved_by` | Integer | Foreign key referencing `Employee.employee_id` of the approving manager (nullable) |


## Gherkin AC 

### US-4.1 
**Scenario:Team Member logs a billable project expense**
**Given:** "Adrian Mascote" is logged in and assigned to the active project "Cloud Migration" 
**When:** "Adrian Mascote" selects "Cloud Migration", enters date "2026-11-19", category "Software", amount "49.44", description "Cloud testing license", checks "Billable"
**And** clicks "Save Draft" 
**Then:** the system creates a new Expense record with status "Draft" and billable set to true 
**And:** the expense appears in "Adrian Mascote"'s expense list. 

### US-4.2 
**Scenario: Team Member submits a draft expense for approval** 
**Given** "Adrian mascote" has a "Draft" Expense for "$49.99" for "Cloud Migration"
**When** "Adrian Mascote" clicks "Submit for Approval" 
**Then** the Expense status updates to "Submitted" 
**And** the expense is locked from further edits by "Adrian Mascote" 

### US-4.3
**Scenario: Project Manager approves a submitted expensee**
**Given** a Project Manager is on the Expense Approval screen viewing "Adrian Mascote"'s "Submitted" expense of "$49.99"
**When** the Project Manager clicks "Approve"
**Then** the Expense status updates to "Approved" with approved_by set to the Project Manager's ID
**And** the expense is locked and available for client invoicing

# Data Ownership & Isolation 

* Team Members own and manage their own `Expense` records; Project Managers can view and approve/reject submitted expenses across projects; Accounting Clerks have Read-only access to `Approved` expenses for invoicing. 

| Rule | Requirement | 
| --- | --- | 
| **Read scope** | Team Members only see `Expense` rows where `employee_id = req.user.id`; Project Managers and Admin/Owners see all expenses; Accounting Clerks see `Approved` expenses |
| **Write scope** | Team Members can only update/delete their own `Draft` or `Rejected` rows; Project Managers and Admin/Owners can only update `status`, `rejection_note`, and `approved_by` on `Submitted` rows | 
| **Create scope** | New `Expense` rows are owned by `req.user.id` and must reference an active `project_id`assigned to `req.user.id`| 
| **Cross-user-access** | Team Member requesting another employee's `Expense` -> `404` (not `403`) | 
| **UI scope** | UI displays only the signed-in user's assigned projects in the expense form, and shows the Expense Approval queue only to Project Managers and Admin/Owners | 
| **Implementation** | Shared helper in `app/authorization/` - do not dulicate scope in every controller | 


## API Requirements 

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/expenses` | Yes | Retrieves expenses (own expenses for Team Members; filterable by project/status/billable for PM/Admin/Accounting)  |
| `POST` | `/api/expenses` | Yes | Creates a new `Draft` Expense for the signed-in employee |
| `PUT` | `/api/expenses/{expense_id}` | Yes | Updates a `Draft` or `Rejected` expense (owner only) | 
| `DELETE` | `/api/expenses/{expense_id}` | Yes | Deletes a `Draft` expense (owner only) |
| `PATCH` | `/api/expenses/{expense_id}/submit` | Yes | Changes a `Draft` or `Rejected` expense's status to `Submitted` (owner only) |
|`PATCH` | `/api/expenses/{expense_id}/approve` | Yes | Approves a `Submitted` expense and records `approved_by` (PM/Admin only) |
| `PATCH`| `/api/expenses/{expense_id}/reject` | Yes | Rejects a `Submitted` expense with a required `rejection_note` (PM/Admin only) |



**Create request body(`POST /api/expenses`):**
```json 
{
 "project_id" : 10,
 "expense_date" : "2026-11-19", 
 "category" : "Software",
 "amount" : 49.99,
 "description" : "Cloud testing license",
 "billable" : true 
}

```


## Screen Requirements

### [View: My Expenses Screen] - route name `expenses-my`
* Heading / purpose: Project expense logging and submission (Team Member)
* Primary action: **Log Expense** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icon actions with labels: Add/Edit Expense dialog (`project_id` dropdown, `expense_date`, `category` dropdown, `amount`, `description`, `billable` checkbox, `Save Draft` button ); row-level Edit, Delete, and **Submit for Approval** actions; status badge (`Draft`, `Submitted`, `Approved`, `Rejected`) with visible `rejection_note` banner
* **Empty state:** "No expenses logged yet."
* Loading / error: skeleton/progress; `<v-alert type="error">` for failures 



### [View: Expense Approval Screen] - route name `expenses-approvals`
* Heading / purpose: Manager review queue for submitted project expenses (Project Manager / Admin)
* Primary action: **Approve** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icnn actions with labels: Project/Status filter bar; Submitted Expenses table displaying Employee, Project, Category, Amount, Billable flag, and Description, with **Approve** and **Reject** buttons (Reject opens dialog requiring `rejection_note`)
* **Empty state:** "No expenses awaiting approval."
* Loading / error: skeleton/progress; `<v-alert type="error">` for failures

## Test Coverage Map 

| Story | Scenario | Test file | Test name | 
|-------|----------|-----------|-----------|
| US-4.1 | Team Member logs a billable project texpense | `backend/tests/expenses.test.js` |  `Team member creates a draft billable expense` | 
| US-4.2 | Team Member submits a draft expense for approval | `backend/tests/expenses.test.js` | `Team member submits draft expense for approval` |
| US-4.3 | Project Manager approves a submitted expense | `backend/tests/expenses.test.js` | `Project manager approves submitted expense` |



## Agent Implementation request 

```text
Implement Feature 4 from @Invoicing System/feature-4-expense-tracking.md on branch `invoicing system`. 


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
* Unloading or storing scanned receipt image files (manual entry of amount and description only)
* Direct employee payroll reimbursement processing 
* Combining approved expenses with approved hours onto a client invoice (handled in Feature 5: Incvoicing)

