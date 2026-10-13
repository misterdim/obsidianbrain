---
title: ASRS Workstation Page
application: ASRS
area: ui
project: TES
document_type: technical-wiki
source: tes-agent/src/ui/src/pages/asrs/AsrsWorkstationPage.vue
tags: [tes, technical-wiki, ui, asrs]
updated: 2026-09-22
---

# ASRS Workstation Page

The ASRS workstation page is a Vue/Quasar picking and putaway UI. It does not create its own workstation data. It receives a display state from the local agent, selects named widgets from that state, and exposes them through a Pinia store.

## One connected data path

~~~text
Browser
  AsrsWorkstationPage.vue
    -> RuntimeStore.fetch()              GET state
    -> RuntimeStore.subscribe()          websocket/STOMP updates
    -> deviceState.displayState
    -> displayState.displayData.widgets
    -> AsrsWorkstationStore
    -> PortsViewer, ToteViewer,
       AsrsProcessSection, PutwallViewer,
       page toolbar and dialogs

Server-side producer
  AsrsG2PApplicationDef
    -> DisplayWidgets
    -> DisplayRawData(name, serialized data)
    -> local-agent application state
~~~

The main source files are:

- UI page: tes-agent/src/ui/src/pages/asrs/AsrsWorkstationPage.vue
- UI store: tes-agent/src/ui/src/stores/model/widgets/AsrsWorkstationStore.ts
- Runtime transport: tes-agent/src/ui/src/stores/RuntimeStore.ts
- Backend producer: tes-activity/src/main/java/com/onomatic/tes/activity/g2p/application/AsrsG2PApplicationDef.java
- Backend payloads: tes-activity/src/main/java/com/onomatic/tes/activity/g2p/application/asrs.kt and data.kt

## What must exist before the page works

The page requires:

- an authenticated ApplicationSessionStore session;
- a valid IApplicationProps.lookupId;
- deviceId when direct application initialization is needed;
- a successful local-agent GET state response;
- an active websocket/STOMP subscription;
- a backend ASRS application that can answer g2p/asrs/refresh-state;
- a displayData.widgets array with the widget names described below.

On mount, the page restores or initializes the session, subscribes, fetches state, and invokes g2p/asrs/refresh-state. If fetching state fails, it redirects to login.

The local page calls the local agent. QueryService is used behind the agent/application layer to call the central TES API. It:

- prefixes ordinary entity/list paths with /api;
- uses the configured TES server base URL;
- authenticates with API key or OAuth;
- unwraps SuccessResponseDto.data;
- provides typed single-object and list responses.

QueryService is therefore part of the backend data path, but it does not directly shape the browser's display widgets.

## Display-state envelope

The backend display infrastructure is defined in:

- tes-coreapi/src/main/java/com/onomatic/tes/coreapi/device/application/model.kt
- tes-engine/src/main/java/com/onomatic/tes/engine/core/model/application/ApplicationDef.java

The relevant envelope is:

~~~text
DisplayWidgets
  widgets: List<IDisplayWidget>

DisplayRawData
  name: String
  data: String?
~~~

ApplicationDef.getData(name, data) serializes the supplied backend object to JSON and returns DisplayRawData. The browser receives that JSON as widget.data and AsrsWorkstationStore parses it.

There are two payload mechanisms:

1. RAW_DATA widgets use widget.data containing serialized JSON.
2. DisplayFields widgets use widget.elements containing named fields.

## Widget contract map

Each section below connects one backend payload to the exact widget name, frontend getter, and UI consumer. The shape is shown only once.

### ports — PortData

Backend class:

~~~text
com.onomatic.tes.activity.g2p.application.PortData
source: tes-activity/.../application/asrs.kt
~~~

Serialized shape:

~~~ts
{
  name: string,
  tote: string,
  selected: boolean,
  taskGroup: string,
  state: CarouselState,
  taskCount: number
}[]
~~~

Producer:

