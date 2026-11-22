# Feature 1: Employee Management 

## User Stories 

### US-1.1: Add New Employee

**As a** warehouse admin 
**I want to** add a new Employee to the System
**So that** we can start including the Employee in Projects. 

**Priority:** P1 
**Independent test:** Verify an admin can create a new Employee profile 
**Acceptance scenarios:** see ### US-1.1 under Gherkin AC 

### US-8.2: Edit Employee Profile
**As a** warehouse admin 
**I want to** edit an employee's profile 
**So that** There information stays up to date in the System

**Priority:** P1
**Independent test:** Verify an admin can update an Employee's address and it saves to the System
**Acceptance scenarios:** see ### US-1.2 under Gherkin AC 

### US-8.3: Inactivate Employee Profile
**As a** warehouse admin
**I want to** Set an Employee's profile as inactive 
**So that** we can keep track of which employees are no longer available. 

**Priority:** P1
**Independent test:** Verify an Admin can set a profile as inactive
**Acceptance scenarios:** see ### US-1.3 under Gherkin AC 

## Functional Requirements 
* The system must allow administrators to add new employee profiles to the database
* The system must allow administrators to edit an employee's profile with their current information
* The system must allow administrators to set an employee's profile as `inactive`

## Key Entities 
* Employee


## Initial Data Model 
                      
### Employee

| Field Name | Data Type | Description | 
|---|---|---| 
| `employee_id` | Integer | Primary key for the employee referenced | 
| `phone_number` | String | The Employee's phone number | 
| `address` | String | References the employee's address | 
| `user_name` | String | Username referencing the employee's id for login |
| `full_name` | String | Full name of the Employee | 
| `email` | String | Contains email of the employee | 
| `system_role` | Enum | References the role/roles of the employee | 
| `status` | Boolean | Contains an Employee's profile status (`true` = Active, `false` = Inactive) | 


## Assumptions 
* Admin profile had been created beforehand 
* Admin is operating on a laptop

## Edge Cases 
**Duplicates** The Admin creates a duplicate of the exact same profile 
**Linked Data** An Admin changes an employee's status to `inactive` while they are still assigned an active project


## Success Criteria 
* An Admin/Owner can create a new employee profile in under 1 minute. 
* 100& of employees are routed to their corresponding role-based menu after login and cannot access unauthorized screens 
* 0 invoices are lost or broken when an employee is 

## Key Entities 
* **Employee** 


## Initial Data Model 
                      
### Employee

| Field Name | Data Type | Description | 
|---|---|---| 
| `employee_id` | Integer | Primary key for the employee referenced | 
| `phone_number` | String | The Employee's phone number | 
| `address` | String | References the employee's address | 
| `user_name` | String | Username referencing the employee's id for login |
| `full_name` | String | Full name of the Employee | 
| `email` | String | Contains email of the employee | 
| `system_role` | Enum | References the role/roles of the employee | 
| `status` | Boolean | Contains an Employee's profile status (`true` = Active, `false` = Inactive) | 

## Gherkin AC 

### US-1.1 
**Scenario: Admin adds a new employee**
**Given** an Admin/Owner is on the Employee Management Screens
**When** the Admin/Owner enters valid details for "Adrian Mascote" with email "anemail@gmail.com", username "Adrianmascote", and role "TeamMember"
**And** clicks "Save" 
**Then** the system creates a new Employee record in the database with status "Active" 
**And** "Adrian Mascote" appears in the Employee Directory list 

### US-1.2 
**Scenario: Admin updates an employee's details** 
**Given** an Admin/Owner is viewing the profile for "Sam Rivera"
**When** the Admin/Owner updates the address to "456 Oak Ave" and clicks "Save" 
**Then** the system updates the Employee record in the database 
**And** the updated address "456 Oak Ave" is displayed on the profile. 

### US-1.3
**Scenario: Admin deactivates an employee's profile 
**Given** "Adrian Mascote" has an active profile and had previously logged hours on a project 
**When** the Admin/Owner sets "Adrian Mascote"'s status to "Inactive" and clicks "Save"
**Then** the Employee record is marked as "Inactive" in the databse 
**And** Adrian Mascote" can no longer log into the system
**And** all previous hours logged by Adrian mascote are kept 

