# Feature 2: Client Project Setup

## User Stories 

### US-2.1: Create a new Project 
 
**As a** admin/Owner
**I want to** add a new Project to the System
**So that** the team can start logging hours on the project

**Priority:** P1 
**Independent test:** Verify an admin can create a new Project 
**Acceptance scenarios:** see ### US-2.1 under Gherkin AC 

### US-2.2: Add a new Client 
**As a** Admin/Owner
**I want to** add a new client to the system
**So that** I can start assigning projects to that client's profile 

**Priority:** P1
**Independent test:** Verify an admin can add a new client to the system 
**Acceptance scenarios:** see ### US-2.2 under Gherkin AC 

### US-2.3: Edit a Project 
**As a** Admin/Owner
**I want to** Update a project's details 
**So that** I can assign employees to specific roles on that project and set the hourly billing rate for each role.

**Priority:** P1
**Independent test:** Verify an Admin can assign employees to a project 
**Acceptance scenarios:** see ### US-2.3 under Gherkin AC 

### US-2.4: Edit client's profile 
**As a** Admin/Owner 
**I want to** update a client's details 
**So that** their contact information stays up to date in my system 

**Priority:** P1
**Independent test:** Verify an admin can update a client's contact information. 
**Acceptance scenarios:** see ### US-2.4 under Gherkin AC 

## Functional Requirements 
* The system must allow Admin/Owner to add a new Project to the system
* The system must allow Admin/Owner to edit a project's details
* The system must allow Admin/Owner to add a new Client to the system 
* The system must allow Admin/Owner to edit a Client's profile
* The system must allow an Admin/Owner to define the project specific roles and set an hourly billing rate for each `Project + Role` combination 
* The system must allow an Admin/Owner to assign one or more employees to a project, including assigning an employee multiple roles on the same project. 


## Key Entities 
* **Client** 
* **Employee** (referenced from feature 1)
* **ProjectRole** 
* **Project** 


## Initial Data Model 
                      
### Client 

| Field Name | Data Type | Description | 
|---|---|---| 
| `client_id` | Integer | Primary key for the client | 
| `client_name` | String | Official name of the client company | 
| `billing_info` | String | Billing address and primary contact details for invoices |

### Project 
| Field Name | Data Type | Description | 
|---|---|---|
| `project_id` | Integer | Primary key for the project | 
| `client_id` | Integer | Foreign key referencing `Client.client_id` | 
| `project_name` | String | Name of the consulting project | 
| `status` | String | Current project status (`Active`, `Completed`, `Archived`) | 

### ProjectRole

| Field Name | Data Type | Description |
|---|---|---|
| `project_role_id` | Integer | Primary key for the project role assignment |
| `project_id` | Integer | Foreign key referencing `Project.project_id` | 
| `employee_id` | Integer | Foreiign key referencing `Employee.employee_id` (from feature 1) | 
| `role_name` | String | Role title on this project (e.g., `Developer`, `QA Tester`, `Designer`) | 
| `hourly_rate` | Decimal | Hourly billing rate charged to the client for this `Project + Role` |
 

## Assumptions 
* A client must exist in the system before a Project can be created and linked to them
* Hourly billing rates are tied to a specific `Project + Role` combination, not to an individual employee globally
* A single active employee can be assigned to multiple roles in the same project (each stored as its own `ProjectRole` row with its own `hourly_rate`).
* Projects typically run 12-18 months and remain in the system for historical invoicing even after completion

## Edge Cases 
* **Missing or Invalid Input:** If an Admin/Owner leaves required fields blank (`client_name`, `email`, `project_name`, `role_name`, or `hourly_rate`) or enters a negative/zero `hourly_rate`, the system must reject the submission and display a validation error 
* **Duplicate Project Role Assignment:** if an Admin/Owner tries to assign the exact same `employee_id` to the exact same `role_name` twice on the same `project_id`, the system must block the duplicate entry. 
* **Assigning an Inactive Employee:** If an Admin/Owner attempts to assign an employee whose `status` is `Inactive` to a project, the system must prevent the assignment and only show `Active` employees in the selection list 
* **Deleting a Role with Logged Hours:** If an Admin/Owner tries to remove a `ProjectRole` that already has logged time entries (from Feature 3), the system must block deletion so historical billing data is not broken. 


## Success Criteria 
* An Admin/Owner can create a new client and link a new project to that Client in under 2 minutes. 
* 100% of `ProjectRole` assignments accurately store a valid positive `hourly_rate` and lnk to an active `Employee`. 
* Team Members can immediately see only the projects and roles assigned to them upon logging in. 


## Gherkin AC 

### US-2.1 
**Scenario: Admin creates a new project linkd to an existing client**
**Given** An Admin/Owner is on the Project Management screen and client "Acme Corp" exists
**When** the Admin/Owner enters project name "Cloud Migration", selects, "Apache Corp" from the Client dropdown
**And** clicks "Save" 
**Then** the system creates a new Project record linked to "Acme Corp" with status "Active"
**And** "Cloud Migration" appears in the Projects list 