~~~text
AsrsG2PApplicationDef.getPorts(session)
  -> session.getPorts()
  -> PortData(...)
  -> getData("ports", getPorts(session))
~~~

Frontend:

- widget name: ports
- getter: AsrsWorkstationStore.ports
- consumer: PortsViewer -> EquipmentViewerSquare
- displayed fields: name, state, taskGroup, taskCount

Notes:

- The backend also sends tote and selected; the current viewer does not use them.
- CarouselState is serialized and the viewer recognizes MOVING.
- The current producer sets selected to false.

### tote — ToteData

Backend classes:

~~~text
com.onomatic.tes.activity.g2p.application.ToteData
com.onomatic.tes.activity.g2p.application.ToteBinData
com.onomatic.tes.activity.g2p.application.SelectedToteBinData
source: tes-activity/.../application/asrs.kt
~~~

Serialized shape:

~~~ts
{
  referenceNo: string,
  name: string,
  layout: Layout | null,
  bins: {
    referenceNo: string,
    name: string,
    layout: Layout | null
  }[],
  selectedBin: {
    name: string,
    quantity: ItemQuantity
  } | null
}
~~~

Producer:

~~~text
AsrsG2PApplicationDef.getTote(session)
  -> session.getTote()
  -> LocationRepository lookup
  -> PositionRepository.findAllByLocation(...)
  -> ToteBinData(...)
  -> SelectedToteBinData(current task position, quantity)
  -> ToteData(...)
  -> getData("tote", getTote(session))
~~~

Frontend:

- widget name: tote
- getter: AsrsWorkstationStore.tote
- consumers: ToteViewer and the store shelf getter
- displayed data: tote name, bins, selected bin, selected quantity

Notes:

- The frontend IToteState omits backend referenceNo and layout.
- ToteViewer highlights the bin whose referenceNo matches selectedBin.name.
- An active task needs a non-null selectedBin because the viewer accesses selectedBin.name.
- The UI currently displays the selected quantity with a hardcoded EA suffix.
- ToteViewer is visible only when status is waiting.

### workstation — WorkstationData

Backend class:

~~~text
com.onomatic.tes.activity.g2p.application.WorkstationData
source: tes-activity/.../application/data.kt
~~~

Serialized shape:

~~~ts
{
  activity: G2PActivityType,
  taskAssignmentMode: TaskAssignmentMode,
  hasPausedTasks: boolean,
  canPauseTasks: boolean
}
~~~

Producer:

~~~text
AsrsG2PApplicationDef.getWorkstation(session)
  -> WorkstationData(
       session.currentActivity,
       session.taskAssignmentMode,
       hasPausedTasks,
       canPauseTasks
     )
  -> getData("workstation", getWorkstation(session))
~~~

Frontend interpretation:

- widget name: workstation
- getter activity -> mode
- getter hasPausedTasks -> paused
- getter taskAssignmentMode != NONE -> assignable/suspendable
- getter canPauseTasks -> canPauseTasks
- consumers: page toolbar, assignment buttons, mode controls

Expected activity values:

~~~text
pick/order       -> picking
pick/adhoc       -> hot-picking
putaway/container -> putaway
putaway/adhoc    -> hot-putaway
~~~

The frontend compares serialized enum values, so backend enum serialization must preserve these activity names and the NONE assignment value.

### staging — StagingStationData

Backend class:

~~~text
com.onomatic.tes.activity.g2p.application.StagingStationData
source: tes-activity/.../application/data.kt
~~~

Serialized shape:

~~~ts
{
  id: string,
  name: string,
  taskCount: number,
  taskId: ActivityTaskItemId | null,
  orderNo: string,
  container: string,
  instructions: string,
  quantity: number,
  unit: string
}[]
~~~

Producer:

~~~text
AsrsG2PApplicationDef.getStagingStations(session)
  -> session.getStagingAreas()
  -> pick/putaway task lookup
  -> StagingStationData(...)
  -> getData("staging", getStagingStations(session))
