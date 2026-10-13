---
title: Yourbar Shipping Application
project: TES
document_type: technical-wiki
area: applications
tags: [tes, technical-wiki, applications, yourbar]
updated: 2026-09-22
---

# Yourbar Shipping Application

Parent: [[Yourbar Chateauguay Workflow Applications]]  
Related: [[TES Runtime Flows]], [[TES Workflow Concepts]], [[TES REST APIs]]

## Configuration usage

Shipping is referenced by:

`resources/config/customers/yourbar/chateauguay/workflow/yourbar-picking.csv`

The `$SITE/picking/pick` activity uses forklift picking, while `$SITE/picking/ship` uses:

- resource class: `WORKER_POOL`;
- type: `ship/order/workers`;
- resource: `$SITE/workers`;
- location: `$SITE/DK-DK`;
- application: `$SITE/shipping-app`.

The shipping activity starts at `$SITE/DK-DK` and ends at `$SITE/dk-shipped`. Its priority is `P0R1` FIFO.

## Event flow

```mermaid
sequenceDiagram
    participant Worker as Shipping operator
    participant App as Shipping app
    participant Engine as Workflow engine
    participant Task as Ship order activity
    participant Inventory as Order/inventory

    Engine-->>App: READY ship/order/ship-order request
    Worker->>App: scan order number
    App->>App: Find READY or IN_PROGRESS shipping task
    Worker->>App: SHIP action
    App->>Engine: RequestTaskAssignmentCommand
    Engine-->>App: ActivityTaskCreatedEvent
    App->>Task: start task
    Task->>Inventory: Read order items by position/container
    Task->>Engine: Create CONTAINER_SHIP worker task per source container
    Engine-->>App: resource task events
    App->>Engine: complete container shipping execution
    Engine-->>Task: ActivityTaskCompletedEvent
    Task-->>Inventory: shipping/order completion
    Engine-->>App: order shipped confirmation
```

## Implementation

- Application: `tes-activity/src/main/java/com/onomatic/tes/activity/shipping/ShippingApplicationDef.java`
- Session: `tes-activity/src/main/java/com/onomatic/tes/activity/shipping/ShippingSessionDef.java`
- State machine: `tes-activity/src/main/java/com/onomatic/tes/activity/shipping/state/ShippingApplicationState.java`
- Agent builder/controller: `tes-agent/src/main/java/com/onomatic/tes/agent/application/shipping/ShippingBuilder.java`, `ShippingController.java`
- Activity: `tes-activity/src/main/java/com/onomatic/tes/activity/shipping/ShippingActivityDef.java`
- Task: `tes-activity/src/main/java/com/onomatic/tes/activity/shipping/task/ShippingOrderDef.java`

The application accepts worker sessions and only count/ship activity events for the Ship activity parent. It shows READY/IN_PROGRESS shipping tasks by order reference.

## Task creation

`ShippingActivityDef` builds a shipping activity request from work-order lines and the configured start/end positions. It activates the request.

After `SHIP`:

1. The application assigns the task request to the current worker.
2. On activity task creation, it starts the task.
3. It groups order items by source container position.
4. It creates a `WorkerTaskExecutionType.CONTAINER_SHIP` task for each container, moving it to the shipping end position.
5. `ShippingOrderDef` completes the activity task once no resource task remains.
6. The application receives `ActivityTaskCompletedEvent`, displays ORDER_SHIPPED, and returns to order selection.

If no order lines are found, the application displays `NO_ORDER_LINES`.