# Data Ownership & Isolation 

* Admin/Owner manages all `Employee` records; non=admin users can only view their own profile, (except Project Managers, who have read-only access to active employee names and emails for staffing and reminders).

| Rule | Requirement | 
| --- | --- | 
| **Read scope** | Admin/Owner sees all rows; Project Manager sees active employee names/emails; Team Members and Accounting Clerks only see rows where `employee_id = req.user.id` |
| **Write scope** | Only authenticated users with `system_role = `AdminOwner` can create, update, or deactivate employee records | 
| **Create scope** | New `Employee` rows created exlusively by an authorized `AdminOwner` | 
| **Cross-user-access** | Non-admin requesting another user's profile -> `404` | 
| **UI scope** | UI shows only menus and employee data permitted for the signed in users `system_role` | 
| **Implementation** | Shared helper in `app/authorization/`- do not duplicate scope in every instance | 


## API Requirements 

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `POST` | `/api/auth/login` | No | Authenticates an employee and returns a session  |
| `GET` | `/api/employees` | Yes | Retrieves the list of employees (Admin/Owner and PM read access) |
| `GET` | `/api/employees/{employee_id}` | Yes | Retrieves a single employee's profile | 
| `POST` | `/api/employees` | Yes | Creates a new employee profile (Admin/Owner only) |
| `PUT` | `/api/employees/{employee_id}` | Yes | Updates an employee's details, `system_role` or `status` (Admin/Owner only) |


**Create request body:**
```json 
{
 "full_name" : "Adrian Mascote", 
 "email" : "anemail@gmail.com", 
 "phone_number" : "405-555-0192",
 "address" : 123 Main St, Oklahoma City Ok
 "user_name" : "Adrian Mascote", 
 "password" : "TempPassword123!", 
 "system_role" : "TeamMember" 
}

```


## Screen Requirements

### [View: Employee Management Screen] - route name `employees-manage`
* Heading / purpose: Employee directory and profile management (Admin/Owner)
* Primary action: **Add Employee** `oc-cta` when it is primary labeled CTA)
* Fields, dialogs, icon actions with labels: Add/Edit Employee diaglog (full_name, email, phone_number, address, user_name, system_role dropdown, status Active/Inactive toggle); row level edit icon 
* **Empty state:** "No employees found" 
* Loading / error: skeleton/progress; <v-alert type="error"> for failures 

## Test Coverage Map 

| Story | Scenario | Test file | Test name | 
|-------|----------|-----------|-----------|
| US-1.1 | Admin adds a new employee | `backend/tests/empyees.teset.js` |  `Admin creates a new employee profile` | 
| US-1.2 | Admin updates an employee's details | `backend/tests/employees.test.js` | `Admin updates employee's details` |
| US-1.3 | Admin deactivates an employee profile | `backend/tests/employees.test.js` | `Admin deactivates employee profile` | 


## Agent Implementation request 

```text
Implement Feature 1 from @Invoicing System/feature-1-employee-management.md on branch `invoicing system`. 


Follow layer order in @features/framework.md (models -> routes -> backend tests -> frontend -> frontend tests).
Map every Gherkin scenario in the Test Coverage Mape; run `npm test` before finishing.
If API routes, payloads, schema, or product rules changed per this spec update @features/reference/api.md, @features/reference/data-model.md, and/or @features/reference/behavior.md in the same PR to match the code. 
```


## Definition of Done 

* [ ] Backend and frontend implemented per this spec (FR-1 through FR-3 satisfied)
* [ ] Success Criteria met
* [ ] All mapped tests pass (`npm test`)
* [ ] Test Coverage Map complete 
* [ ] `features/reference/data-model.md` updated (if schema changed)
* [ ] `features/reference/api.md` updated (if API changed)
* [ ] `features/ reference/behavior.md` updated (if product rules changed)

## Out of Scope 
* Self-service employee registration 
* Payroll, salary, or HR benefits management
* Single Sign-On (SSO) or third party authentication logins