~~~

Frontend:

- widget name: staging
- getter: AsrsWorkstationStore.stations
- consumers: PutwallViewer -> PutwallCard
- secondary consumers: AssignOrdersDialog, AssignContainersDialog, ShortPickDialog

Frontend uses:

- name for the station label;
- taskCount to identify occupied stations;
- orderNo/container for assignment display;
- instructions for the station instruction;
- quantity/unit for the quantity tag;
- taskId for short-pick task payload.

Notes:

- The backend id is currently ignored by IStationState.
- The backend taskId is ActivityTaskItemId; the frontend treats it as any.
- quantity originates as BigDecimal and is consumed as a JavaScript number/string.
- extractInstructions currently returns the literal string null.
- For putaway, the producer fills both orderNo and container with the task reference number because the UI primarily reads orderNo.

### carrier — CarrierData (parsed but not currently emitted here)

Backend classes:

~~~text
com.onomatic.tes.activity.g2p.application.CarrierData
com.onomatic.tes.activity.g2p.application.CarrierShelfData
com.onomatic.tes.activity.g2p.application.CarrierBinData
com.onomatic.tes.activity.g2p.application.SelectedBinData
source: tes-activity/.../application/data.kt
~~~

Serialized shape:

~~~ts
{
  referenceNo: string,
  name: string,
  layout: Layout | null,
  shelves: {
    referenceNo: string,
    name: string,
    layout: Layout | null,
    bins: {
      referenceNo: string,
      name: string,
      layout: Layout | null
    }[]
  }[],
  selectedBin: {
    shelf: string,
    name: string,
    quantity: ItemQuantityInfo
  } | null
}
~~~

Frontend:

- getter: AsrsWorkstationStore.carrier and shelf
- intended consumer: carrier/tote equipment layout logic
- current active consumer: none; ToteViewer defines a rowTemplate from carrier but does not use it

Important distinction: CarrierData exists as a backend model, but the observed AsrsG2PApplicationDef.displayState() adds tote, workstation, staging, and ports only. It does not add a carrier widget in that method.

## DisplayFields contract for the active task

The following are not RAW_DATA payloads. They are DisplayFields widgets created by:

~~~text
AsrsG2PApplicationDef.getCurrentTaskFields(HandleItemPositionDef)
~~~

That method creates the item, task, and position widgets when there is a current task.

### item — DisplayFields

Backend fields created:

~~~text
itemName       -> STRING -> item master name
vendorNo       -> STRING -> item vendor property
manufacturerNo -> STRING -> item manufacturer property
imageURL       -> IMAGE  -> item image URL property
~~~

Frontend:

- widget name: item
- getter: AsrsWorkstationStore.item
- consumer: AsrsProcessSection
- imageURL becomes the product image;
- all other elements become DisplayFieldSet entries.

The getter requires both item and task widgets.

### task — DisplayFields

Backend fields created:

~~~text
itemNo   -> STRING   -> current task item number
quantity -> QUANTITY -> current task ItemQuantity
~~~

Frontend:

- widget name: task
- getter: AsrsWorkstationStore.item reads itemNo;
- getter: AsrsWorkstationStore.quantity reads quantity.quantity;
- consumers: page summary, AsrsProcessSection, ShortPickDialog.

The frontend currently hardcodes the displayed unit to EA.

### position — DisplayFields

Backend fields created:

~~~text
position  -> STRING -> position reference number
carousel  -> STRING -> parent carousel name
carrier   -> STRING -> carrier/location parent name
shelf     -> STRING -> shelf/location name
bin       -> STRING -> first two characters of position name
binDepth  -> STRING -> last two characters of position name
~~~

Frontend:

- widget name: position
- getter: AsrsWorkstationStore.position
- consumer: status logic and the currently commented-out position display.

The presence of position is significant even when its fields are not rendered: no position widget means the store reports standing; a position widget plus an instruction normally reports waiting.

