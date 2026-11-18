# ADR-0003 update: asynchronous Wire II, with Capabilities 3 and 10 split again

The Capability 10 page was revised on 2026-11-17 to an asynchronous design: two microservices in ASVWIREII, the **Wire II API** and the **Wire II Submitter**, with managed dispatch (DynamoDB stream, EventBridge Pipe, SQS) between them. That design is the baseline of this update. Two things are done here:

1. **Split.** Capabilities 3 and 10 were merged when SQS was removed. SQS is back, and the design now has two separately deployed microservices, so the capabilities are split again along that line: **Capability 10 is the Wire II API**, **Capability 3 is dispatch and the Wire II Submitter**.
2. **Review items.** The design review of 2026-11-17 asked for answers on a prolonged OPF outage, the dead-letter queue, partial regional failures and a failure analysis. Those are added as new sections 8.9 to 8.12 of Capability 10 and in the Capability 3 page.

Nothing in Option A is reversed. The additions keep its rules: one table owner, a Submitter without database access, the stored payload re-sent byte for byte.

## Which page becomes what

| Page | Today | After this update |
|---|---|---|
| Capability 10 (Wire II — pain.001 generation) - ADR-0003 (page 3445334201, under "Old") | Proposed, updated 2026-11-17. Covers generation, detokenization and submission | **Capability 10: the Wire II API.** Move from "Old" to "3. OPF Integration - ADR-0003" |
| Capability 3 (Submit Request to OPF API) - ADR-0003 (page 3445334194, under "Old") | The synchronous inline version, retired by the merge | **Capability 3: dispatch and the Wire II Submitter.** Whole page replaced. Move from "Old" to "3. OPF Integration - ADR-0003" |
| Capability 10 (Wire II — pain.001 Generation, Detokenization & OPF Submission) - ADR-0003 (page 3475287056) | DRAFT, the merged synchronous page | Superseded. New banner, then move to "Old" |
| Capability 3 (Submit Request to OPF API) - ADR-0003 (Copy) (page 3472160403) | Stale copy | Delete or archive, as already recommended |
| Capability 2 (OPF API integration) - ADR-0003 (page 3424399244) | Names one Wire II service as the caller | Three small edits |
| ADR-0003 main page, capability summary table | One row for the merged capability | Two rows again |

## What changes, at a glance

| Page | Section | Change |
|---|---|---|
| Capability 10 | Status panel | Scope narrowed to the Wire II API; points to Capability 3 |
| Capability 10 | 5. Considered options, row F | Records that three parts of option F are adopted into option A |
| Capability 10 | 6. Decision | First sentence; components table gains a Capability column; submission path, 6.3 and 6.4 move to Capability 3 |
| Capability 10 | 6.2 Internal API | Lease guard also covers the circuit breaker and the submit deadline |
| Capability 10 | 8.1 Transaction record | Three attributes added; control records added |
| Capability 10 | 8.3 Recovery sweeper | Three rows added: breaker open, probe and drain, deadline |
| Capability 10 | 8.4, 8.5 | Move to Capability 3 |
| Capability 10 | 8.6 Regional failover | One bullet added: strong consistency closes the replication-lag window |
| Capability 10 | 8.8 Observability | Two alarms added |
| Capability 10 | New 8.9 to 8.12 | Prolonged outage and circuit breaker; submit deadline and unknown outcome; partial regional failure; failure analysis |
| Capability 10 | 9. Risks, last row | Mitigation replaced: no DLQ redrive after an outage |
| Capability 10 | 12. Open questions | Four added |
| Capability 3 | Whole page | Replaced |
| Merged page | Status banner | Superseded notice |
| Capability 2 | Decision first paragraph; Implications first bullet; Question 4 | The caller is the Wire II Submitter |
| ADR-0003 main page | Capability summary table | Rows for Capabilities 3 and 10 |

---

## Capability 10 (Wire II — pain.001 generation) - ADR-0003

### Status panel, second paragraph

