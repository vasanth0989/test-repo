# Capability 10 (Wire II — pain.001 generation) - ADR-0003: full page, ready to paste

Everything below the line "PAGE CONTENT STARTS HERE" is the whole page. Paste it over the whole page body. Nothing from the old page has to be kept.

## The markers in the text

Every marker is in double square brackets, so a search for `[[` finds each one. After pasting, work through them and delete the marker text.

| Marker | What to do | How many |
|---|---|---|
| `[[ INFO PANEL STARTS HERE ]]` … `[[ INFO PANEL ENDS HERE ]]` | Insert an Info panel. Move everything between the two markers into it. Delete both marker lines. | 1 |
| `[[ WARNING PANEL STARTS HERE ]]` … `[[ WARNING PANEL ENDS HERE ]]` | Insert a Warning panel. Move everything between the two markers into it. Delete both marker lines. | 2 |
| `[[ DIAGRAM GOES HERE: Figure n — import the file … ]]` | Insert a draw.io diagram at that spot and import the named file. Delete the marker line. | 5 |
| `[[LINK: text]]` | Replace the marker with the text inside it, and make that text a link to the page in the table below. | 9 |

## The links

| Link text | Points to | How many |
|---|---|---|
| ADR-0003 | The ADR-0003 main page | 2 |
| Capability 2 (OPF API integration) | The page "Capability 2 (OPF API integration) - ADR-0003" | 3 |
| Capability 3 (Submit Request to OPF API) | The page "Capability 3 (Submit Request to OPF API) - ADR-0003" | 4 |

## The diagrams

Five draw.io files are in the folder `diagrams`, next to this file. Each is named after the section it belongs to.

| Section | Figure | File |
|---|---|---|
| 7.1 Context and multi-region deployment | Figure 1 | `7.1-context-and-multi-region-deployment.drawio` |
| 7.2 Request, dispatch and submission | Figure 2 | `7.2-request-dispatch-and-submission.drawio` |
| 7.3 Transaction state machine | Figure 3 | `7.3-transaction-state-machine.drawio` |
| 7.4 Request handling, dispatch, submission and replay | Figure 4 | `7.4-request-handling-dispatch-submission-and-replay.drawio` |
| 7.5 Regional failover and auto-heal | Figure 5 | `7.5-regional-failover-and-auto-heal.drawio` |

A picture of each diagram is in `diagrams/previews`, so you can compare what Confluence shows after the import.

## Section numbers

The page keeps the numbering it has today: 8.1 to 8.10. Every reference inside the text points to these numbers.

| Section | Title | Compared with the page today |
|---|---|---|
| 8.1 | Transaction record and breaker record | Heading changed; Breaker table replaces "Control records" |
| 8.2 | Idempotency rules | Unchanged |
| 8.3 | Recovery Sweeper (Lambda) | Rewritten |
| 8.4 | Regional failover and auto-heal | Rewritten; the warning panel has nine bullets |
| 8.5 | Security and least privilege | One cell changed, one row and one bullet added |
| 8.6 | Observability | Two bullets changed, three added |
| 8.7 | Prolonged OPF or Turing outage: the circuit breaker | Rewritten; the path check is here |
| 8.8 | Submit deadline and unknown outcome | Unchanged |
| 8.9 | Partial regional failure | Rewritten |
| 8.10 | Failure analysis | 22 rows, was 18 |

## Three things to check after pasting

1. Section 11, the package name `…wireiiintg.converter.painv9`. The name is hard to read on the screen. If the page today spells it differently, keep the spelling of the page.
2. The headings "6.1" and "6.2". They are written as headings here. If the page today shows them as bold lines, either form is fine.
3. Six sentences that still said PayX never replays after a 202 now match the rule already on the page (PayX replays once when `.WIRECLEARING` misses its SLA): the Info panel, the third decision driver, the sentence under the 6.1 table, the sixth bullet of 8.6, the fifth "Negative" consequence and the risk "A payment accepted just before a regional loss".

---

**PAGE CONTENT STARTS HERE**

---
**`[[ INFO PANEL STARTS HERE ]]`**

**Status:** Proposed — decision updated 2026-11-19. **Scope:** Phase 1 (transitional; retired at the full OPF exit in Phase 2). **ASV:** ASVWIREII. **Parent decision:** [[LINK: ADR-0003]].

Capability 10 covers the **Wire II API** and the **Sweeper**. The Wire II API accepts the payment idempotently, builds and stores the pain.001, returns 202, is the only writer of a payment's request and outcome, and keeps the circuit breaker. The Sweeper is a scheduled Lambda outside the API: it puts stalled payments back on the normal path and takes over the sending of a region that cannot reach OPF (8.3, 8.4). Dispatch, BAN detokenization and OPF submission are [[LINK: Capability 3 (Submit Request to OPF API)]]: the **Wire II Submitter** takes work from SQS and submits to OPF, a third-party API, in the background. The two capabilities are one design and are read together. Every in-flight wire can be replayed safely, and the path heals itself when an AWS region is lost. Once PayX has its 202 it waits, so Wire II owns recovery all the way to the asynchronous `.WIRECLEARING` event.

**Deciders:** to be confirmed at the PIE design forum.

**`[[ INFO PANEL ENDS HERE ]]`**

## 1. Context and problem statement

As part of [[LINK: ADR-0003]] Phase 1, Payment Exchange (PayX) must initiate payment instructions in OPF for fulfilment to correspondent banks, but PayX does not integrate with OPF's external API (FIS UEAPI) directly. ASVWIREII sits between PayX and OPF: it converts the completed PayX payment object into the ISO 20022 pain.001.001.11 message OPF requires (JSON only in Phase 1), resolves the tokenized account numbers through Turing, submits to OPF, and returns the outcome to PayX so the payment is advanced or stopped.

OPF is a third-party API reached over an external network. Its latency and availability are outside Capital One's control, so the PayX-facing call must not wait on it. The PayX-facing API must be idempotent. And the path must survive the loss of an AWS region without manual data repair: every wire in flight when the region failed must be replayable, and recovery must never create a duplicate wire at FIS or a false stop on a wire OPF already accepted.

## 2. Decision drivers

- **The PayX call must not wait on third-party latency** (OPF, and Turing in front of it).
- **The PayX-facing API is idempotent** on the UETR.
- **Wire II owns recovery after the 202.** PayX waits for the asynchronous `.WIRECLEARING` event that Wire II produces, and replays the call once only if that event misses its SLA.
- **Accepting a payment and submitting it are separate microservices**, scaled and deployed independently.
- **OPF accepts pain.001.001.11 in JSON only**, wrapped in a Business Application Header — see [[LINK: Capability 2 (OPF API integration)]].
- **Clear-text BAN values must never be persisted or logged.**
- **Replay must be safe:** no duplicate wire at FIS, and no `.WIRECLEARING.STOP` for a wire OPF accepted.
- **Automatic recovery from the loss of us-east-1**, with no manual data restore.

## 3. Assumptions

- The existing pain.001 converter is the reference implementation and is extended to produce the V11 output OPF requires; build detail is in section 11 and in the main ADR §4d.
- The UETR received from the Payment Network Gateway is the idempotency key, and FIS honours it for the life of a payment (to be confirmed — OQ-A).
- The connection to the OPF API (auth, CodeConnect, firewall, DevExchange registration) is established by [[LINK: Capability 2 (OPF API integration)]].
- BAN values in the PayX payment object are tokenized (Turing BAN / GLOBALBAN / UKBAN / NONUSBAN depending on the destination) and are detokenized through the Turing API (Capital One Exchange asset 6809).
- PayX accepts a 202 from Wire II and receives the final outcome later through the PayX status API (OQ-G).
- After a 202, PayX waits for `.WIRECLEARING`. PayX re-calls when it got no 202 (timeout or 5xx). When `.WIRECLEARING` misses its SLA, PayX alerts and replays the call once with the same UETR; the replay is idempotent (OQ-D). Wire II still owns recovery in every case where the record exists.
- Regional routing (for example Route 53 ARC) decides only where PayX calls land, on the API health check (OQ-E). Wire II reads no routing state. Which region sends a payment is decided by the takeover rule (8.4).
- EventBridge Pipes, DynamoDB Streams and SQS are supported by the pipeline (OQ-H).
- Turing and the PayX status API are reachable from both us-east-1 and us-west-2.
- FIS offers a call that creates no payment and can serve as a path check, or a token request to the FIS gateway is acceptable for that purpose (OQ-M).

## 4. Constraints

