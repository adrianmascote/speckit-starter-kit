# Feature 16: Put Away Orders from Suppliers 

## User Stories 

### US-16.1: Store Received Items 
**As a** worker
**I want to** know exactly which storage zone to put the new received items in 
**So that** the warehouse stays organized and items are easy to find

**Priority:** P1
**Independent test:** Verify the system assigns a valid storage zone (`zone_id`) for the received items and updates the item's location status 

### US-16.2: Update Available Inventory 
**As a** manager 
**I want to** ensure inventory amounts update automatically when items are put away. 
**So that** I have an accurate count of what is available to sell to customers. 

**Priority:** P1 
**Independent test**Verify the system increases the available inventory for the specific item put away 

## Functional Requirements 
* The system must allow a worker to view a list of items currently sitting on the receiving dock waiting to be put away
* The system must allow a worker to scan a received item, input the quantity, and scan the barcode of the `StorageZone` where they place it. 
* The system must record the `worker_id` of the person who moved the items and timestamp the exact `movement_date`. 
* The system must automatically trigger an update to the inventory, adding the put-away quantity to the item's total `qty_on_hand`. 

## Assumptions 
* The items have already been officially checked in via the Receiving Dock (Feature 15).
* The warehouse has predefined storage zones (aisles, racks, bins) stored in a `StorageZone` database table. 

## Edge Cases 
* **Split Put-Away:** 50 boxes arrive, but the assigned shelf only holds 30. The worker must put 30 in zone A and the remaining 20 in zone B. 
* **Wrong Item Scan:** A worker attempts to put away an item that the system does not recognize as part of a pending delivery. 

## Key Entities 
* Worker / Manager 
* Item 
* StorageZone
* InventoryMovement 
* ReceivingLogItem (from Feature 15)

## Initial Data Model 

### InventoryMovement 

| Field Name | Data Type | Constraints | Description | 
|---|---|---|---|
| `movement_id` | Integer | PK, Not Null | Unique identifier for the specific put away task | 
| `receipt_item_id` | Integer | FK, Not Null | Links to the Feature 15 delivery (for traceability) |
| `item_id` | Integer | Foreign key referencing the storage zone it was | 
| `zone_id` | Integer | FK, Not Null | References the physical shelf/location it was placed in | 
| `worker_id` | Integer | FK, Not Null |  Foreign key referencing the worker who stored it | 
| `quantity` | Integer | Not Null, >0 | The amount of the item that was put away in the specific zone | 
| `movement_date` | DateTime | the exact date and time the items were put on the shelves | 

## Gherkin AC 

### US-16.1 
**Scenario: Worker Stores items in a specific zone**
**Given** a worker has a cart of newly receivied items on the dock 
**When** They scan the item, enter the quantity, and scan the destination `zone_id`
**Then** the system should log the new `InventoryMovement` record
**And** mark that specific quantity as successfully put away. 

### US-16.2 
**Scenario: System updates available stock**
**Given** a worker has just completed a put-away task for 50 boxes 
**When** the movement task is saved to the database 
**Then** the system should automatically update the `Item` table 
**And** increase the total `qty_on_hand` for that item by exactly 50. 

## Data Ownership & Isolation 

| Rule | Requirement | 
|---|---|
| **Read scope** | Any worker can view items pending put-away | 
| **Write scope** | Only authorized floor workers can generate an `InventoryMovement`. | 
| **Side Effects** | Writing to `InventoryMovement` MUST automatically trigger an update to the `Item` quantity | 

## API Requirements 

| Method | Endpoint | Auth | Purpose | 
|---|---|---|---|
| `GET` | `/api/receiving/pending-putaway` | Yes | Fetches all received items sitting on the dock | 
| `POST` | `/api/inventory-movements` | Yes | Sbmits the put-away action and updates the stock | 

**Movement request body:** 
```json 
{
    "receipt_item_id": 4992,
    "item_id": 44, 
    "zone_id": 12, 
    "quantity" : 50,
    "worker_id": 105
}