**Replace**
> Capability 10 covers pain.001 generation, BAN detokenization and OPF submission. It is delivered by two microservices: the **Wire II API** accepts the payment idempotently, stores it and returns 202; the **Wire II Submitter** takes work from SQS and submits to OPF, a third-party API, in the background.

**With**
> Capability 10 covers the **Wire II API**: it accepts the payment idempotently, builds and stores the pain.001, returns 202, owns every state transition of the transaction record and runs the recovery sweeper. Dispatch, BAN detokenization and OPF submission are [Capability 3 (Submit Request to OPF API)]: the **Wire II Submitter** takes work from SQS and submits to OPF, a third-party API, in the background. The two capabilities are one design and are read together.

The rest of the panel is unchanged.

### 5. Considered options, row F

**Replace the Outcome cell** `Not chosen` **with**
> Not chosen as a whole. Three of its parts are adopted into option A, because option A alone leaves a prolonged vendor outage to operators: the circuit breaker (8.9), the submit deadline with its two reasons (8.10) and the failure analysis (8.12). Its separate notify path, its extra queues and its Lambdas are not adopted.

### 6. Decision

**Replace**
> Option A. Capability 10 is delivered by two microservices in ASVWIREII, with managed dispatch between them.

**With**
> Option A. Capabilities 10 and 3 are delivered by two microservices in ASVWIREII, with managed dispatch between them: the Wire II API (this page) and the Wire II Submitter ([Capability 3 (Submit Request to OPF API)]).

**Components table: add a fourth column "Capability"**

| Component | Capability |
|---|---|
| Wire II API (Fargate) | 10 |
| Dispatch (managed) | 3 |
| Wire II Submitter (Fargate) | 3 |

**Delete four pieces from this page.** You do not need to copy them anywhere: the Capability 3 page further down in this file already contains all four, ready to paste.

| # | What to delete on the Capability 10 page | Where it starts | Where it ends |
|---|---|---|---|
| 1 | The block headed "Submission path — Wire II Submitter" | The bold heading "Submission path — Wire II Submitter" | The end of step 6 ("...the record is marked PARKED, with an alert.") |
| 2 | One sentence at the end of section 6 | "The Submitter can run as an ECS service..." | "...the contract is the same." |
| 3 | Section 6.3 | The heading "6.3 OPF response mapping" | The end of the yellow warning panel about 422 |
| 4 | Section 6.4 | The heading "6.4 Turing detokenize response handling" | The last row of its table ("500") |

**Then type this one line where piece 1 was**, so a reader of Capability 10 knows where the text went:

> Submission, the OPF response mapping and the Turing response handling are in [Capability 3 (Submit Request to OPF API)].

Sections 6.1 and 6.2 keep their numbers. Section 6.3 and 6.4 no longer exist on this page, and section 7 follows 6.2.

The blocks "Request path", "Dispatch" and "Recovery path" stay on this page.

### 6.2 Internal API, first row

**Replace the Guard cell**
> Conditional: owner = this region, no live lease, not reported. Terminal-but-unreported returns the outcome so the Submitter can report it.

**With**
> Conditional: owner = this region, no live lease, not reported, submit deadline not passed, and the circuit breaker for OPF and Turing closed (or this UETR is the probe, section 8.9). Terminal-but-unreported returns the outcome so the Submitter can report it. A lease refused because the breaker is open tells the Submitter to delete the message; the record waits and the sweeper dispatches it again later.

### 8.1 Transaction record: add three rows

| Attribute | Type | Purpose |
|---|---|---|
| `submitDeadline` | string (ISO-8601) | Set at acceptance. After it, no lease is granted and the payment is never sent automatically (8.10). |
| `unansweredAttempts` | number | How many attempts may have reached OPF without a clear answer. Raised when a lease is granted; taken back when the outcome is clear, or when the Submitter reports that nothing was sent. |
| `parkedReason` | string | Set with PARKED: `MAX_RECEIVES`, `NOT_SENT` or `OUTCOME_UNKNOWN` (8.10). Tells the operator whether a replay is safe. |