## Status is derived from data presence

~~~text
no displayData                         -> initializing
store.sending === true                 -> sending
no instruction                         -> refreshing
instruction type waiting-completion    -> holding
instruction type waiting-carousel      -> moving
no position widget                     -> standing
otherwise                              -> waiting
~~~

These states control skeletons, task details, action buttons, assignment controls, and tote visibility.

The page displays:

- displayState.instruction.message through HandheldInstruction;
- displayState.message.message through the process confirmation toast.

## Actions required by the page

The page posts to /action:

- g2p/asrs/refresh-state
- g2p/pause
- g2p/resume
- g2p/select-activity with pick/order
- g2p/select-activity with putaway/container

The process section additionally uses:

- g2p/confirm-task
- g2p/skip-task
- the short-pick flow

Assignment dialogs use:

- GET /orders
- GET /containers
- g2p/assign-orders
- g2p/assign-containers

The ASRS local-agent controller exposes the list endpoints below, with the configurable application base path:

~~~text
/api/applications/g2p/asrs/orders
/api/applications/g2p/asrs/containers
~~~

## Component and store map

~~~text
AsrsWorkstationPage.vue
  -> useAsrsWorkstationStore()
  -> RuntimeStore.deviceState

  -> PortsViewer
       -> ports

  -> ToteViewer
       -> tote

  -> AsrsProcessSection
       -> instruction, message, item, task, position, workstation

  -> PutwallViewer -> PutwallCard
       -> staging

  -> AssignOrdersDialog / AssignContainersDialog
       -> staging, workstation, orders/containers endpoints
~~~

## Failure and compatibility notes

- JSON-backed widget data is parsed without local validation.
- Missing displayData produces initializing.
- Missing instruction produces refreshing.
- Missing position produces standing.
- Missing either item or task suppresses active item controls.
- A null tote selectedBin is unsafe while ToteViewer is visible.
- Layout, ItemQuantity, enum, and identifier serialization must remain compatible with the frontend assumptions.
- The observed producer methods in AsrsG2PApplicationDef are marked Deprecated. If a newer producer replaces them, re-check widget names and serialized shapes.

## Source files

- tes-agent/src/ui/src/pages/asrs/AsrsWorkstationPage.vue
- tes-agent/src/ui/src/stores/model/widgets/AsrsWorkstationStore.ts
- tes-agent/src/ui/src/stores/RuntimeStore.ts
- tes-agent/src/ui/src/stores/model/Workstation.ts
- tes-agent/src/ui/src/components/widgets/equipmentViewer/asrs/PortsViewer.vue
- tes-agent/src/ui/src/components/widgets/equipmentViewer/asrs/ToteViewer.vue
- tes-agent/src/ui/src/components/widgets/equipmentViewer/putwall/PutwallViewer.vue
- tes-agent/src/ui/src/pages/asrs/AsrsProcessSection.vue
- tes-agent/src/ui/src/pages/asrs/AssignOrdersDialog.vue
- tes-agent/src/ui/src/pages/asrs/AssignContainersDialog.vue
- tes-agent/src/ui/src/pages/asrs/ShortPickDialog.vue
- tes-agent/src/main/java/com/onomatic/tes/agent/application/g2p/asrs/AsrsG2PRestController.java
- tes-agent/src/main/java/com/onomatic/tes/reagent/bootstrap/QueryService.java
- tes-activity/src/main/java/com/onomatic/tes/activity/g2p/application/AsrsG2PApplicationDef.java
- tes-activity/src/main/java/com/onomatic/tes/activity/g2p/application/asrs.kt
- tes-activity/src/main/java/com/onomatic/tes/activity/g2p/application/data.kt
- tes-coreapi/src/main/java/com/onomatic/tes/coreapi/device/application/model.kt
- tes-engine/src/main/java/com/onomatic/tes/engine/core/model/application/ApplicationDef.java
