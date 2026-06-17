# PXS Workflows — Hands-on Lab

This lab uses the PXS Workflows UI as a front end [https://aka.ms/pxs-workflows-demo](https://aka.ms/pxs-workflows-demo). **You don't change the UI.** All the work happens in **Azure** — Logic App **workflows** and Logic App **automations** backed by **Azure Functions**.

You are given use cases (below) and implement them on Azure. The UI already
sends the right requests; your job is to build the destinations that receive
them.

## What the UI already gives you

The frontend posts JSON to two **configurable Logic App URLs** that you set on
the **Settings** page:

- **Employees endpoint** — fired when a new employee is added on the
  **Employees** page.
- **Automations endpoint** — fired from the per-row **gear** button on the
  **Employees** page. The payload includes an extra `automation` field that is
  either `generate-employee-id` or `onboard-machine`.

Useful facts:

- The `role` field is always `Technician` or `Office`.
- Requests are sent **straight from the browser**, so each Logic App / Function
  must allow **CORS** from the web app's origin, or calls fail.
- A response is treated as success **only on a `2xx`** status code. Non-`2xx`,
  network, or CORS failures surface as an error in the UI.
- The UI automatically appends `api-version=2016-10-01` to the endpoint URL if
  it's missing, so the standard Logic App manual-trigger URL works as-is.

---

## Story 1 — New-hire onboarding workflow

> **As** an HR/IT operations user, **when** I add a new employee in the UI,
> **I want** an onboarding workflow to run automatically, **so that** the new
> hire is recorded and the right people are notified before their start date.

- **Trigger (already wired):** HTTP POST to the **Employees** Logic App endpoint
  with the full employee payload (`name`, `email`, `department`, `role`,
  `startDate`, `manager`, …).
- **Acceptance criteria:**
  - The workflow accepts the POST and returns a `2xx` so the UI shows success.
  - The employee is persisted to a store of your choice (Table Storage,
    SharePoint list, Dataverse, email — open).
  - The workflow **branches on `role`** (`Technician` vs `Office`) and follows a
    different onboarding path for each.
  - A notification is sent (Teams / email) containing at least name, department,
    role and start date .
- **Azure building blocks:** a single Logic App (Consumption or Standard)
  workflow.
- **Stretch:** validate required fields and return a non-`2xx` when something is
  missing, then confirm the UI surfaces the error.

---

## Story 2 — "Generate employee ID" automation

> **As** an IT administrator, **when** I run the _Generate employee ID_
> automation for an employee, **I want** a unique, policy-compliant ID
> generated, **so that** downstream systems share one consistent identifier.

- **Trigger (already wired):** POST to the **Automations** endpoint where
  `automation == "generate-employee-id"`. For this story the **Automations**
  endpoint URL points **directly at the Azure Function** — there is **no Logic
  App** in between.
- **Acceptance criteria:**
  - The **Azure Function** is the direct target of the **Automations** endpoint
    (no Logic App router): the function itself inspects the `automation` field
    and handles only the `generate-employee-id` case.
  - It generates the ID (e.g. derived from department + initials + sequence —
    rules are open).
  - Because the browser calls the function directly, the Function App **allows
    CORS** from the web app's origin and returns `2xx` with the generated ID.
  - Same employee in → same ID out (idempotency approach is open).
- **Azure building blocks:** a single **HTTP-triggered Azure Function** called
  **directly** by the Automations endpoint (no Logic App). 

---

## Story 3 — "Onboard machine" automation

> **As** an IT provisioning agent, **when** I run the _Onboard machine_
> automation for an employee, **I want** the correct equipment ordered for their
> role, **so that** their hardware is ready on day one.

- **Trigger (already wired):** POST to the **Automations** endpoint where
  `automation == "onboard-machine"`.
- **Pipeline shape:** this story is intentionally a **multi-hop chain** —
  **Logic App workflow (router/guard) → Azure Function (worker) → durable sink
  (Storage Account *or* Event Hub)**. The browser only ever talks to the Logic
  App; the Function is called server-to-server from the workflow.
- **Acceptance criteria:**
  - **Logic App as the gatekeeper:** the workflow inspects the incoming payload
    and **only continues when `automation == "onboard-machine"`**; any other
    `automation` value is short-circuited (e.g. ignored or rejected) so this
    branch never runs for the wrong case.
  - **Forward to the Function:** the workflow calls an **HTTP-triggered Azure
    Function** (passing the employee + `role`), and the function "places an
    order" and returns an order/tracking ID (mirror the `orderId` + `traceId` +
    `status` response from `/security-gear/order`).
  - **Persist to a durable sink (the new, harder part):** the Function must
    **emit the order to one of**:
    - a **Storage Account** — create a **new blob** per order (e.g.
      `orders/{role}/{orderId}.json`) containing the order payload, **or**
    - an **Event Hub** — publish a **new event** per order carrying the same
      order payload.
  - Equipment differs by `role`: e.g. a `Technician` also gets field/security
    gear, while `Office` gets a standard laptop bundle (catalog is open) — and
    this is reflected in the persisted file/event.
  - The Logic App returns `2xx` to the UI with the order confirmation **only
    after** the Function reports the file/event was successfully written.
- **Azure building blocks:** Logic App (router + guard) + HTTP-triggered Azure
  Function + a **Storage Account (Blob)** *or* **Event Hub** as the sink. 
- **Stretch:** make the chain resilient — add a **retry policy** on the
  Logic App → Function call, **simulate a failure** in the sink write and prove
  the UI surfaces a non-`2xx`, and have a **downstream consumer** (a second
  Function or Logic App triggered by the new blob / Event Hub event) pick the
  order up and send a confirmation email containing the order ID.