**Add below the table**
> **Control records.** The Wire II API also keeps a few small control items, in the same table under reserved keys: one circuit-breaker item per dependency and region (`state` CLOSED / OPEN / PROBING, the UETRs that failed in a row, `nextProbeAt`, `probeUetr`, `probeInterval`), and the sweeper lock. Only the API reads or writes them. The Submitter never sees them; it only sees a granted or a refused lease.

### 8.3 Recovery sweeper: add three rows to the table

| Found | Sweeper action |
|---|---|
| The circuit breaker of this region is open | Hold: do not request dispatch for pending records. When a probe is due, name one waiting record as the probe and request dispatch for it only. |
| The circuit breaker has just closed | Drain: request dispatch for the waiting records, oldest first, a set number per sweep. |
| A pending record whose `submitDeadline` has passed, with no live lease | Mark PARKED with `parkedReason` = `NOT_SENT` or `OUTCOME_UNKNOWN`; alert (8.10). |

### 8.4 Dispatch and retry, 8.5 Wire II Submitter runtime

**Delete both sections from this page.** The Capability 3 page below already contains them (its sections 5 and 6). Leave one line here:
> Dispatch, retry and the Submitter runtime are in [Capability 3 (Submit Request to OPF API)]. The recovery sweep row of that table is described in 8.3 above.

### 8.6 Regional failover and auto-heal: add one bullet after "The replication-lag window"

> - **Closing the window (option).** With DynamoDB multi-Region strong consistency (replicas in us-east-1 and us-west-2 and a witness in a third region), the 202 is given only after a second region holds the record. The replication-lag window disappears, and a lease becomes a real cross-region lock. It needs platform confirmation (OQ-I); the design above works without it.

### 8.8 Observability: add two bullets

> - Alarm when a circuit breaker opens, by dependency and region; and a separate alarm when it is open in the active region while the other region's is closed.
> - Alarm on PARKED records by `parkedReason`.

### New section 8.9 Prolonged OPF or Turing outage: the circuit breaker

> OPF is a vendor system; an outage of hours must cost delay and nothing else. Without a breaker, every in-flight payment retries on its own until its message reaches the DLQ, and operators redrive hundreds of messages afterwards.
>
> **Where it lives.** In the Wire II API, as a control record per dependency (OPF, Turing) and region. The gate is the lease: a Submitter cannot call OPF without one.
>
> | Step | What happens |
> |---|---|
> | Opens | The outcome call reports each FAILED_RETRYABLE with its cause. When a set number of *different* payments have failed in a row (proposed: 5), the API sets the breaker to OPEN and pages. One payment that keeps failing does not open it. |
> | While open | Lease requests are refused. The Submitter deletes the message. The record stays DUE or FAILED_RETRYABLE; no receive is burned and no attempt is counted. The API keeps answering 202 to PayX. Queues stay empty; the backlog is in the table. |
> | Probe | FIS has no health endpoint, so the test is one real payment. When a probe is due, the sweeper names one waiting record and requests dispatch; the API grants the lease for that UETR only. A clear answer from OPF, accepted or rejected, closes the breaker. A failure reopens it and doubles the wait: 1, 2, 4, 8 minutes, then every 15. The next probe is a different payment. |
> | Drain | After closing, the sweeper requests dispatch for the waiting records oldest first, a set number per sweep (proposed: 60). Consumer concurrency still caps the call rate. |
> | By hand | Operations can open or close a breaker through the API. Restricted and audited. |
>
> **Example.** OPF is down from 13:00 to 17:00. The breaker opens at about 13:02. Probes go out at 13:03, 13:05, 13:09, 13:17, then every 15 minutes. The first probe after 17:00 succeeds, by 17:15 at the latest; the backlog then drains at 60 a minute. No message reaches the DLQ and no operator action is needed. Payments whose submit deadline passes during the outage are parked (8.10).
>
> At low volume the breaker may never open, since it takes several different payments. Each payment then follows the normal redelivery backoff, which has the same effect.

### New section 8.10 Submit deadline and unknown outcome

