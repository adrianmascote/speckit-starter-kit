# Feature 3: Time Entry & Approval

## User Stories 

### US-3.1: Log Work Hours
 
**As a** Team Member 
**I want to** log my daily hours against one of my assigned project roles 
**So that** my billable time is accurately recorded for the client. 

**Priority:** P1 
**Independent test:** Verify a Team Member can create a new draft `TimeEntry` linked to one of their assigned `ProjectRole` records
**Acceptance scenarios:** see ### US-3.1 under Gherkin AC 

### US-3.2: Edit and Submit Time Entries
**As a** Team Member 
**I want to** edit my draft or rejected time entries and submit them for approval
**So that** my Project Manager can review and approve my weekly hours. 

**Priority:** P1
**Independent test:** Verify a Team Member can update a draft/rejected `TimeEntry` and change its status to `Submitted`. 
**Acceptance scenarios:** see ### US-3.2 under Gherkin AC 

### US-3.3: Approve or Reject Submitted Time Entries
**As a** Project Manager
**I want to** review submitted time entries and either approve them or reject them with a note
**So that** only verified hours move forward to client invoicing. 

**Priority:** P1
**Independent test:** Verify a Project Manager can change a `Submitted` entry to `Approved` or `Rejected` (with a required rejection reason)
**Acceptance scenarios:** see ### US-3.3 under Gherkin AC 

### US-3.4: View Missing Timesheet Report 
**As a** Project Manager 
**I want to** view a list of active employees who have not submitted time entries for the current week
**So that** I can use their email addresses to follow upbefore invoicing.

**Priority:** P1
**Independent test:** Verify a Project Manager can retrieve a list of active employees with zero submitted hours for a selected week, including their email addresses. 
**Acceptance scenarios:** see ### US-3.4 under Gherkin AC 

## Functional Requirements 
* **FR-1:** The system must allow Team Members to log hours (`work_date`, `hours`, `description`) strictly against `ProjectRole` assignments that belong to them. 
* **FR-2:** The system must track each `TimeEntry` through a defined status lifecycle: `Draft`, `Submitted`, `Approved`, and `Rejected`.
* **FR-3:** The system must allow Team Members to edit or delete their own time entries only while the status is `Draft` or `Rejected`, and submit them to change the status to `Submitted`.  
* **FR-4:** The system must allow Project Managers to approve `Submitted` time entries (locking them from further edits) or reject them with a `rejection_note`.
* **FR-5:** The system must provide Project Managers with a Missing Time report for any given week showing active employees who have not submitted hours, along with their email addresses. 


## Assumptions 
* Each `TimeEntry` links to a single `ProjectRole`, which automatically identifies the `project_id`, `employee_id`, `role_name`, and `hourly_rate`(from Features 1 and 2).
* Hours are logged in decimal increments (e.g., `0.25` for 15 minutes, `1.5` for 1 hour 30 minutes).
* Once a `TimeEntry` is `Approved`, it is locked so historical billing totals cannot be altered before or after invoicing.
* Employees who hold multiple roles on the same project log separate `TimeEntry` rows for each role they worked. 

## Edge Cases 
* **Invalid Hours:** If a Team Member enters `0`, a negative number, or more than `24` hours for a single day, the system must reject the submission and display a validation error.
* **Editing a locked Entry:** If a Team Member attempts to edit or delete a `TimeEntry` whose status is `Submitted` or `Approved`, the system must block the action. 
* **Rejecting Without a Reason:** If a Project Manager attempts to reject a `TimeEntry` without providing a `rejection_note`, the system must reject the request and prompt for a reason. 
* **Logging Time on an Unassigned Role:** If a Team Member attempts to submit a `TimeEntry` for a `project_role_id` that does not corrspond to them (or for an `Archived/Completed` project), the system must block the request.
* **Self-Approval Prevention:** A Project Manager who also logs hours as a Team Member must not be able to approve their own `TimeEntry`(or must have an Admin/Owner approve it)