### US-2.2 
**Scenario: Admin addds a new client** 
**Given** an Admin/Owner is on the Client Management screen
**When** the Admin/Owner enters client name "Apache Corp", billing info "100 Main St, OKC", and email "billing@acme.com"
**And** clicks "Save" 
**Then** the system creates a new Client record in the database
**And** "Apache Corp" is available  in the Client dropdown when creating a Project

### US-2.3
**Scenario: Admin edits a project to assign an empoyee with a role and hourly rate**
**Given** an Admin/Owner is viewing the "Cloud Migration" project and "Adrian Mascote" is an active employee
**When** the Admin/Owner clicks "Add" under active roles, selects "Adrian Mascote", enters role "Developer" and hourly rate "150.00",
**And** clicks "Save
**Then** the system creates a new ProjectRole record linking "Adrian Mascote" to "Cloud Migration" at "$150.00/hr"
**And** "Adrian Mascote - Developer ($150.00/hr)" appears in the project's roles list

### US-2.4
**Scenario: Admin updates a client's contact information**
**Given** an Admin/Owner is viewing the profile for client "Apache Corp"
**When** the Admin/Owner updates the email to anemail@gmail.com" 
**And** clicks "Save"
**Then** the system updates the Client record in the database
**And** the updated email "anemail@gmail.com" is displayed on the Client profile


# Data Ownership & Isolation 

* Admin/Owner has full management control over all `Client`, `Project`, and `ProjectRole` records; Team Members only see projects and roels assigned to them.

| Rule | Requirement | 
| --- | --- | 
| **Read scope** | Admin/Owner, Project Managers, and Accounting Clerks see all Clients and Projects; Team Members only see Projects and `ProjectRole` rows where `employee_id = req.user.id` |
| **Write scope** | Only authenticated users with `system_role = 'AdminOwner'` can create or update `Client`, `Project`, and `ProjectRole` records | 
| **Create scope** | New `Client`, `Project`, and `ProjectRole` rows created exclusively by an authenticated `AdminOwner | 
| **Cross-user-access** | Team Member requesting a Project they are not assigned to -> `404` (not `403`) | 
| **UI scope** | UI shows only Clients, Projects, and roles permitted for the signed-in user's `system_role` | 
| **Implementation** | Shared helper in `app/authorization/`- do not duplicate scope in every instance | 


## API Requirements 

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/clients` | Yes | Retrieves the list of all clients  |
| `POST` | `/api/clients` | Yes | Creates a new client profile (Admin/Owner only) |
| `PUT` | `/api/clients/{client_id}` | Yes | Updates an existing client's details (Admin/Owner only) | 
| `GET` | `/api/projects` | Yes | Retrieves projects (all for Admin/PM/Accounting; assigned only for Team Members) |
| `POST` | `/api/projects` | Yes | Creates a new project linked to a `client_id` (Admin/Owner only) |
|`PUT` | `/api/projects/{project_id}` | Yes | Updates project details or status (Admin/Owner only) |
| `POST`| `/api/projects/{project_id}/roles` | Yes | Assigns an employee to a role with an `hourly_rate` on a project (Admin/Owner only)
| `PUT` | `/api/projects/{project_id}/roles/{project_role_id}` | Yes | Updates a role name or `hourly_rate` (Admin/Owner only) |
| `DELETE` | `/api/projects/{project_id}.roles/{project_role_id}` | Yes | Removes an unused role assignment from a project (Admin/Owner only) |


**Create request body(`POST /api/projects/{project_id}/roles`):**
```json 
{
 "employee_id" : 7,
 "role_name" : "Developer", 
 "hourly_rate" : 150.00
}

```


## Screen Requirements

### [View: Client Management Screen] - route name `clients-manage`
* Heading / purpose: Employee directory and profile management (Admin/Owner)
* Primary action: **Add Client** (`oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icon actions with labels: Project form (`project_name`, `Client` dropdown, `status`); Active Roles sub-table with **Add**, **Edit**, and **Delete** acctions (`employee_id` dropdown of active employees, `role_name`, `hourly_rate`)
* **Empty state:** "No projects found." / "No roles assigned to this project." 
* Loading / error: skeleton/progress; <v-alert type="error"> for failures 

## Test Coverage Map 

| Story | Scenario | Test file | Test name | 
|-------|----------|-----------|-----------|
| US-2.1 | Admin creates a new project linked to an existing client | `backend/tests/projects.test.js` |  `Admin creates a new project` | 
| US-2.2 | Admin adds a new client | `backend/tests/clients.test.js` | `Admin creates a new client` |
| US-2.3 | Admin edits a project to assign an employee with a role and hourly rate | `backend/tests/projects.test.js` | `Admin assigns employee role a hourly rate to project` |
| US-2.4 | Admin updates a client's contact information | `backend/tests/clients.test.js` | `Admin updates client contact details` | 


## Agent Implementation request 

```text
Implement Feature 2 from @Invoicing System/feature-2-client-project-setup.md on branch `invoicing system`. 


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
* Logging daily hours against project roles (handled in Feature 3: Time Entry & Approval)
* Tracking reimbursable project expenses (handled in Feature 4: Expense Tracking)
* Generating ro sending client invoices (handled in Feature 5: Invoicing)