> `submitDeadline` is set at acceptance: the earliest of how long FIS remembers an idempotency key (OQ-A), the cut-off for the payment's execution date, and how long product allows a payment to wait. No lease is granted after it, so a retry can never reach OPF after the key has been forgotten.
>
> When the deadline passes, the sweeper marks the record PARKED, with one of two reasons:
>
> - **`NOT_SENT`**: no attempt ever reached OPF, for example because the breaker was open the whole time. An operator replay is always safe.
> - **`OUTCOME_UNKNOWN`**: at least one attempt ended without a clear answer (`unansweredAttempts` above zero). OPF may hold the payment. It is not replayed before someone has checked OPF.
>
> PARKED reached through maximum receives keeps the reason `MAX_RECEIVES`.

### New section 8.11 Partial regional failure

> A region can be healthy and still unable to send, for example when a security group or proxy change cuts the active region's Submitter off from the partner gateway. The API accepts payments and grants leases; every OPF call times out.
>
> - The breaker of that region opens (8.9). Payments wait; none is parked or sent to the DLQ.
> - The alarm "breaker open in the active region, closed in the other" names the situation.
> - "OPF breaker open for longer than a set time" is an input to the routing control (OQ-E). When the active region is switched, the other region's sweeper adopts the waiting records under the existing adoption rule and its Submitter sends them.
> - Attempts that timed out count as unanswered although nothing reached OPF. The resend with the same key and bytes resolves them.

### New section 8.12 Failure analysis

> Severity is the effect on a payment if the response did not exist: **High** = money could move twice or a payment could be lost; **Medium** = delay; **Low** = none.
>
> | # | Component | Failure | Detected by | Response | Severity |
> |---|---|---|---|---|---|
> | 1 | Wire II API | Down in the active region | Routing health check | Routing control switches region; the other sweeper adopts | Medium |
> | 2 | Wire II API | Stops after the write, before the 202 | PayX timeout | PayX re-calls; replay response | Low |
> | 3 | Transaction table | Cannot be written | 5xx rate; health check | API answers 5xx; PayX re-calls; nothing is stored | Medium |
> | 4 | Pipe | Stopped, or not delivering to SQS | Pipe failures; records the sweeper had to re-dispatch | Pipe retries and its DLQ; the sweeper re-dispatches after the stall threshold | Medium |
> | 5 | Submit queue | Message lost or queue purged | Stalled-record alarm | Sweeper re-dispatches | Low |
> | 6 | Submitter | Stops mid-attempt, after OPF accepted | Lease expiry | Redelivery or sweeper; same payload, same key | High |
> | 7 | Submitter | Cannot run: defect, bad deploy, missing permission | Message in the DLQ; stalled records | Record PARKED with `MAX_RECEIVES`; operator replay after the fix | Medium |
> | 8 | Submitter | Cannot reach OPF from the active region | Breaker open here, closed in the other region | Breaker; switch the active region (8.11) | Medium |
> | 9 | Turing | Down | Breaker for Turing | Payments wait; probe; drain | Medium |
> | 10 | OPF | 5xx or timeout on some calls | Error rate | Redelivery with backoff; same key | High |
> | 11 | OPF | Down for hours | Breaker open | Wait in the table; probe; drain (8.9) | Medium |
> | 12 | OPF | Rejects our credentials or set-up (401 / 403) | Same answer on several payments | Breaker opens and pages; no payment is failed | Medium |
> | 13 | OPF | Accepts a repeat as a new payment | Reconciliation; OPF duplicate checks | None beyond same key, same bytes and the deadline. Must be ruled out with FIS (OQ-A, OQ-K) | High |
> | 14 | PayX status API | Unreachable | Terminal-not-reported alarm | Message redelivered; sweeper dispatches a report-only attempt | Medium |
> | 15 | Sweeper | Not running in the active region | No-sweep alarm | Normal path still works; lost messages wait | Medium |
> | 16 | Region | Lost with payments in flight | Health check | Routing control; adoption by the other sweeper | Medium |
> | 17 | Region | Record not replicated before the loss | PayX SLA alert | Waits for the region; removed by strong consistency (8.6) | High |
> | 18 | Deploy | The conversion changes while payments wait | Contract test | The stored payload is re-sent, never rebuilt | High |