- The OPF API accepts pain.001.001.11 in JSON only — no XML on the wire.
- The message must be a valid pain.001.001.11 wrapped in a Business Application Header, per the OPF API contract ([[LINK: Capability 2 (OPF API integration)]]).
- The OPF API returns 202 (no content) on acceptance, or an error code. No pain.002 over MQ is received in Phase 1.
- Each submission carries the required `uuid` and `idempotency-key` headers.
- Detokenized BAN values are never logged and never written to DynamoDB, SQS or any other store.

## 5. Considered options

| # | Option | How it works | Main trade-off | Outcome |
|---|---|---|---|---|
| A | **Two microservices: Wire II API (the only writer of a payment's request and outcome) + Wire II Submitter, dispatched through DynamoDB stream → EventBridge Pipe → SQS, with a scheduled Sweeper Lambda as the safety net** | The API stores the payment and returns 202; the table's change stream feeds SQS through a filtered Pipe; the Submitter consumes SQS and works on the record only through the API's internal endpoints. A recovery Sweeper, a scheduled Lambda outside the API, re-dispatches anything whose message is gone and takes over the sending of a region that cannot reach OPF. | A Pipe, a queue and a DLQ per region (managed, no code), a small Sweeper Lambda per region and a small Breaker table; one extra internal call per step. | **Chosen** |
| B | One service, asynchronous, with an in-service reconciler | The API returns 202; a background worker and a 30-second reconciler in the same service submit and retry. | No queue to run, but request handling and third-party calls share one runtime, and all retry and scheduling is custom code. Option A keeps normal retries in SQS and needs only a small, dispatch-only sweeper as a safety net. | Not chosen |
| C | API writes DynamoDB, then sends to SQS itself | Two separate writes in the request path. | Dual write: a crash between the two strands a payment unless a sweeper is added. | Not chosen |
| D | Two microservices that both read and write the table | The Submitter updates the record directly. | Shared-database coupling: lease and replay rules duplicated in two codebases; unclear record ownership; the component holding clear-text BANs would also hold data-store access. | Not chosen. The Sweeper's direct access (8.3) is a different thing: a short list of conditional updates, no access to the stored request or the outcome, and no clear-text BAN. |
| E | Synchronous inline Wire II | One request lifecycle including the OPF call. | OPF latency becomes PayX latency. | Not chosen |
| F | Event-driven pipeline (Relay, Submit queue, Submitter, Sweeper, Notify queue, Notifier, control records) | Several Lambdas and queues with a separate notify path. | About ten components and three overlapping recovery paths. | Not chosen as a whole. Four of its parts are adopted into option A, because option A alone leaves a prolonged vendor outage to operators: the circuit breaker (8.7), the submit deadline with its two reasons (8.8), the failure analysis (8.10) and the Sweeper as a scheduled Lambda outside the API (8.3). Its separate notify path, its extra queues, its Relay and its Notifier are not adopted. |
| G | Wire II → SQS → separate Submit-to-OPF service, outcome path unspecified | As earlier proposed at architecture review. | No defined way for the outcome to reach PayX. Option A closes this gap: the Submitter records the outcome through the API and reports it to PayX. | Rejected at architecture review |

## 6. Decision

**Option A.** Capabilities 10 and 3 are delivered by two microservices in ASVWIREII, with managed dispatch between them: the Wire II API (this page) and the Wire II Submitter ([[LINK: Capability 3 (Submit Request to OPF API)]]). A scheduled Sweeper Lambda (this page, 8.3) is the safety net. All of them run in us-east-1, which serves the PayX calls, and in us-west-2, the warm standby.

| Component | Responsibility | Data access | Capability |
|---|---|---|---|
| Wire II API (Fargate) | Idempotent PayX-facing `POST /wires` and `GET /wires/{uetr}`; builds and stores the pain.001; the only writer of a payment's request and outcome; internal endpoints for the Submitter; keeps the circuit breaker records. | Reads and writes the transaction table and the Breaker table. | 10 |
| Sweeper (Lambda) | Runs every minute in both regions. Re-dispatches stalled or unreported records, names the breaker probe, drains after an outage, parks a record at its deadline, sends the path check, and adopts the unfinished records of a region that is out. Never calls Turing, OPF, PayX or the API. | Transaction table: query on the sweeper index and a short list of conditional updates (8.3). Breaker table: read, and conditional update of the probe fields. Sends path-check messages to its region's submit queue. | 10 |
| Dispatch (managed) | DynamoDB stream → EventBridge Pipe, filtered to this region's records that request dispatch → SQS submit queue (message = UETR only), with a DLQ. | Stream read; queue send. | 3 |
| Wire II Submitter (Fargate) | Consumes SQS; leases the record through the API; detokenizes through Turing; submits to OPF; records the outcome through the API; reports the final status to PayX. | No database access. The only component that holds a clear-text BAN, in memory only. | 3 |

**Request path — Wire II API, synchronous, no third-party call**

1. Build the pain.001.001.11 JSON in memory with BAN fields still tokenized, deriving every generated identifier and timestamp (BAH and group header) from the UETR and the payment, never from the wall clock. A build or validation failure is a rejection.
2. Register the payment with one conditional put-if-new keyed on the UETR: the payment hash, the built payload, the request `uuid`, status DUE, the owning region, and `dispatchRequested` = true. A UETR that already exists is a replay (section 8.2).
3. Respond to PayX straight away (section 6.1). The response never waits on Turing or OPF.

**Dispatch — managed, no code**

The table's change stream feeds an EventBridge Pipe in each region. The Pipe passes only records whose owning region is that region and whose `dispatchRequested` flag is true, and sends an SQS message carrying the UETR alone. No payload or personal data enters the queue. Because the message is produced from the committed write, there is no dual-write gap between storing a payment and queueing it.

**Recovery path — Sweeper Lambda**

Nothing outside Wire II is relied on to re-drive a payment that has lost its queue message. Every minute a Sweeper Lambda runs in each region, outside the Wire II API (section 8.3). It finds records of its own region that are stalled or whose outcome PayX was never told, and requests dispatch for them again. When the other region is out and its own region is able, it adopts the other region's unfinished records and requests dispatch for those too (section 8.4). The Sweeper never calls Turing, OPF or PayX, and it calls none of the API's endpoints; it only puts work back on the normal path.

Submission, the OPF response mapping and the Turing response handling are in [[LINK: Capability 3 (Submit Request to OPF API)]].

Scope is credit transfer (pain.001) only; pain.008 is Phase 2 scope.

### 6.1 PayX-facing API — idempotent on the UETR

| Request | Response | Effect |
|---|---|---|
| `POST /wires`, new UETR | 202 Accepted + `Location: /wires/{uetr}` | Payment stored and dispatched; the outcome follows through the PayX status API. |
| `POST /wires`, build or validation failure | 400 Bad Request (reason in the error body) | Recorded as REJECTED; PayX produces `.WIRECLEARING.STOP`. |
| `POST /wires`, same UETR, same payload, in flight | 202 Accepted (same as the first response) | No change. Recovery is owned by Wire II: SQS redelivery and the recovery Sweeper. |
| `POST /wires`, same UETR, same payload, terminal | 200 OK with the stored outcome | OPF is never called again. |
| `POST /wires`, same UETR, different payload | 409 Conflict — duplicate UETR with different content | Record unchanged; alert. PayX produces `.WIRECLEARING.STOP`. |
| Two first calls at the same moment | One 202; the other gets the replay response | The conditional put admits exactly one. |
| `GET /wires/{uetr}` | 200 OK with the current status | Lets PayX check a payment whose status report it did not receive. |

PayX re-calls when it got no 202 (a timeout or a 5xx), and once more if `.WIRECLEARING` misses its SLA (OQ-D); a repeat call is normally a lost response rather than a recovery request. The `Idempotency-Key` header must equal the UETR in the payload; a mismatch is a 400. Paths and payloads are finalised in the Wire II API specification.

### 6.2 Internal API — used only by the Submitter

| Endpoint | Purpose | Guard |
|---|---|---|
| `POST /internal/wires/{uetr}/leases` | Claim the record for one attempt; returns the stored payload, `uuid` and status. | Conditional: owner = this region, no live lease, not reported, submit deadline not passed, and the circuit breaker for OPF and Turing closed (or this UETR is the probe, section 8.7). Terminal-but-unreported returns the outcome so the Submitter can report it. A lease refused because the breaker is open tells the Submitter to delete the message; the record waits and the Sweeper dispatches it again later. |
| `PUT /internal/wires/{uetr}/outcome` | Record the OPF result and release the lease. | Conditional on the lease id; the first terminal outcome wins. |
| `PATCH /internal/wires/{uetr}` | Mark the outcome as reported to PayX. | Only on a terminal record. |
| `POST /internal/breakers/{dependency}/checks` | Record the result of a path check on this region's breaker record for that dependency (OPF or TURING), section 8.7. A pass sets `lastProvenAt` and clears `checkFailures`. A fail raises `checkFailures`; three in a row open the breaker. | Submitter only. |
| `POST /internal/wires/{uetr}/dispatches` | Operator replay: request dispatch again (revive a PARKED record, or force a re-drive). | Operations role only; never on a terminal record that is already reported. |
| `PUT /internal/breakers/{dependency}` | Open or close a circuit breaker by hand, for planned maintenance or to force a state. | Operations role only; audited. |

The internal endpoints are private to ASVWIREII, authenticated service-to-service (operator replay and manual breaker control with an operations role), and not published on Exchange. The Sweeper calls none of them: it works on the tables directly (section 8.3).

## 7. Architecture views

The diagrams are native draw.io diagrams and can be edited in place.

### 7.1 Context and multi-region deployment

**`[[ DIAGRAM GOES HERE: Figure 1 — import the file 7.1-context-and-multi-region-deployment.drawio ]]`**

*Figure 1 — The two microservices, the Sweeper Lambda and the managed dispatch between them. us-east-1 serves the PayX calls; us-west-2 is the warm standby with the same components.*

- Only the Wire II API writes a payment's request and outcome. The Sweeper Lambda makes a short list of conditional updates (8.3). The Submitter has no table access and reaches the record through the API's internal endpoints (steps 5 and 8 in the diagram).
- Each region's Pipe passes only that region's records, so replicated writes from the other region are never dispatched twice.
- The circuit breaker is a record in its own Breaker table. The API reads it at every lease (8.7).
- Failover works only if FIS allow-lists the NAT egress IPs of both regions, and credentials, secrets and KMS keys exist in us-west-2 (8.4).

### 7.2 Request, dispatch and submission

**`[[ DIAGRAM GOES HERE: Figure 2 — import the file 7.2-request-dispatch-and-submission.drawio ]]`**

*Figure 2 — The API responds before any third-party call; the Submitter does the Turing and OPF work.*

- Steps 1–4 make the payment durable and replayable, then respond. Nothing in them leaves Capital One.
- The queue message carries only the UETR; the Submitter gets the payload from the lease response (step 9).
- The clear-text BAN from step 10 is used for step 11 only and never reaches DynamoDB, SQS or the logs.

### 7.3 Transaction state machine

**`[[ DIAGRAM GOES HERE: Figure 3 — import the file 7.3-transaction-state-machine.drawio ]]`**

*Figure 3 — States of the Wire II transaction record. The Wire II API changes them; the one exception is PARKED at the submit deadline, which the Sweeper sets.*

- DUE is written together with the 202, so there is no window in which PayX was told "accepted" but nothing was stored.
- Retries are SQS redeliveries. PARKED means the message reached the DLQ after maximum receives, or the submit deadline passed (8.8).
- Anything whose message is gone is picked up by the recovery Sweeper, so no pending state depends on PayX.

### 7.4 Request handling, dispatch, submission and replay

**`[[ DIAGRAM GOES HERE: Figure 4 — import the file 7.4-request-handling-dispatch-submission-and-replay.drawio ]]`**

*Figure 4 — What each microservice does with a payment, first call or replay.*

- Every path that needs submission goes through the same mechanism: set `dispatchRequested` and let the Pipe enqueue it.
- A refused lease is normal: it is how duplicates, stale messages and stranded messages from another region are discarded.
- The Submitter handles two kinds of message: a payment and a path check (8.7).

### 7.5 Regional failover and auto-heal

**`[[ DIAGRAM GOES HERE: Figure 5 — import the file 7.5-regional-failover-and-auto-heal.drawio ]]`**

*Figure 5 — The worst case: us-east-1 is lost after OPF accepted the wire but before the outcome was recorded or PayX was told.*

- No PayX replay is needed: the us-west-2 Sweeper adopts the record once us-east-1 has proven nothing for five minutes, us-west-2 is able and the lease has expired (8.4).
- Adoption is just a write. Setting the owner to us-west-2 and requesting dispatch makes the us-west-2 Pipe enqueue it.

## 8. Detailed design

### 8.1 Transaction record and breaker record (DynamoDB Global Tables)

| Attribute | Type | Purpose |
|---|---|---|
| `uetr` (partition key) | string | Idempotency key from the Payment Network Gateway; one record per payment. |
| `status` | string | DUE · SUBMITTING · ACCEPTED · REJECTED · FAILED_RETRYABLE · PARKED. |
| `payloadHash` | string | SHA-256 of the canonical PayX payment object; detects a reused UETR with different content. |
| `pain001` | string | The built pain.001 V11 JSON with BAN fields still tokenized; re-sent byte for byte on every attempt. |
| `requestUuid` | string | The `uuid` header. Derived from the UETR as a name-based UUID, not random, and reused on every attempt. A record created again in the other region therefore sends exactly the same headers (OQ-A). |
| `ownerRegion` | string | The region whose Pipe dispatches the record and whose Submitter may lease it. |
| `dispatchRequested` | boolean | Set by the API on create, by the recovery Sweeper and by operator replay; cleared by a lease. The Pipe filter matches on it. |
| `pendingRegion` / `checkAt` | string / number | Sparse index keys for the recovery Sweeper, present only while the record is non-terminal or its outcome is unreported. `pendingRegion` = owner; `checkAt` = last update + stall threshold. |
| `leaseId` / `leaseUntil` | string / number | The current attempt's claim and its expiry. |
| `attempts` | number | OPF submission attempts. |
| `opfResponse` | map | HTTP status, OPF error code and time of the last response. Never a clear-text BAN. |
| `statusReported` | boolean | Whether PayX has been told the final status. |
| `createdRegion` | string | Where the record was created, for failover diagnostics. |
| `createdAt` / `updatedAt` | string (ISO-8601) | Audit timestamps. |
| `version` | number | Optimistic-lock counter checked on every conditional update. |
| `expiresAt` | number | DynamoDB TTL; retention to be set with records management (OQ-F). |
| `submitDeadline` | string (ISO-8601) | Set at acceptance. After it, no lease is granted and the payment is never sent automatically (8.8). |
| `unansweredAttempts` | number | How many attempts may have reached OPF without a clear answer. Raised when a lease is granted; taken back when the outcome is clear, or when the Submitter reports that nothing was sent. |
| `parkedReason` | string | Set with PARKED: `MAX_RECEIVES`, `NOT_SENT` or `OUTCOME_UNKNOWN` (8.8). Tells the operator whether a replay is safe. |

- The Wire II API reaches a record by its UETR: get, conditional put, conditional update. The Sweeper queries the sparse index and makes the conditional updates of its write contract (8.3). No other component has access; the Submitter has none. Removing `pendingRegion` when a record is terminal and reported keeps the index small.
- Encryption at rest uses a KMS customer-managed key in each replica region. The record holds Turing tokens, never clear-text BAN values.

**Breaker table.** The circuit breaker records are in a table of their own, the Breaker table: a DynamoDB Global Table with replicas in us-east-1 and us-west-2, on-demand capacity, encrypted with a customer-managed KMS key in each replica region like the transaction table. The partition key is `breakerId`, the region and the dependency: `<region>#<dependency>`. There are four records: `us-east-1#OPF`, `us-east-1#TURING`, `us-west-2#OPF` and `us-west-2#TURING`. The Global Table copies the records, which is how one region reads the other region's breaker. The table has no index and no Pipe, and nothing consumes its stream.

| Field | Meaning |
|---|---|
| `breakerId` (partition key) | `<region>#<dependency>`, for example `us-east-1#OPF` |
| `state` | CLOSED, OPEN or PROBING |
| `failedUetrs` | The different payments that failed in a row. A clear answer empties it |
| `openedAt` | When it opened |
| `nextProbeAt`, `probeInterval` | When the next probe may go, and the current wait |
| `probeUetr` | The one payment that may get a lease while the breaker is not closed |
| `lastProvenAt` | Last clear answer from OPF, or last passing path check |
| `checkFailures` | Path checks failed in a row |

The Wire II API keeps the state, the failed set, `openedAt`, `lastProvenAt` and `checkFailures`, and reads the record with one get at every lease request. The Sweeper reads the breaker records of both regions from its own replica. It writes `probeUetr` and `nextProbeAt` when it names a probe, and it puts the state back to OPEN with a doubled wait when a probe's lease runs out without an outcome. The Submitter never sees a breaker record; it only sees a granted or a refused lease. There is no sweeper lock: the Sweeper Lambda has a reserved concurrency of 1.

**Why a table of its own.** A breaker record is not a transaction. It is always easy to reach: one get by key. Breaker records stay out of the transaction stream and out of anything that reads the transaction table in bulk, such as a report or an export. Permissions can be set per table. The cost is one more small Global Table to provision, encrypt and replicate. The option not taken was to keep the breaker records as extra items in the transaction table.

### 8.2 Idempotency rules (Wire II API)

1. **One record per UETR.** The first call creates it with a conditional put; any later call with the same UETR is a repeat. Two concurrent first calls resolve to one winner.
2. **Same UETR, different content** (payload hash differs): 409 Conflict; record unchanged; alert.
3. **Terminal record** (ACCEPTED or REJECTED): 200 with the stored outcome; never call OPF again. If PayX was never told, the recovery Sweeper dispatches a report-only attempt.
4. **In flight** (DUE, SUBMITTING, FAILED_RETRYABLE or PARKED): 202 and no change. Recovery is owned by Wire II — SQS redelivery, the recovery Sweeper (section 8.3) and operator replay.
5. **Never rebuild a stored payload.** Every attempt re-sends the stored pain.001 with the stored `uuid` and the same idempotency key.
6. **Deterministic generation.** The BAH business message identifier and the group-header message identifier are the UETR with hyphens removed (32 characters, within the 35-character limit). Creation date and time come from the PayX payment's creation timestamp (OQ-C), so a rebuilt payload is byte-identical. The `uuid` header is derived from the UETR in the same way.
7. **At-least-once delivery is expected.** SQS may deliver a message twice, and the Sweeper may dispatch a record that already has a message; the conditional lease admits one attempt, and the OPF idempotency key covers anything that slips through.

### 8.3 Recovery Sweeper (Lambda)

- **Where and when.** A Lambda in each region, written in Go, outside the Wire II API. EventBridge Scheduler starts it every minute. Its reserved concurrency is 1, so one run is active at a time and no lock record is needed; a second run would be harmless anyway, because every write is conditional. It runs in both regions all the time, whichever region serves the PayX calls.
- **How it finds work.** It queries the sparse index for `pendingRegion` = this region with `checkAt` ≤ now. During a takeover (8.4) it also queries `pendingRegion` = the other region and takes the unfinished records that have no live lease, oldest first. The index exists in every Global Table replica, so each region can see the other region's pending records.
- **What else it reads.** The breaker records of both regions, from its own replica of the Breaker table (8.1): its own to hold, probe and drain (8.7), and the other region's for the takeover rule (8.4).
- **Thresholds.** The same-region stall threshold exceeds the longest SQS backoff plus the lease TTL (about 20 minutes), so the Sweeper never races normal retries. A takeover starts when the other region has been out for five minutes and this region is able. It moves 60 records per sweep.
- **Never calls Turing, OPF or PayX, and calls none of the API's endpoints.** It only sets `dispatchRequested` (and the owner, when adopting), so all submissions still go through the Pipe, the queue, the lease and the stored payload.
- **A re-dispatch always changes the item.** A write that changes nothing produces no stream event. The Sweeper therefore always raises `version` and moves `checkAt` when it requests dispatch, also when `dispatchRequested` is already true.
- **Direct table access, through a short write contract.** The Sweeper writes the two tables directly and does not go through the API. Every write is a conditional update, so DynamoDB enforces the rules of the record whoever sends the write. Going through the API would put the sweep load back into the API tasks and add a caller the API must authenticate, for the same conditional writes. The rule is: the Wire II API is the only writer of a payment's request and outcome; the Sweeper writes only who sends a payment next. PARKED at the deadline is the one status the Sweeper sets. It never touches the stored request, the payload hash, the outcome or the reported flag.
- **What it costs.** Two deployables know the shape of the record. The Sweeper knows about ten attributes. Those attributes change only by adding, and the Sweeper's pipeline runs a contract test against the record schema.

**The Sweeper's whole write contract**

| Action | Write | Condition |
|---|---|---|
| Push a stalled payment again | Transaction table: `dispatchRequested` = true, `version` + 1, `checkAt` moved on | Owner is this region, version unchanged, not terminal and reported |
| Name the probe | Breaker table: `probeUetr`, `nextProbeAt`. Then the push above for that payment | Breaker is OPEN, a probe is due and a path check has passed |
| Adopt | Transaction table: `ownerRegion` = this region, `pendingRegion` = this region, `dispatchRequested` = true, `version` + 1 | Owner is the other region, no live lease, version unchanged |
| Reset a lost probe | Breaker table: `state` from PROBING back to OPEN, `probeInterval` doubled, `nextProbeAt` | State is PROBING, `probeUetr` unchanged, and that payment's lease has run out with no outcome |
| Park | Transaction table: `status` = PARKED, `parkedReason` | Deadline passed, no live lease, not terminal |
| Path check | A message on the submit queue of its region. No table write | This region has proven nothing for a minute |

**What it does with what it finds**

| Found | Sweeper action |
|---|---|
| DUE, SUBMITTING with an expired lease, or FAILED_RETRYABLE, owned by this region and stalled | Request dispatch again — the Submitter re-sends the stored payload with the same key. |
| ACCEPTED or REJECTED with the outcome not reported to PayX | Request dispatch again — the lease returns "terminal, not yet reported" and the Submitter only reports. Repeated until PayX has been told. |
| This region has proven nothing for a minute | Put a path-check message on this region's submit queue (8.7). |
| The other region has been out for five minutes and this region is able | Adopt the other region's unfinished records that have no live lease (owner = this region) and request dispatch. Oldest first, 60 per sweep. Stop as soon as the other region is able again. Never move a record back (8.4). |
| Both regions' breakers are open | No adoption. That is an outage of OPF itself. Records wait where they are. |
| The circuit breaker of this region is open | Hold: do not request dispatch for pending records. When a probe is due and a path check has passed, name one waiting record as the probe and request dispatch for it only. |
| A probe was named, and its lease has run out with no outcome | Treat the probe as failed: the breaker is OPEN again, the wait doubles, and the next probe is another record. |
| The circuit breaker has just closed | Drain: request dispatch for the waiting records, oldest first, 60 per sweep. |
| A pending record whose `submitDeadline` has passed, with no live lease | Mark PARKED with `parkedReason` = `NOT_SENT` or `OUTCOME_UNKNOWN`; alert (8.8). |
| PARKED | No automatic action — it stays alerted until an operator replays the record. |

Dispatch, retry and the Submitter runtime are in [[LINK: Capability 3 (Submit Request to OPF API)]].

### 8.4 Regional failover and auto-heal

- **Topology — accepting is active / passive; sending follows ability.** us-east-1 serves the PayX calls. us-west-2 runs the same microservices at minimum capacity, with its own Pipe, queue, DLQ and Sweeper, against its Global Table replicas. Each Pipe passes only its own region's records, so us-west-2 never dispatches a record that us-east-1 owns. Where a payment is accepted and where it is sent are two questions. Accepting needs a healthy API and the table. Sending needs a route to OPF. A region that cannot send keeps accepting, and the other region sends for it. Regional routing decides only where PayX calls land (OQ-E). Wire II reads no routing state.
- **Detection.** Two things are detected, by two means. A failed API is detected by the routing health check: new PayX calls go to the other region, and there is no data restore step, because the tables are already replicated. A region that cannot reach OPF, or a region that is lost, is detected from its breaker record: OPEN, or nothing proven (8.7).
- **Why the breaker state alone is not enough.** The standby has no traffic, so its breaker stays closed whatever happens to OPF. A closed breaker in a quiet region is not evidence. `lastProvenAt` and the path check supply the evidence (8.7).
- **Able and out.** A region is *able* when its breaker is CLOSED and `lastProvenAt` is less than three minutes old. A region is *out* when its breaker has been OPEN for five minutes, or it has proven nothing for five minutes. A region cannot be both.
- **Takeover — auto-heal, no PayX replay.** When the other region is out and this region is able, this region's Sweeper adopts the other region's unfinished records that have no live lease: oldest first, 60 per sweep. Adoption is one conditional write: owner and `pendingRegion` = this region, dispatch requested, `version` raised. It applies only if the owner is still the other region, no lease is live and the version is unchanged. The adopting region's Pipe then enqueues the record, and its Submitter re-sends the stored payload with the same key and `uuid`. OPF treats a payload it already accepted as an idempotent repeat (OQ-A). The takeover stops as soon as the other region is able again. Adopted records are never moved back.
- **Both breakers open.** Nobody adopts. That is an outage of OPF itself. Payments wait where they are, and no retry is used up.
- **New payments do not have to move.** The region that cannot send keeps answering 202, because a 202 never depends on OPF, and the other region sends each new payment about a minute after it is accepted. Moving the PayX calls as well is optional.
- **Who decides.** Three components, each with one job. The API of the failing region decides its own breaker from the outcomes its Submitter reports. The API of the other region keeps `lastProvenAt` from its path checks. The Sweeper of the other region reads both breaker records and adopts. No person and no routing control is involved.
- **The Sweeper runs in both regions all the time.** The two regions still never adopt from each other, because a region adopts only when it is able and the other region is out.
- **Stranded messages.** Messages left in the us-east-1 queue are harmless. When us-east-1 returns, its Submitter's lease request is refused because the record is owned by us-west-2, and the message is deleted.
- **Recovery objectives.** New payments are accepted wherever PayX calls land, and move when routing fails over on the API health check. In-flight payments: about six minutes for the first 60, which is five minutes until the region counts as out and then the next sweep; after that, 60 per minute. There is no dependence on PayX. The recovery point is the Global Table replication lag (typically about a second). Whether the five-minute wait fits the PayX status deadline is open (OQ-Q).
- **The replication-lag window.** A payment accepted in the last moment before the loss, whose record had not yet replicated, is not in us-west-2. This is the one case Wire II cannot heal by itself, because the record is not there. When `.WIRECLEARING` misses its SLA, PayX replays the call with the same UETR (OQ-D). us-west-2 has no record, creates it and submits. Deterministic generation makes the payload and the `uuid` identical, so if us-east-1 had already reached OPF, this is an idempotent repeat. When us-east-1 returns, the older copy loses to the newer one.
- **Closing the window (option).** With DynamoDB multi-Region strong consistency (replicas in us-east-1 and us-west-2 and a witness in a third region), the 202 is given only after a second region holds the record. The replication-lag window disappears, and a lease becomes a real cross-region lock. It needs platform confirmation (OQ-I); the design above works without it.
- **Fallback.** When us-east-1 is able again, us-west-2 stops adopting. us-east-1 sends the records it still owns and every new one it accepts. Adopted records finish in the region that adopted them. Routing moves the PayX calls back once replication has caught up; that changes nothing in sending.
- **Split-brain.** DynamoDB conditional writes are evaluated per region, and concurrent writes in two regions resolve last-writer-wins, so the lease is not a cross-region lock. Protection comes from four things together: the takeover rule, which cannot hold in both regions at once; region-scoped Pipes; the deterministic payload and `uuid`; and the UETR idempotency key at OPF.
- **The residual case.** The takeover rule is read from replicated breaker records. If replication between the regions stops while both are healthy, us-west-2 sees nothing new from us-east-1 and, after five minutes, treats it as out. The few payments that were in flight when replication stopped may be re-sent. The same key and the same bytes at OPF are the backstop, as in every other resend (OQ-A). Strong consistency would close that window (OQ-I).

**`[[ WARNING PANEL STARTS HERE ]]`**

**Failover prerequisites — required before go-live:**

- FIS allow-lists the NAT egress IPs of both regions.
- OPF, Turing and PayX status API credentials, and the Secrets Manager secrets, exist in us-west-2.
- KMS keys for the two tables, the queues and the secrets exist in both regions.
- The us-west-2 Pipe, queue and DLQ are deployed and enabled (not created at failover time).
- The Breaker table exists as a Global Table with replicas in both regions.
- The Sweeper Lambda and its one-minute schedule run in both regions.
- The FIS call for the path check is agreed (OQ-M), and path checks pass from both regions.
- Regional routing and its API health checks are in place. Wire II does not need to read any routing state.
- A regional failover game day passes in a non-production environment before production go-live.

**`[[ WARNING PANEL ENDS HERE ]]`**

### 8.5 Security and least privilege

| Identity | Allowed | Not allowed |
|---|---|---|
| Wire II API | Conditional get / put / update and the sweeper index query on the transaction table; get and conditional update on the Breaker table; KMS for both tables | SQS; Turing; OPF; PayX status API |
| EventBridge Pipe | Read the table stream; send to the submit queue | Anything else |
| Wire II Submitter | Receive / delete / change visibility on its queue; Turing; OPF; PayX status API; the Wire II internal endpoints | Any DynamoDB access |
| Sweeper (Lambda) | Transaction table: query on the sweeper index, and conditional update limited to `dispatchRequested`, `version`, `checkAt`, `ownerRegion`, `pendingRegion`, and `status` to PARKED with `parkedReason`. Breaker table: get, and conditional update of `probeUetr`, `nextProbeAt` and the reset of a lost probe. Send to its region's submit queue. KMS for both tables and the queue. | Put or delete on either table; the stored request, the payload hash, the outcome and the reported flag; Turing; OPF; PayX status API; the Wire II internal endpoints |

- Queues and DLQs are encrypted with a customer-managed KMS key, and messages carry the UETR only.
- Logs carry the UETR for correlation, and never a clear-text BAN or a full request or response body.
- Whether the pipeline can limit a Lambda's update to named attributes is open (OQ-N). Until then the conditional expressions and the contract test guard the Sweeper's writes.

### 8.6 Observability

- A metric per state transition, tagged with the region.
- Alarm on the age of the oldest message in the submit queue, on any message in the DLQ, and on Pipe delivery failures.
- Alarm on records that are terminal but not reported to PayX within the expected window, and on stalled records.
- Alarm on any OPF 422, any 409 from the API, and Global Table replication latency.
- Sweeper metrics per run: records re-dispatched, records adopted, PARKED count; alarm when a sweep has not run in a region for five minutes.
- On the PayX side, an alert when `.WIRECLEARING` is not produced within the SLA — the last line of defence, and the trigger of the one replay.
- Alarm when a circuit breaker opens, by dependency and region; and a separate alarm when it is open in one region while the other region is able.
- Alarm when a region is not able: its breaker is not closed, or nothing was proven within three minutes. This covers the standby too.
- Alarm on failing path checks, by region.
- Alarm when a takeover starts: one region's Sweeper adopts the other region's records.
- Alarm on PARKED records by `parkedReason`.

### 8.7 Prolonged OPF or Turing outage: the circuit breaker

OPF is a vendor system; an outage of hours must cost delay and nothing else. Without a breaker, every in-flight payment retries on its own until its message reaches the DLQ, and operators redrive hundreds of messages afterwards.

**Where it lives.** In the Breaker table (8.1): one record per dependency (OPF, Turing) and region, four in total, kept by the Wire II API. The API runs as several tasks in two regions; a breaker held in memory would exist once per task. As a record, every task reads the same state, and each region can read the other's. The gate is the lease: a Submitter cannot call OPF without one, and the API reads the breaker record at every lease request.

| Who | When | What |
|---|---|---|
| API, outcome call | Each attempt result | A failure adds the UETR to `failedUetrs`. The fifth different one sets OPEN and pages. A clear answer empties the set, sets `lastProvenAt` and closes the breaker |
| API, lease call | Each lease request | Reads only. Refuses unless the breaker is closed or the UETR is the probe |
| API, path-check call | Each check result | Pass: `lastProvenAt`, `checkFailures` = 0, and CLOSED if no payment is waiting. Fail: `checkFailures` + 1. Three set OPEN |
| Sweeper | Each minute | Names the probe when one is due. Holds while open. Drains after closing. Reads the other region's record for the takeover rule (8.4) |
| Operations | Planned maintenance, or to force a state | OPEN or CLOSED by hand, through the API. Restricted and audited |
| Submitter | Never | It only sees a granted or a refused lease |

**What counts as a failure.** A 5xx, a timeout or connection error, and a 401 or 403. A 2xx, a 400 and a 422 are clear answers: OPF is working, and a clear answer closes the breaker. For Turing the counted answers are 203 code 109, 401 / 403 and 500.

| Step | What happens |
|---|---|
| Opens | When a set number of *different* payments have failed in a row (proposed: 5), or three path checks have failed in a row, the API sets the breaker to OPEN, records `openedAt` and pages. One payment that keeps failing does not open it. |
| While open | Lease requests are refused. The Submitter deletes the message. The record stays DUE or FAILED_RETRYABLE; no receive is burned and no attempt is counted. The API keeps answering 202 to PayX. Queues stay empty; the backlog is in the table. |
| Path check first | A payment is spent as a probe only after a path check has passed. While the route itself is broken, the check fails and no payment is spent. |
| Probe | The only test that proves OPF takes payments is one real payment. When a probe is due, the Sweeper names one waiting record and requests dispatch; the API grants the lease for that UETR only. A clear answer from OPF, accepted or rejected, closes the breaker and sets `lastProvenAt`. A failure reopens it and doubles the wait: 1, 2, 4, 8 minutes, then every 15. The next probe is a different payment. |
| A probe that never reports | If no outcome arrives, the Sweeper puts the state back to OPEN when the probe's lease runs out, and the wait doubles. |
| No payment waiting | A region with no payment waiting has nothing to probe with. Its breaker closes on a passing path check alone. |
| Any clear answer | Any clear answer from OPF closes the breaker, including the answer to an attempt that was leased just before the breaker opened. |
| Drain | After closing, the Sweeper requests dispatch for the waiting records oldest first, a set number per sweep (proposed: 60). Consumer concurrency still caps the call rate. |
| By hand | Operations can open or close a breaker through the API (`PUT /internal/breakers/{dependency}`). Restricted and audited. |

**The path check.** A closed breaker in a region with no traffic is not evidence that the region can reach OPF. The path check supplies the evidence, and `lastProvenAt` on the breaker record keeps it.

| Step | What happens |
|---|---|
| When | A region has proven nothing for a minute: no clear answer from OPF and no passing check. The Sweeper sees this on each run. It runs in both regions all the time. |
| Message | The Sweeper puts a path-check message on the region's submit queue: a small JSON body naming the dependency, for example `{"check":"OPF"}`, with its send time. It writes nothing to the tables. |
| Call | The Submitter makes one authenticated call to FIS that creates no payment. The preferred call is a heartbeat or status URL from FIS, if one exists; the FIS specification available today exposes only the payment POST. The fallback is a token request to the FIS gateway, which is the same host as the payment API: it proves the region's network path and credentials, not the OPF application behind the gateway. Which call to use is open (OQ-M). |
| Road | The same as a payment's: queue, Submitter, proxy, partner gateway, FIS, API, table. |
| Report | The Submitter reports the result to the API of its region: `POST /internal/breakers/{dependency}/checks`. |
| Pass | The API sets `lastProvenAt` and `checkFailures` = 0. If the breaker is OPEN and no payment of the region is waiting, it closes the breaker. |
| Fail | The API raises `checkFailures`. Three failed checks in a row open the breaker. |
| One attempt only | The Submitter deletes a check message after one attempt, whatever the result, and deletes unread any check older than two minutes. A check is never retried: the next minute brings a new one. |

- A real answer proves more than a check. A clear answer from OPF to a real payment sets `lastProvenAt` and resets `checkFailures`. The check runs only when nothing real was proven in the last minute.
- A passing check does not prove that OPF takes payments. If the check passes while OPF itself is failing, real failures open the breaker.
- With the fallback call, an OPF application outage leaves the token request passing. The standby region would look able, adopt one or two batches (up to 60 per sweep), fail on them and open its own breaker. Nothing is lost or sent twice; those payments then wait in the standby region.

**Proposed values** (configuration, to be tuned with OPF's rate limit, OQ-J)

| Setting | Proposed |
|---|---|
| Opens after | 5 different payments failing in a row, or 3 failed path checks |
| First probe | 1 minute after opening |
| Probe interval | Doubles after each failed probe: 1, 2, 4, 8 minutes, then every 15 minutes |
| Path check | When nothing was proven for 1 minute |
| Able | CLOSED and proven within 3 minutes |
| Out | OPEN for 5 minutes, or nothing proven for 5 minutes |
| Drain and adoption | 60 records per sweep, oldest first |

**Example.** OPF is down from 13:00 to 17:00. The breaker opens at about 13:02. Probes go out at 13:03, 13:05, 13:09, 13:17, then every 15 minutes, each one only after a path check has passed. The first probe after 17:00 goes at 17:02 and succeeds; the backlog then drains at 60 a minute. No message reaches the DLQ and no operator action is needed. Both regions' breakers are open during the outage, so no takeover happens. Payments whose submit deadline passes during the outage are parked (8.8).

At low volume five different payments may never fail. If the route is broken, the path check opens the breaker after three failed checks, about three minutes after the route breaks. If the route works and OPF itself fails, each payment follows the normal redelivery backoff, which has the same effect.

The Turing breaker uses the same kind of record and the same steps. Leases are refused while either breaker is open. Whether the Turing breaker gets its own path check is open (OQ-P).

### 8.8 Submit deadline and unknown outcome

`submitDeadline` is set at acceptance: the earliest of how long FIS remembers an idempotency key (OQ-A), the cut-off for the payment's execution date, and how long product allows a payment to wait. No lease is granted after it, so a retry can never reach OPF after the key has been forgotten.

When the deadline passes, the Sweeper marks the record PARKED, with one of two reasons:

- **`NOT_SENT`**: no attempt ever reached OPF, for example because the breaker was open the whole time. An operator replay is always safe.
- **`OUTCOME_UNKNOWN`**: at least one attempt ended without a clear answer (`unansweredAttempts` above zero). OPF may hold the payment. It is not replayed before someone has checked OPF.

PARKED reached through maximum receives keeps the reason `MAX_RECEIVES`.

### 8.9 Partial regional failure

A region can be healthy and still unable to send, for example when a security group or proxy change cuts its Submitter off from the partner gateway. The API accepts payments and grants leases; every OPF call times out.

- The breaker of that region opens (8.7). Payments wait; none is parked or sent to the DLQ because of it.
- The region keeps answering 202. Routing does not move, because the API is healthy.
- Its path checks fail, so no payment is spent as a probe.
- Once its breaker has been open for five minutes and the other region is able, the other region's Sweeper adopts the waiting records, 60 per sweep, and its Submitter sends them (8.4).
- The alarms "a region is not able" and "a takeover has started" name the situation (8.6).
- Attempts that timed out count as unanswered although nothing reached OPF. The resend with the same key and bytes resolves them.
- When the route is repaired, a passing path check closes the breaker if no payment is waiting, and the other region stops adopting.

**One payment, with clock times: us-east-1 is cut from OPF**

| Time | What happens |
|---|---|
| 13:00:00 | A proxy change cuts the us-east-1 Submitter from the partner gateway. us-west-2 is unaffected; its last path check passed at 12:59. |
| 13:00:10 | Payment W arrives in us-east-1. Stored, 202 to PayX, status DUE, owner us-east-1. |
| 13:00:12 | The Submitter leases W and posts to OPF. Timeout. Outcome FAILED_RETRYABLE. The API counts W on the us-east-1 breaker (1 of 5). |
| 13:01 | Four more payments time out. The us-east-1 breaker opens: `openedAt` 13:01. Leases are refused from now on. Payments wait as DUE or FAILED_RETRYABLE. |
| 13:01 to 13:06 | us-east-1 runs path checks. They fail, so no payment is spent as a probe. The path checks of us-west-2 keep passing: it is able. |
| 13:06:00 | The us-west-2 Sweeper runs. us-east-1 has been open for five minutes and us-west-2 is able. It adopts W and up to 59 others: owner us-west-2, dispatch requested. |
| 13:06:02 | W flows through the us-west-2 Pipe and queue. Its Submitter leases W from the us-west-2 API and sends the stored bytes with the same key. OPF answers 202. |
| 13:06:03 | Outcome ACCEPTED, PayX is told, marked reported. |
| 13:10:00 | Payment X arrives in us-east-1, which still answers 202. Its lease is refused in us-east-1. |
| 13:11:00 | The next us-west-2 sweep adopts X. It is sent a few seconds later. |
| 13:40 | The route is repaired. us-east-1 has no payment waiting, so a passing path check closes its breaker. us-west-2 stops adopting. |

### 8.10 Failure analysis

Severity is the effect on a payment if the response did not exist: **High** = money could move twice or a payment could be lost; **Medium** = delay; **Low** = none.

| # | Component | Failure | Detected by | Response | Severity |
|---|---|---|---|---|---|
| 1 | Wire II API | Down in the region that serves the PayX calls | Routing health check | Routing moves the PayX calls to the other region. If the region then proves nothing for five minutes it is out, and the other region's Sweeper adopts its unfinished payments (8.4) | Medium |
| 2 | Wire II API | Stops after the write, before the 202 | PayX timeout | PayX re-calls; replay response | Low |
| 3 | Transaction table | Cannot be written | 5xx rate; health check | API answers 5xx; PayX re-calls; nothing is stored | Medium |
| 4 | Pipe | Stopped, or not delivering to SQS | Pipe failures; records the Sweeper had to re-dispatch | Pipe retries and its DLQ; the Sweeper re-dispatches after the stall threshold | Medium |
| 5 | Submit queue | Message lost or queue purged | Stalled-record alarm | Sweeper re-dispatches | Low |
| 6 | Submitter | Stops mid-attempt, after OPF accepted | Lease expiry | Redelivery or Sweeper; same payload, same key | High |
| 7 | Submitter | Cannot run: defect, bad deploy, missing permission | Message in the DLQ; stalled records | Record PARKED with `MAX_RECEIVES`; operator replay after the fix | Medium |
| 8 | Submitter | Cannot reach OPF from one region | That region's breaker opens, on five different payments or three path checks; alarm "region not able" | The region keeps accepting. After five minutes the other region, if it is able, adopts and sends (8.9). Routing does not have to move | Medium |
| 9 | Turing | Down | Breaker for Turing | Payments wait; probe; drain | Medium |
| 10 | OPF | 5xx or timeout on some calls | Error rate | Redelivery with backoff; same key | High |
| 11 | OPF | Down for hours | Breaker open | Wait in the table; path check, probe, drain (8.7). No takeover while both regions' breakers are open | Medium |
| 12 | OPF | Rejects our credentials or set-up (401 / 403) | Same answer on several payments | Breaker opens and pages; no payment is failed | Medium |
| 13 | OPF | Accepts a repeat as a new payment | Reconciliation; OPF duplicate checks | None beyond same key, same bytes and the deadline. Must be ruled out with FIS (OQ-A, OQ-K) | High |
| 14 | PayX status API | Unreachable | Terminal-not-reported alarm | Message redelivered; the Sweeper dispatches report-only attempts until PayX is told. The record stays terminal, never PARKED | Medium |
| 15 | Sweeper | Not running in a region | No-sweep alarm | The normal path still works; lost messages wait, and that region runs no path check, probe, drain, parking or adoption. If the region proves nothing for five minutes it counts as out, and the other region's Sweeper adopts its unfinished payments | Medium |
| 16 | Region | Lost with payments in flight | Routing health check; the region proves nothing | Routing moves the PayX calls. After five minutes the region is out and the other region's Sweeper adopts its unfinished payments (8.4) | Medium |
| 17 | Region | Record not replicated before the loss | PayX SLA alert | PayX replays once with the same UETR, and the region that takes the call creates the record again (8.4). Without a replay it waits for the region. Removed by strong consistency (8.4) | High |
| 18 | Deploy | The conversion changes while payments wait | Contract test | The stored payload is re-sent, never rebuilt | High |
| 19 | Region | The standby's route to OPF is broken | Its path checks fail and its breaker opens after three; alarm "region not able" | Repair it while the other region still sends. Without the path check nobody would know until the failover | Medium |
| 20 | OPF | The path check passes but OPF itself is failing | Breaker open on real failures in one region, then in the other | The first region opens on real failures. The other looks able, takes over after five minutes, and opens after five different failures of its own. Cost: one moved batch and a few unanswered attempts. No flapping, because neither region is able | Medium |
| 21 | Volume | Very low volume: five different payments never fail | Path checks failing | The path check opens the breaker after three failed checks, about three minutes after the route breaks | Medium |
| 22 | Tables | Replication between the regions stops while both are healthy | Replication latency alarm; takeover alarm | After five minutes one region treats the other as out and may re-send the few payments that were in flight. The same key and bytes at OPF are the backstop (OQ-A). Strong consistency would close the window (8.4) | High |

## 9. Consequences

**Positive**

- The PayX call returns in milliseconds; Turing and OPF latency no longer reach PayX.
- Two microservices and a small Sweeper Lambda, each scaled and deployed on its own. The Wire II API is the only writer of a payment's request and outcome; the Sweeper writes only who sends a payment next, through conditional updates; the Submitter has no database access.
- Retry, backoff, parking and replay come from SQS (redelivery, DLQ) rather than custom scheduling code.
- Dispatch has no dual-write gap: messages are produced from committed writes.
- The component that holds clear-text BANs has no data-store access, and the queue never carries payment data.
- Every failure mode resolves by re-sending the same stored payload with the same key.
- Wire II owns recovery end to end; it does not depend on PayX replaying an accepted call.
- A region that cannot reach OPF is taken over for sending automatically, without a routing switch and without a person (8.4).

**Negative**

- More managed resources per region: a Pipe, a queue, a DLQ and the Pipe's own DLQ.
- Each submission step makes an internal call to the API, so the Submitter depends on the API being up in its region.
- PayX must handle an asynchronous outcome (202 now, status later) — to confirm with PayX (OQ-G).
- The Sweeper Lambda and its sparse index are custom logic, kept small. Two deployables know the shape of the record, the API and the Sweeper; the Sweeper's attributes change only by adding, and a contract test guards them.
- A payment accepted just before a regional loss and not yet replicated needs the one PayX replay (OQ-D); without it, it waits until that region's data returns.
- One more small Global Table (the Breaker table) and one Lambda with a schedule per region.
- A regional takeover of sending starts five minutes after a region stops proving its route (OQ-Q).

**Risks**

| Risk | Impact | Mitigation |
|---|---|---|
| FIS does not honour the UETR idempotency key as assumed (short retention, header participation, non-exact comparison) | Duplicate wire on replay | Close OQ-A before go-live. If idempotency cannot be guaranteed, query OPF status by UETR before any re-POST (OQ-B). |
| A non-deterministic field slips into the payload | 422 on replay → a false `.WIRECLEARING.STOP` on a wire OPF accepted | Deterministic-generation rule, plus a contract test that builds the same payment twice and compares the bytes. |
| Both regions serve traffic at once | Concurrent submissions of one UETR | Active / passive routing of the PayX calls; region-scoped Pipes; the OPF idempotency key is the backstop. |
| EventBridge Pipes not supported by the pipeline | Dispatch cannot be deployed as designed | Replace the Pipe with a small stream-triggered Lambda that applies the same filter and sends to SQS (OQ-H). |
| us-west-2 not allow-listed at FIS, or credentials missing there | Failover fails at the first OPF call | Prerequisites checklist and a failover game day. The standby's path checks fail and raise an alarm before any takeover is needed. |
| A payment accepted just before a regional loss had not replicated | It is invisible to us-west-2 until us-east-1 returns | PayX SLA alert and one replay with the same UETR (OQ-D, deterministic payload); keep replication latency alarmed; strong consistency removes the window (OQ-I). |
| The Sweeper stops running in a region | Lost messages are not re-driven; that region runs no path check, probe, drain, parking or adoption | Alarm when no sweep has run in a region for five minutes. A region that proves nothing for five minutes counts as out, and the other region's Sweeper adopts its unfinished records. |
| A prolonged OPF or Turing outage | Messages accumulate, then move to the DLQ | Circuit breaker at the lease (8.7): payments wait in the table without using receives, one probe at a time tests recovery, and the backlog drains at a set rate. No message reaches the DLQ because of an outage, so no redrive is needed afterwards. |
| Replication between the regions stops while both are healthy | After five minutes one region treats the other as out and may re-send the few payments that were in flight | Replication latency alarm and takeover alarm; the same key and bytes at OPF are the backstop (OQ-A); strong consistency closes the window (OQ-I). |
| FIS has no call that can serve as a path check | With a token request as the fallback, the check proves the path and the credentials, not the OPF application. In an OPF application outage the standby adopts one or two batches before its own breaker opens | Ask FIS for a heartbeat or status URL, or a status inquiry by UETR (OQ-M, OQ-B). Nothing is lost or sent twice in the meantime. |
| The pipeline cannot limit the Sweeper's update to named attributes | The Sweeper's role is wider than its write contract | Every write is conditional; contract test in the Sweeper's pipeline; confirm with the platform (OQ-N). |

## 10. Technical debt incurred

Transitional. The V11 JSON generation, the Submitter, the dispatch resources, the Sweeper Lambda, the Breaker table, and the Global Table and failover configuration exist to satisfy OPF's interim API and are retired at the full OPF exit (Phase 2). pain.008 is not covered.

## 11. Implementation detail — pain.001 v9 to V11

**Detail (migrated from ADR-0003 §4d)**

Item 10 in §4b generates the pain.001 message submitted to OPF. An existing converter in `PmtInfrEngn-wire-ii-intg` already maps the PayX payment object to pain.001.001.09 and is the reference implementation; V11 is an increment on it, not a rewrite.

**Existing v9 converter** (package `…wireiiintg.converter.painv9`, ~820 lines): `PaymentInitiationV9Converter` implements Spring's `Converter<SDPPayload, InitiatePain001V09Payments>` and delegates to `GroupHeaderConversionUtilV9` for the group header and `PaymentInfoUtilV9` for the payment instruction, with party mapping in `model.v9` helpers (`Debtor`, `Creditor`, `UltimateDebtor`, `UltimateCreditor`). Source object is `SDPPayload`.

**Structural mapping — v9 → V11**

The document shape is unchanged: `Document → CstmrCdtTrfInitn → GrpHdr + PmtInf[] → CdtTrfTxInf[]`. Only the versioned type names change.

| Element | v9 type (existing code) | V11 type (per the FIS specification) | Change |
|---|---|---|---|
| Message root | `CustomerCreditTransferInitiationV09` | `CustomerCreditTransferInitiationV11` | Renamed; adds a message-level `supplementaryData` |
| Group header | `GroupHeader85` | `GroupHeader95` | Renamed; adds `initiationSource` (`PaymentInitiationSource1`) |
| Payment instruction | `PaymentInstruction30` | `PaymentInstruction40` | Renamed; adds `requestedAdviceType`, `instructionForDebtorAgent`, `chargesAccount`, `chargesAccountAgent` |
| Credit transfer transaction | `CreditTransferTransaction34` | `CreditTransferTransaction54` | Renamed; adds a third intermediary agent (+account), `instructionForDebtorAgent`, `mandateRelatedInformation`, `chequeInstruction` |
| Instruction for creditor agent | `InstructionForCreditorAgent1` | `InstructionForCreditorAgent3` | Renamed; code + `instructionInformation` (140 char) |
| Remittance information | `RemittanceInformation16` | `RemittanceInformation21` | Renamed; structured remittance is `StructuredRemittanceInformation17` |
| Cash account | `CashAccount38` | `CashAccount40` | Renamed |
| Payment identification | `PaymentIdentification6` | `PaymentIdentification6` | Unchanged — but see UETR below |
| Party / agent / address / amount / payment type | `PartyIdentification135`, `PostalAddress24`, `AmountType4Choice`, `PaymentTypeInformation26`, `ExchangeRate1`, `DateAndDateTime2Choice`, and the `BranchAndFinancialInstitution` agent type (suffix 6) | Same type names in the FIS specification | No change — existing mapping code carries over |

**What actually needs work**

- **Serialization format, not just the schema.** The v9 converter produces JAXB/XML — it marshals supplementary data with a `Marshaller` and builds a DOM `Element`, and uses `XMLGregorianCalendar` for dates. The OPF endpoint consumes JSON (Capability 2). This is the largest single change and is easy to underestimate: supplementary data must become a JSON object rather than a marshalled XML envelope, and dates become ISO-8601 strings rather than `XMLGregorianCalendar`.
- **Business Application Header is new.** The V11 payload is `businessApplicationHeader` + `businessDocument`. The v9 converter produces only the document, so the BAH (`from`, `to`, `businessMessageIdentifier`, `messageDefinitionIdentifier`, `creationDate` — all mandatory) is net-new mapping with no v9 equivalent.
- **UETR.** `PaymentIdentification6` carries a UUID-v4-patterned UETR that the v9 code does not populate. The UETR is generated upstream by the Payment Network Gateway, arrives with the payment and is used as the idempotency key; Wire II populates `PaymentIdentification6` with it, never generates one, and persists it on the transaction record. Whether OPF keeps the supplied UETR is pending with FIS.
- **Supplementary data envelope changes shape.** v9 wraps Dovetail `ClientData/ClientProperty` as marshalled XML under `SupplementaryDataEnvelope1`. V11 uses per-context supplementary-data types — one per containing element, e.g. a variant specific to `CreditTransferTransaction54` — each with an open `envelope`. Confirm with FIS what OPF expects inside the envelope — the existing FX-debit-currency, FX-debit-amount, initiator and approver properties must land somewhere equivalent.
- **Enumerations become plain strings.** v9 uses generated enums (`PaymentMethod3Code.TRF`, `ChargeBearerType1Code`, `Priority2Code.HIGH`, `ExchangeRateType1Code`). In the V11 JSON specification `paymentMethod` and `chargeBearer` are typed `string` with documented allowed values, so validation moves from the compiler to explicit code.
- **New optional fields are decisions, not defaults.** `initiationSource`, `requestedAdviceType`, `intermediaryAgent3`, `chargesAccount`: decide explicitly whether to populate each, and record why. Note the existing code already handles only two intermediary agents.
- **Carry over the known workarounds deliberately.** The v9 code contains behaviour that exists for real downstream reasons and must be consciously re-evaluated, not silently dropped: the `feature.blank.structured.address.enabled` structured-address policy (a PIE defect workaround, on in dev/qa, off in prod); TMOBS/OBS sender-specific sanitisation of bank-to-bank text; and 140-character chunking of bank-to-bank instructions with separators.
- **Reuse the shape, not the classes.** Keep the converter/util/party-helper split and add a parallel `painv11` package plus `model.v11` helpers rather than parameterising the v9 classes — the two must coexist while both versions are in play.

**`[[ WARNING PANEL STARTS HERE ]]`**

The FIS "V11" is a canonical JSON representation, not the ISO 20022 XML schema. Several types the ISO pain.001.001.11 XSD renames — `PostalAddress27`, `PartyIdentification272`, `RemittanceInformation22`, and the agent type at suffix 8 — do not appear in the FIS specification, which retains the v9-era `PostalAddress24`, `PartyIdentification135`, the agent type at suffix 6, and uses `RemittanceInformation21`. Do not map against the published ISO V11 XSD — map against the FIS specification, or the converter will emit types OPF does not accept. This also means the change is smaller than a true ISO v9→V11 migration, since most party and address mapping carries over unchanged. Confirm before build (OQ20).

**`[[ WARNING PANEL ENDS HERE ]]`**

## 12. Open questions

- **OQ-A (new, blocks go-live).** Confirm with FIS how OPF applies the idempotency key: retention window, whether the payload comparison is byte-exact, and whether the `uuid` header takes part. This design re-sends the stored payload with the stored `uuid`. Also: what OPF returns for a repeat that arrives while the first request is still in progress.
- **OQ-B (new).** Does OPF offer a status inquiry by UETR? It is needed if OQ-A cannot be guaranteed, it helps with records parked as `OUTCOME_UNKNOWN`, and it would also serve as the path check (OQ-M).
- **OQ-C (new).** Confirm that the PayX payment object carries a stable creation timestamp for the BAH creation date and the group-header creation date-time.
- **OQ-D (new).** Confirm PayX's behaviour around the call: it retries when no 202 is received, and it alerts and replays once with the same UETR when `.WIRECLEARING` misses its SLA. Agree the SLA value.
- **OQ-E (new).** The regional routing mechanism and control (for example Route 53 ARC): how the PayX calls are moved to the other region, automatically on the API health check or by runbook. This only concerns where PayX calls land. Wire II does not read the control, and the regional takeover of sending does not depend on it (8.4).
- **OQ-F (new).** Retention (TTL) for transaction records and stored payloads, per data classification.
- **OQ-G (new).** Confirm with PayX that Wire II can return 202 and report the outcome later through the PayX status API, and whether PayX should record an intermediate stage on the 202.
- **OQ-H (new).** Confirm that EventBridge Pipes (DynamoDB stream source, SQS target, event filtering) is supported by OnePipeline / Bogie; otherwise use a stream-triggered Lambda forwarder.
- **OQ-I (new).** Is DynamoDB multi-Region strong consistency, with a witness region, available through the provisioning template, with customer-managed keys? It closes the replication-lag window (8.4).
- **OQ-J (new).** OPF's rate limit and any planned maintenance windows, to set the breaker threshold, the probe interval and the drain rate (8.7).
- **OQ-K (new).** If a repeat were ever accepted as a second payment, would OPF's own duplicate checks stop it?
- **OQ-L (new).** The value of the submit deadline: FIS's key retention, the execution cut-off, and the longest wait product allows (8.8).
- **OQ-M (new).** Which FIS call can serve as the path check: authenticated, cheap and creating no payment? Preferred: a heartbeat or status URL from FIS, if one exists; the FIS specification available today exposes only the payment POST. Fallback: a token request to the FIS gateway, the same host as the payment API, which proves the region's network path and credentials but not the OPF application behind the gateway (8.7). A status inquiry by UETR would also serve (OQ-B).
- **OQ-N (new).** Can the deployment pipeline limit a Lambda's DynamoDB update to named attributes? The Sweeper's narrow access relies on it (8.3, 8.5).
- **OQ-O (new).** The expected number of wires per hour, to set "60 per sweep" for the drain and for adoption.
- **OQ-P (new).** Does the Turing breaker get its own path check (8.7)?
- **OQ-Q (new).** Is a five-minute wait before a regional takeover of sending acceptable against the PayX status deadline (8.4)?
- **OQ20 (existing).** The FIS "V11" is a canonical JSON representation, not the ISO 20022 V11 XSD. Map against the FIS specification, or the converter will emit types OPF does not accept.
- **Existing.** Confirm which Turing token types apply across all Phase 1 correspondent-bank destinations (BAN alone may not cover UKBAN or NONUSBAN destinations).
- **Existing.** Confirm that `.WIRECLEARING` / `.WIRECLEARING.STOP` do not collide with existing PayX stage names on other rails.
- **Existing.** Ordering guarantees across concurrent payments are not specified; a standard SQS queue does not preserve order.
- **Resolved by this revision.** Behaviour when Wire II fails between submitting to OPF and storing the response is now defined: the stored payload, the lease, SQS redelivery and the recovery Sweeper re-drive the same submission safely, without relying on PayX.
