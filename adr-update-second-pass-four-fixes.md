# ADR-0003 update, second pass: four more fixes

Apply these after the first update (Capabilities 3 and 10 split). Nothing in the first update changes. Eight small edits: seven on the Capability 10 page and one on the Capability 3 page.

Found on a closer review of option A. Each is small on the page and closes a real hole.

| # | Problem in option A | Fix |
|---|---|---|
| 1 | The `uuid` header is random and stored on the record. If a record has to be created again in the other region, it gets a different `uuid`. If FIS includes that header in its idempotency check (OQ-A), OPF sees a new request and the payment is sent twice | Derive the `uuid` from the UETR, so it is the same wherever the record is created |
| 2 | A payment answered with 202 but not yet replicated when its region is lost exists nowhere else. Wire II cannot heal it, PayX does not replay it, and it waits for an operator | PayX replays the call once, with the same UETR, when `.WIRECLEARING` misses its SLA. With fix 1 this is always safe |
| 3 | A terminal record whose report to PayX keeps failing could be marked PARKED after maximum receives, which would hide that OPF accepted it | PARKED is for non-terminal records only. The sweeper keeps dispatching report-only attempts |
| 4 | A re-dispatch that writes the same values produces no stream event, so nothing is queued | The re-dispatch always changes the item |

### Capability 10, section 3 Assumptions

**Replace**
> After a 202, PayX waits for `.WIRECLEARING` and does not replay the call. PayX re-calls only when it got no 202 (timeout or 5xx), and alerts when `.WIRECLEARING` misses its SLA (OQ-D).

**With**
> After a 202, PayX waits for `.WIRECLEARING`. PayX re-calls when it got no 202 (timeout or 5xx). When `.WIRECLEARING` misses its SLA, PayX alerts and replays the call once with the same UETR; the replay is idempotent (OQ-D). Wire II still owns recovery in every case where the record exists.

### Capability 10, section 8.1, row `requestUuid`

**Replace the Purpose cell**
> The `uuid` header, generated once and reused on every attempt (OQ-A).

**With**
> The `uuid` header. Derived from the UETR as a name-based UUID, not random, and reused on every attempt. A record created again in the other region therefore sends exactly the same headers (OQ-A).

### Capability 10, section 8.2, rule 6 (Deterministic generation): add one sentence at the end

> The `uuid` header is derived from the UETR in the same way.

### Capability 10, section 8.3: add one bullet after "Never calls Turing or OPF"

> - **A re-dispatch always changes the item.** A write that changes nothing produces no stream event. The sweeper therefore always raises `version` and moves `checkAt` when it requests dispatch, also when `dispatchRequested` is already true.

### Capability 10, section 8.6, the bullet "The replication-lag window"

**Replace the whole bullet with**
> - **The replication-lag window.** A payment accepted in the last moment before the loss, whose record had not yet replicated, is not in us-west-2. This is the one case Wire II cannot heal by itself, because the record is not there. When `.WIRECLEARING` misses its SLA, PayX replays the call with the same UETR (OQ-D). us-west-2 has no record, creates it and submits. Deterministic generation makes the payload and the `uuid` identical, so if us-east-1 had already reached OPF, this is an idempotent repeat. When us-east-1 returns, the older copy loses to the newer one.

### Capability 10, section 12, OQ-A and OQ-D

**OQ-A: add at the end**
> Also: what OPF returns for a repeat that arrives while the first request is still in progress.

**OQ-D: replace**
> it retries only when no 202 is received, and it alerts when `.WIRECLEARING` misses its SLA.

**With**
> it retries when no 202 is received, and it alerts and replays once with the same UETR when `.WIRECLEARING` misses its SLA.

### Capability 3, section 5: add below the "Dispatch and retry" table

> **PARKED is for non-terminal records only.** If the report to PayX keeps failing for a record that is already ACCEPTED or REJECTED, the record stays terminal. Its message may reach the DLQ; the sweeper keeps dispatching report-only attempts until PayX has been told.