### 9. Risks, last row

**Replace the Mitigation cell**
> Backoff caps the call rate; DLQ alarm; DLQ redrive once the dependency returns.

**With**
> Circuit breaker at the lease (8.9): payments wait in the table without using receives, one probe at a time tests recovery, and the backlog drains at a set rate. No message reaches the DLQ because of an outage, so no redrive is needed afterwards.

### 12. Open questions: add

> - **OQ-I (new).** Is DynamoDB multi-Region strong consistency, with a witness region, available through the provisioning template, with customer-managed keys? It closes the replication-lag window (8.6).
> - **OQ-J (new).** OPF's rate limit and any planned maintenance windows, to set the breaker threshold, the probe interval and the drain rate (8.9).
> - **OQ-K (new).** If a repeat were ever accepted as a second payment, would OPF's own duplicate checks stop it?
> - **OQ-L (new).** The value of the submit deadline: FIS's key retention, the execution cut-off, and the longest wait product allows (8.10).

---

## Capability 3 (Submit Request to OPF API) - ADR-0003: full page

Replace the whole page with the text below. It is complete: nothing has to be copied over from Capability 10 by hand. Sections marked "moved" are the current Capability 10 text, unchanged except where noted.

> **Status:** Proposed — 2026-11-18. **Scope:** Phase 1 (transitional; retired at the full OPF exit in Phase 2). **ASV:** ASVWIREII. **Parent decision:** [ADR-0003].
> Capability 3 covers dispatch and submission: the managed dispatch (DynamoDB stream, EventBridge Pipe, SQS) and the **Wire II Submitter**, which detokenizes the BAN values, submits the stored pain.001 to OPF and reports the final status to PayX. It is the second half of one design; the first half, the **Wire II API**, is [Capability 10 (Wire II — pain.001 generation)].
>
> ### 1. Context and problem statement
> After the Wire II API has stored a payment and answered 202, the pain.001 must be submitted to the FIS OPF API and the outcome returned to PayX. OPF is a third-party API whose latency and availability are outside Capital One's control, so submission runs in the background, can be repeated safely, and survives the loss of a region.
>
> ### 2. Assumptions
> - The Wire II API has stored the built pain.001 V11 JSON, tokenized, and owns the transaction record (Capability 10).
> - The connection to the OPF API (auth, CodeConnect, firewall, DevExchange registration) is established by [Capability 2 (OPF API integration)].
> - BAN values are tokenized and are detokenized through the Turing API (Capital One Exchange asset 6809).
> - EventBridge Pipes, DynamoDB Streams and SQS are supported by the pipeline (OQ-H). EventBridge Pipes is confirmed as an approved service.
> - Turing and the PayX status API are reachable from both us-east-1 and us-west-2.
>
> ### 3. Constraints
> - The Submitter has no database access. It works on a record only through the Wire II API's internal endpoints.
> - Each submission carries the required `uuid` and `idempotency-key` headers, and re-sends the stored payload byte for byte.
> - Detokenized BAN values are held in memory for one attempt and are never logged or stored.
> - The queue message carries the UETR only.
>
> ### 4. Decision
> **Dispatch — managed, no code** (moved)
> The table's change stream feeds an EventBridge Pipe in each region. The Pipe passes only records whose owning region is that region and whose `dispatchRequested` flag is true, and sends an SQS message carrying the UETR alone. No payload or personal data enters the queue. Because the message is produced from the committed write, there is no dual-write gap between storing a payment and queueing it.
>
> **Submission path — Wire II Submitter** (moved; step 2 and step 6 extended)
> 1. Receive the SQS message (UETR).
> 2. Lease the record through the API (POST /internal/wires/{uetr}/leases). The API moves it to SUBMITTING, clears `dispatchRequested`, and returns the stored payload and `uuid`. A refused lease (not owner, lease held, already reported, deadline passed, **or circuit breaker open**) means the message is deleted or left to reappear.
> 3. Detokenize the BAN value(s) through Turing (POST /enterprise/detokenize, OAuth token from the DevExchange Gateway, type = BAN, GLOBALBAN, UKBAN or NONUSBAN). The clear-text value is held in memory for this attempt only.
> 4. Submit the stored payload to the OPF API (POST /customer-credit-transfer-initiations) with the stored `uuid` and `idempotency-key` = UETR.
> 5. Record the outcome through the API (PUT /internal/wires/{uetr}/outcome): ACCEPTED, REJECTED or FAILED_RETRYABLE. **With FAILED_RETRYABLE the Submitter states the cause and whether the request left Capital One**, so the API can count failures for the breaker and keep `unansweredAttempts` right.
> 6. Finish: for a terminal outcome, report the final status through the PayX status API, mark it reported (PATCH /internal/wires/{uetr}) and delete the message. For a retryable outcome, leave the message and set its visibility to the next backoff; SQS redelivers it. After the maximum number of receives the message moves to the DLQ and the record is marked PARKED, with an alert. **A timeout or a 5xx is never retried at once inside the attempt**, because OPF may still be working on the first request.
>
> **Runtime.** The Submitter runs as an ECS service that scales on queue depth. If it is run as a Lambda with an SQS trigger instead, it is written in Go: Java is discouraged for Lambdas because of cold starts. The contract is the same.
>
> ### 4.1 OPF response mapping (moved; rows 4 and 5 extended)
> | OPF response | Wire II status | Reported to PayX | PayX stage |
> |---|---|---|---|
> | 202 Accepted | ACCEPTED | Accepted | `.WIRECLEARING` |
> | 400 Bad Request (validation) | REJECTED | Rejected | `.WIRECLEARING.STOP` |
> | 422 Idempotent replay, different payload | REJECTED (alert) | Rejected — duplicate UETR with different content | `.WIRECLEARING.STOP` |
> | 401 / 403 | FAILED_RETRYABLE (alert); counts toward the circuit breaker | Nothing yet — redelivered after backoff, or held while the breaker is open | none until a terminal outcome |
> | 500 / 503 / timeout | FAILED_RETRYABLE; counts toward the circuit breaker | Nothing yet — redelivered after backoff, or held while the breaker is open | none until a terminal outcome |
> | Maximum receives reached | PARKED (alert), reason `MAX_RECEIVES` | Nothing yet — an operator replay revives it | none until a terminal outcome |
>
> Warning panel, below this table: A 422 should be impossible on a genuine replay, because the stored payload is re-sent byte for byte and a rebuilt payload is deterministic. Any 422 therefore signals a defect or a reused UETR, and raises an alert.
>
> ### 4.2 Turing detokenize response handling (moved; three rows extended)
> Read against Turing's documented error codes for 203, rather than one blanket policy. "Retryable" means the message is redelivered after backoff.
>
> | Turing response | Outcome | Treatment |
> |---|---|---|
> | 200 | Success — clear-text value returned | Proceed to the OPF submission |
> | 203, code 105 (ID parameter length exceeds limit) | Malformed request construction | Block submission; non-retryable; alert — this is a Wire II defect, not a transient issue |
> | 203, code 106 (VALUE parameter invalid/out of range) | The stored token value itself is malformed | Block submission; non-retryable; alert — indicates a bad token upstream of this call |
> | 203, code 108 (token parameter invalid) | Not a valid Turing token | Block submission; non-retryable; alert — cannot detokenize what isn't a token |
> | 203, code 109 (lookup database internal error) | Transient backend lookup failure | Retryable — same bucket as Turing/OPF 500; counts toward the Turing circuit breaker |
> | 400 | Bad request (malformed call) | Block submission; non-retryable; alert — same bucket as OPF 400 |
> | 401 / 403 | Unauthorized / client not entitled | Technical failure; alert — same bucket as OPF 401/403; counts toward the Turing circuit breaker |
> | 500 | Turing backend error | Technical failure, retryable — same bucket as OPF 500/503; counts toward the Turing circuit breaker |
>
> ### 5. Dispatch and retry (moved; three rows changed)
> | Layer | Owner | Trigger | Proposed starting values |
> |---|---|---|---|
> | Pipe dispatch | EventBridge Pipe | A write with `dispatchRequested` = true and owner = this region | Pipe retries and its own DLQ for undeliverable events; anything lost is re-dispatched by the recovery sweeper. |
> | Redelivery with backoff | SQS + Wire II Submitter | Retryable outcome (5xx, timeout, 401 / 403, Turing 203 code 109) while the breaker is closed | Visibility set from the receive count: 30 s, 1 min, 2 min, 5 min, 10 min, then 15 min. |
> | Circuit breaker | Wire II API | Several different payments failing in a row | Opens after 5; probes after 1, 2, 4, 8 minutes, then every 15; drains 60 per sweep (Capability 10, 8.9). |
> | Lease expiry | SQS | A Submitter task stopped mid-attempt | Visibility timeout and lease TTL both exceed the worst-case Turing + OPF attempt. |
> | Parking | SQS DLQ | Maximum receives reached (about 2 hours of retries), or the submit deadline | Alarm when a message arrives in the DLQ; the API marks the record PARKED with its reason. |
> | Recovery sweep | Wire II API (active region) | Stalled or unreported record; record of an inactive region | Every minute; stall threshold about 20 min; adoption threshold about 2–3 min (Capability 10, 8.3). |
> | Operator replay | Operations | Investigated PARKED record | POST `/internal/wires/{uetr}/dispatches`. The DLQ message itself is not needed: it holds only the UETR and expires by itself (DLQ retention 24 hours, working queue 12 hours). |
> | PayX call retry | PayX | No 202 received (timeout or 5xx) | Per PayX policy; idempotent on the UETR. |
>
> *The starting values are proposals, to be tuned in performance and resilience testing.*
>
> **The DLQ and the sweeper.** A queue message is only a pointer; the payment is in the table. SQS redelivery handles normal retries. The sweeper acts only after the stall threshold, which is longer than the longest backoff, so the two never race. A message in the DLQ means a defect or a payment that failed for two hours while others succeeded; the record is PARKED and an operator replays the record, not the message.
>
> ### 6. Wire II Submitter runtime (moved)
> - Consumer concurrency is the back-pressure limit on Turing and OPF; bursts wait in the queue.
> - Scales on queue depth and the age of the oldest message, independently of the API.
> - On deploy or scale-in, tasks stop receiving on SIGTERM and finish in-flight attempts within the stop timeout. An attempt that is cut off leaves its message to reappear and its lease to expire.
> - Holds the only clear-text BAN, in memory, and has no DynamoDB permissions.
>
> ### 7. Security and least privilege
> | Identity | Allowed | Not allowed |
> |---|---|---|
> | EventBridge Pipe | Read the table stream; send to the submit queue | Anything else |
> | Wire II Submitter | Receive / delete / change visibility on its queue; Turing; OPF; PayX status API; the Wire II internal endpoints | Any DynamoDB access |
>
> Queues and DLQs are encrypted with a customer-managed KMS key, and messages carry the UETR only. Logs carry the UETR for correlation, and never a clear-text BAN or a full request or response body.
>
> ### 8. Consequences
> - Retry, backoff and parking come from SQS; the circuit breaker keeps a vendor outage out of the DLQ.
> - The component that holds clear-text BANs has no data-store access.
> - Each submission step makes an internal call to the API, so the Submitter depends on the API being up in its region.
> - More managed resources per region: a Pipe, a queue, a DLQ and the Pipe's own DLQ.
>
> ### 9. Technical debt incurred
> Transitional. The Submitter and the dispatch resources exist to satisfy OPF's interim API and are retired at the full OPF exit (Phase 2).
>
> ### 10. Open questions
> Tracked on [Capability 10 (Wire II — pain.001 generation)], section 12: OQ-A, OQ-B, OQ-H, OQ-J, OQ-K for this page.

