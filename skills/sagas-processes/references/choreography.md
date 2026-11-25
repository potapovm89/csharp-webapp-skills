# Choreography (Event-Driven Chains)

Choreography coordinates a process by events: each participant reacts to the previous event and emits the next one. There is no central state — every service owns its part. Use it only for **short chains (2–3 participants)** where each reaction is independently useful and nobody needs a single view of the whole flow.

## Rules
- Every event is published through the transactional outbox and consumed with inbox idempotency (`message-brokers`); a chain is only as durable as its weakest consumer.
- Correlation is explicit: `correlation-id` + a business process id travel in message headers (`message-brokers`, kafka.md tracing) — without them the flow is undebuggable.
- A confirmed step failure publishes `<step>-failed` through the outbox. Earlier participants compensate only their completed parts. An unknown external outcome stays in the owning participant's reconciliation or manual state and emits no failure event that would trigger compensation. Apply [the same outcome rules](../SKILL.md#external-effect-with-an-unknown-outcome) before allowing any retry.
- A participant must tolerate the "failed" event arriving **before** it finished its own step (races are normal): the handler resolves the conflict against current state, not against an assumed order.
- Non-compensable effects (external payment, physical shipment) terminate the chain into an explicit state owned by a service with a runbook — choreography does not make them undoable.
- Event contracts are versioned like any broker message (`message-brokers`); a chain spanning N services is N× more sensitive to contract drift — additive changes only, `buf`-style discipline for payloads.
- The flow is documented end-to-end: `docs/contracts/<broker>/` lists every topic in the chain, and the C4 model shows the sequence (`project-documentation`, arch-dsl.md). An undocumented choreography is invisible by construction — that is its main risk.

## When to Promote to a Process Manager
Move to the orchestration pattern (`process-manager.md`) when any of these becomes true:
- someone asks "where is my order stuck right now?" and the answer requires opening N dashboards;
- a new step needs conditional branching or a timeout of its own;
- compensation order started to matter across more than two participants;
- the chain grew past ~3 participants or past one MR of reading.

Promotion is mechanical: keep the existing events as step triggers, add the instance table, replay nothing — new instances use the process manager, in-flight ones finish under choreography. Record the decision in an ADR.

## Anti-Patterns
- A "chain" of 5+ topics reacting to each other — this is a process manager that lost its state table.
- Compensation logic duplicated in every consumer with different rules — compensations are declared, reviewed and tested once per participant.
- Publishing a `<step>-failed` event outside a transaction (the failure must be durable like any state change).
- Cycles: A→B→A. A cycle is a loop, not a process — break it with an explicit state.