## Success Criteria 
* A Team Member can log and submit a full week of hours across their assigned project roles in under 3 minutes. 
* 100% of `Approved` time entries are locked and cannot be modified by Team Members
* The Missing Time report accurately identifies 100% of active employees with no submitted hours for the selected week and displays their email addresses. 

## Key Entities 
* **TimeEntry** 
* **ProjectRole** (referenced from Feature 2)
* **Employee** (referenced from Feature 1)

## Initial Data Model 
                      
### TimeEntry

| Field Name | Data Type | Description | 
|---|---|---| 
| `time_entry_id` | Integer | Primary key for the time entry | 
| `project_role_id` | Integer | Foreign key referencing `ProjectRole.project_role_id` (links employee, project, role, and hourly rate) | 
| `work_date` | Date | Calendar date the work was performed (`YYYY-MM-DD`) |
| `hours` | Decimal | Number of hours worked (must be ` > 0` and `<=24`) |
| `description` | String | Brief summary of the work completed | 
| `status` | Enum | Current workflow state (`Draft`, `Submitted`, `Approved`, `Rejected`) | 
| `rejection_note` | String | Optional manager feedback explaining why an entry was rejected (nullable) | 
| `approved_by` | Integer | Foreign key referencing `Employee.employee_id` of the approving manager (nullable) | 



## Gherkin AC 

### US-3.1 
**Scenario: Team Member logs daily hours to an assigned project role**
**Given:** "Adrian Mascote" is logged in and assigned as "Developer" on "Cloud Migration" 
**When:** "Adrian Mascote" is selects "Cloud Migration - Developer", enters date "2026-11-18", hours "6.5", description "API setup", 
**And** clicks "Save Draft" 
**Then:** the system creates a new TimeEntry record with status "Draft" 
**And** the entry appears in "Adrian Mascote"'s weekly timesheet

### US-3.2 
**Scenario: Team Member submits a draft time entry for approval** 
**Given** "Adrian mascote" has a "Draft" TimeEntry for "6.5" hours on "2026-11-18"
**When** "Adrian Mascote" clicks "Submit for Approval" 
**Then** the TimeEntry status updates to "Submitted"
**And** the entry is locked from further edits by "Adrian Mascote" 

### US-3.3
**Scenario: Project Manager rejects a submitted time entry with a note**
**Given** a Project Manager is on the Time Approval screen viewing "Adrian Mascote"'s "Submitted" entry
**When** the Project Manager clicks "Reject", enters "Please split QA and Dev hours"
**And** clicks "Confirm" 
**Then** the TimeEntry status updates to "Rejected" with the `rejection_note` saved
**And** "Adrian Mascote" can edit and resubmit the entry

### US-3.4
**Scenario: Project Manager views employees missing time for the week**
**Given** "Alex Chen" is an active employee with zero submitted hours for the week of "2026-11-16"
**When** a Project Manager opens the Missing Time report for "2026-11-16" 
**Then** the system lists "Alex Chen" along with their amail address "alex@consulting.com"


# Data Ownership & Isolation 

* Team Members own and manage their own `TimeEntry` records; Project Managers can view and approve/reject submitted entries across projects; Accounting Clerks have Read-only access to `Approved` entries for invoicing. 

| Rule | Requirement | 
| --- | --- | 
| **Read scope** | Team Members only see `TimeEntry` rows where `ProjectRole.employee_id = req.user.id`; Project Managers and Admin/Owners see all entries; Accounting Clerks see `Approved` entries |
| **Write scope** | Team Members can only update/delete their own `Draft` or `Rejected` rows; Project Managers and Admin/Owners can only update `status`, `rejection_note`, and `approved_by` on `Submitted` rows | 
| **Create scope** | New `TimeEntry` rows must reference a `project_role_id` belonging to `req.user.id` | 
| **Cross-user-access** | Team Member requesting another employee's `TimeEntry` -> `404` (not `403`) | 
| **UI scope** | UI displays only the signed-in user's assigned project roles in the time entry form, and shows the Approval/Missing Time views only to Project Managers and Admin/Owners| 
| **Implementation** | Shared helper in `app/authorization/`- do not duplicate scope in every instance | 