---

## Merged page: Capability 10 (Wire II — pain.001 Generation, Detokenization & OPF Submission) - ADR-0003

**Replace the red status panel** ("Status: DRAFT — published, not yet reviewed or approved. This page merges ...") **with**
> **Status: SUPERSEDED — 2026-11-18.** This page described one synchronous service covering Capabilities 10 and 3. The design is now asynchronous with two microservices, and the capabilities are split again: [Capability 10 (Wire II — pain.001 generation)] for the Wire II API and [Capability 3 (Submit Request to OPF API)] for dispatch and the Wire II Submitter. This page is kept for history only.

Then move the page under "Old". Delete its last section, "What happens to the current pages", which no longer applies.

---

## Capability 2 (OPF API integration) - ADR-0003

### Decision, first paragraph

**Replace**
> the Wire II service (`bank-wire-rail—ii-api`, Capability 10 (Wire II — pain.001 generation) - ADR-0003) calls the FIS-hosted

**With**
> the Wire II Submitter (ASVWIREII, [Capability 3 (Submit Request to OPF API) - ADR-0003]) calls the FIS-hosted

**And replace, at the end of the same paragraph**
> the call itself is made by the Wire II service.

**With**
> the call itself is made by the Wire II Submitter, in the background, after the Wire II API ([Capability 10 (Wire II — pain.001 generation) - ADR-0003]) has stored the payment and answered PayX with 202.

### Implications, first bullet

**Replace**
> are both done by the Wire II service, `bank-wire-ii-api`.

**With**
> are done by two microservices in ASVWIREII: the Wire II API (Capability 10) and the Wire II Submitter (Capability 3).

### Questions table, row 4 (Non-happy path responses), the "Resolved" sub-bullet

**Replace**
> Synchronous error codes are mapped by the Wire II service and returned to PayX.

**With**
> OPF's response codes are mapped by the Wire II Submitter (Capability 3). The final status reaches PayX later, through the PayX status API.

---

## ADR-0003 main page: capability summary table

Replace the single merged row with two rows.

| # | Capability | Summary |
|---|---|---|
| 3 | Submit Request to OPF API | Managed dispatch (DynamoDB stream, EventBridge Pipe, SQS) and the Wire II Submitter: leases the record through the Wire II API, detokenizes through Turing, submits the stored pain.001 to OPF with the UETR as idempotency key, and reports the final status to PayX. |
| 10 | Wire II — pain.001 generation | The Wire II API: idempotent on the UETR, builds and stores the pain.001.001.11 JSON, answers 202, owns the transaction record, and runs the recovery sweeper and the circuit breaker. |

---

## Build pieces, for story creation

**Capability 10: Wire II API**
- PayX-facing `POST /wires` and `GET /wires/{uetr}` with the idempotency rules.
- pain.001 V11 JSON build with deterministic identifiers; payload hash.
- Transaction table: record, sparse sweeper index, control records, encryption, retention.
- Internal API: leases, outcome, reported, operator dispatch.
- Recovery sweeper: stalled and unreported records, adoption, lock.
- Circuit breaker: counting, lease gate, probe, drain, manual control.
- Submit deadline and the parked reasons.
- Regional set-up: routing control read, health check, warm standby.
- Alarms and metrics.

**Capability 3: dispatch and Wire II Submitter**
- EventBridge Pipe with filter, submit queue, DLQs, per region.
- Submitter: receive, lease, Turing, OPF call, outcome, PayX status report, message handling with backoff.
- OPF and Turing response mapping.
- Scaling, shutdown behaviour, least-privilege role.
- Alarms: queue age, DLQ arrival, Pipe failures.

**Shared**
- Failover game day in a non-production environment.
- FIS sandbox tests: repeat with the same key and payload, and two repeats at once (OQ-A).

## Open points to confirm before publishing

1. That the split follows the two microservices, as proposed here.
2. That the circuit breaker, the submit deadline and the failure analysis are accepted as additions to option A.
3. Whether Figures 1 to 5 stay on Capability 10 as they are. They show both microservices; no redraw is needed for the split. Figure 3 could gain one note: "DUE also covers a payment that waits while the breaker is open".
4. Whether strong consistency is wanted now or kept as an option (OQ-I).