## API Requirements 

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/time-entries` | Yes | Retrieves time entries (own entries for Team Members; filterable by project/status/week for PM/Admin)  |
| `POST` | `/api/time-entries` | Yes | Creates a new `Draft` time entry for the signed-in employee |
| `PUT` | `/api/time-entries/{time_entry_id}` | Yes | Updates a `Draft` or `Rejected` time entry (owner only) | 
| `DELETE` | `/api/time-entries/{time_entry_id}` | Yes | Deletes a `Draft` time entry (owner only) |
| `PATCH` | `/api/time-entries/{time_entry_id}/submit` | Yes | Changes a `Draft` or `Rejected` entry's status to `Submitted` (owner only) |
|`PATCH` | `/api/time-entries/{time_entry_id}/approve` | Yes | Approves a `Submitted` entry and records `approved_by` (PM/Admin only) |
| `PATCH`| `/api/time-entries/{time_entry_id}/reject` | Yes | Rejects a `Submitted` entry with a required `rejection_note` (PM/Admin only)
| `GET` | `/api/reports/missing-time?week={YYYY-MM-DD}`| Yes | Returns active employees with no submitted hours for the given week and their email addresses |



**Create request body(`POST /api/projects/{project_id}/roles`):**
```json 
{
 "project_role_id" : 1,
 "work_date" : "2026-11-18", 
 "hours" : 6.5,
 "description" : "Implemented client and project setup API routes" 
}

```


## Screen Requirements

### [View: My Timesheet Screen] - route name `timesheet-my`
* Heading / purpose: Weekly time entry logging and submission (Team Member)
* Primary action: **Log Time** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icon actions with labels: Add/Edit Time Entry dialog (`project_role_id` dropdown, `work_date`, `hours`, `description`, `Save Draft` button); row-level Edit, Delete, and **Submit for Approval** actions; status badge (`Draft`, `Submitted`, `Approved`, `Rejected`) with visible `rejection_note` tooltip/banner
* **Empty state:** "No time entries logged for this week."
* Loading / error: skeleton/progress; <v-alert type="error"> for failures 

## Screen Requirements

### [View: Time Approval & Missing Time Screen] - route name `timesheet-approvals`
* Heading / purpose: Manager review queue for submitted hours and missing-timesheet tracking (Project Manager/Admin)
* Primary action: **Approve** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icnn actions with labels: Week selector; Submitted Entries table with **Approve** and **Reject** buttons (Reject opens dialog requiring `rejection_note`); Missing Time tab listing employee `full_name` and `email`
* **Empty state:** "No time entries awaiting approval." / "All active employees have submitted time for this week."
* Loading / error: skeleton/progress; <v-alert type="error"> for failures

## Test Coverage Map 

| Story | Scenario | Test file | Test name | 
|-------|----------|-----------|-----------|
| US-3.1 | Team Member logs daily hours against an assigned project role | `backend/tests/timeEntries.test.js` |  `Team member creates a draft time entry` | 
| US-3.2 | Team Member submits a draft time entry for approval | `backend/tests/timeEntries.test.js` | `Team member submits draft time entry for approval` |
| US-3.3 | Project Manager rejects a submitted time entry with a note | `backend/tests/timeEntries.test.js` | `Project manager rejects submitted time entry with note` |
| US-3.4 | Project Manager views employees missing time for the week | `backend/tests/clients.test.js` | `Project manager retrieves missing time report with emails` | 


## Agent Implementation request 

```text
Implement Feature 3 from @Invoicing System/feature-3-time-entry-approval.md on branch `invoicing system`. 


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
* Automated email sending for missing-timesheet reminders (manula email follow-up using the displayed email addresses is in scope; automated notifications are out of scope)
* Tracking reimbursable project expenses (handled in Feature 4: Expense Tracking)
* Calculating invoice totals or generating client invoices (handled in Feature 5: Invoicing)

