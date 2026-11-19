# Capability 3 (Submit Request to OPF API) - ADR-0003: full page, ready to paste

Everything below the line "PAGE CONTENT STARTS HERE" is the whole page. Paste it over the whole page body. Nothing from the old page has to be kept.

## The markers in the text

Every marker is in double square brackets, so a search for `[[` finds each one. After pasting, work through them and delete the marker text.

| Marker | What to do | How many |
|---|---|---|
| `[[ INFO PANEL STARTS HERE ]]` … `[[ INFO PANEL ENDS HERE ]]` | Insert an Info panel. Move everything between the two markers into it. Delete both marker lines. | 1 |
| `[[ WARNING PANEL STARTS HERE ]]` … `[[ WARNING PANEL ENDS HERE ]]` | Insert a Warning panel. Move everything between the two markers into it. Delete both marker lines. | 1 |
| `[[LINK: text]]` | Replace the marker with the text inside it, and make that text a link to the page in the table below. | 4 |

## The links

| Link text | Points to | How many |
|---|---|---|
| ADR-0003 | The ADR-0003 main page | 1 |
| Capability 10 (Wire II — pain.001 generation) | The page "Capability 10 (Wire II — pain.001 generation) - ADR-0003" | 2 |
| Capability 2 (OPF API integration) | The page "Capability 2 (OPF API integration) - ADR-0003" | 1 |

## Section references

References to Capability 10 use the numbering of that page as it is today: 8.3 Sweeper, 8.4 regional failover, 8.5 security, 8.7 circuit breaker and path check.

---

**PAGE CONTENT STARTS HERE**

---
**`[[ INFO PANEL STARTS HERE ]]`**

**Status:** Proposed — 2026-11-19. **Scope:** Phase 1 (transitional; retired at the full OPF exit in Phase 2). **ASV:** ASVWIREII. **Parent decision:** [[LINK: ADR-0003]].

Capability 3 covers dispatch and submission: the managed dispatch (DynamoDB stream, EventBridge Pipe, SQS) and the **Wire II Submitter**, which detokenizes the BAN values, submits the stored pain.001 to OPF and reports the final status to PayX. It is the second half of one design; the first half, the **Wire II API** and the **Sweeper**, is [[LINK: Capability 10 (Wire II — pain.001 generation)]].

**`[[ INFO PANEL ENDS HERE ]]`**

## 1. Context and problem statement

After the Wire II API has stored a payment and answered 202, the pain.001 must be submitted to the FIS OPF API and the outcome returned to PayX. OPF is a third-party API whose latency and availability are outside Capital One's control, so submission runs in the background, can be repeated safely, and survives the loss of a region.

## 2. Assumptions

- The Wire II API has stored the built pain.001 V11 JSON, tokenized, and owns the transaction record (Capability 10).
- The connection to the OPF API (auth, CodeConnect, firewall, DevExchange registration) is established by [[LINK: Capability 2 (OPF API integration)]].
- BAN values are tokenized and are detokenized through the Turing API (Capital One Exchange asset 6809).
- EventBridge Pipes, DynamoDB Streams and SQS are supported by the pipeline (OQ-H). EventBridge Pipes is confirmed as an approved service.
- Turing and the PayX status API are reachable from both us-east-1 and us-west-2.
- A call to FIS that creates no payment is available for the path check, or a token request to the FIS gateway is used instead (Capability 10, OQ-M).

## 3. Constraints

- The Submitter has no database access. It works on a record only through the Wire II API's internal endpoints.
- Each submission carries the required `uuid` and `idempotency-key` headers, and re-sends the stored payload byte for byte.
- Detokenized BAN values are held in memory for one attempt and are never logged or stored.
- The queue message carries the UETR only.

## 4. Decision

**Dispatch — managed, no code**

The table's change stream feeds an EventBridge Pipe in each region. The Pipe passes only records whose owning region is that region and whose `dispatchRequested` flag is true, and sends an SQS message carrying the UETR alone. No payload or personal data enters the queue. Because the message is produced from the committed write, there is no dual-write gap between storing a payment and queueing it. The submit queue also carries path-check messages, which the Sweeper puts there directly; they hold no payment data either. The Breaker table has no Pipe.

**Submission path — Wire II Submitter**

1. Receive the SQS message (UETR).
2. Lease the record through the API (`POST /internal/wires/{uetr}/leases`). The API moves it to SUBMITTING, clears `dispatchRequested`, and returns the stored payload and `uuid`. A refused lease (not owner, lease held, already reported, deadline passed, or circuit breaker open) means the message is deleted or left to reappear.
3. Detokenize the BAN value(s) through Turing (`POST /enterprise/detokenize`, OAuth token from the DevExchange Gateway, type = BAN, GLOBALBAN, UKBAN or NONUSBAN). The clear-text value is held in memory for this attempt only.
4. Submit the stored payload to the OPF API (`POST /customer-credit-transfer-initiations`) with the stored `uuid` and `idempotency-key` = UETR.
5. Record the outcome through the API (`PUT /internal/wires/{uetr}/outcome`): ACCEPTED, REJECTED or FAILED_RETRYABLE. With FAILED_RETRYABLE the Submitter states the cause and whether the request left Capital One, so the API can count failures for the breaker and keep `unansweredAttempts` right.
6. Finish: for a terminal outcome, report the final status through the PayX status API, mark it reported (`PATCH /internal/wires/{uetr}`) and delete the message. For a retryable outcome, leave the message and set its visibility to the next backoff; SQS redelivers it. After the maximum number of receives the message moves to the DLQ and the record is marked PARKED, with an alert. A timeout or a 5xx is never retried at once inside the attempt, because OPF may still be working on the first request.

**Path check — the second kind of message**

The Submitter also handles path-check messages. The Sweeper puts one on the region's submit queue when the region has proven nothing for a minute (Capability 10, 8.7).

1. Receive the path-check message: a small JSON body naming the dependency, for example `{"check":"OPF"}`, with its send time. A payment message is the bare UETR, so the two cannot be confused.
2. Call FIS once: an authenticated call that creates no payment (Capability 10, OQ-M). No lease is taken and nothing is detokenized, because no payment is sent.
3. Report pass or fail to the Wire II API of its region (`POST /internal/breakers/{dependency}/checks`).
4. Delete the message, whatever the result. A check older than two minutes is deleted unread. A check is never retried: the next minute brings a new one.

The check travels the same road as a payment: queue, Submitter, proxy, partner gateway, FIS, API, table. That is why a passing check is evidence that this region can send.

**Runtime.** The Submitter runs as an ECS service that scales on queue depth. If it is run as a Lambda with an SQS trigger instead, it is written in Go: Java is discouraged for Lambdas because of cold starts. The contract is the same.

### 4.1 OPF response mapping

| OPF response | Wire II status | Reported to PayX | PayX stage |
|---|---|---|---|
| 202 Accepted | ACCEPTED | Accepted | `.WIRECLEARING` |
| 400 Bad Request (validation) | REJECTED | Rejected | `.WIRECLEARING.STOP` |
| 422 Idempotent replay, different payload | REJECTED (alert) | Rejected — duplicate UETR with different content | `.WIRECLEARING.STOP` |
| 401 / 403 | FAILED_RETRYABLE (alert); counts toward the circuit breaker | Nothing yet — redelivered after backoff, or held while the breaker is open | none until a terminal outcome |
| 500 / 503 / timeout | FAILED_RETRYABLE; counts toward the circuit breaker | Nothing yet — redelivered after backoff, or held while the breaker is open | none until a terminal outcome |
| Maximum receives reached | PARKED (alert), reason `MAX_RECEIVES` | Nothing yet — an operator replay revives it | none until a terminal outcome |

**`[[ WARNING PANEL STARTS HERE ]]`**

A 422 should be impossible on a genuine replay, because the stored payload is re-sent byte for byte and a rebuilt payload is deterministic. Any 422 therefore signals a defect or a reused UETR, and raises an alert.

**`[[ WARNING PANEL ENDS HERE ]]`**

### 4.2 Turing detokenize response handling

Read against Turing's documented error codes for 203, rather than one blanket policy. "Retryable" means the message is redelivered after backoff.

| Turing response | Outcome | Treatment |
|---|---|---|
| 200 | Success — clear-text value returned | Proceed to the OPF submission |
| 203, code 105 (ID parameter length exceeds limit) | Malformed request construction | Block submission; non-retryable; alert — this is a Wire II defect, not a transient issue |
| 203, code 106 (VALUE parameter invalid/out of range) | The stored token value itself is malformed | Block submission; non-retryable; alert — indicates a bad token upstream of this call |
| 203, code 108 (token parameter invalid) | Not a valid Turing token | Block submission; non-retryable; alert — cannot detokenize what isn't a token |
| 203, code 109 (lookup database internal error) | Transient backend lookup failure | Retryable — same bucket as Turing/OPF 500; counts toward the Turing circuit breaker |
| 400 | Bad request (malformed call) | Block submission; non-retryable; alert — same bucket as OPF 400 |
| 401 / 403 | Unauthorized / client not entitled | Technical failure; alert — same bucket as OPF 401/403; counts toward the Turing circuit breaker |
| 500 | Turing backend error | Technical failure, retryable — same bucket as OPF 500/503; counts toward the Turing circuit breaker |

## 5. Dispatch and retry

| Layer | Owner | Trigger | Proposed starting values |
|---|---|---|---|
| Pipe dispatch | EventBridge Pipe | A write with `dispatchRequested` = true and owner = this region | Pipe retries and its own DLQ for undeliverable events; anything lost is re-dispatched by the recovery Sweeper. |
| Redelivery with backoff | SQS + Wire II Submitter | Retryable outcome (5xx, timeout, 401 / 403, Turing 203 code 109) while the breaker is closed | Visibility set from the receive count: 30 s, 1 min, 2 min, 5 min, 10 min, then 15 min. |
| Circuit breaker | Wire II API | Several different payments failing in a row, or three failed path checks | Opens after 5 different payments or 3 failed path checks. A probe goes only after a path check has passed: after 1, 2, 4, 8 minutes, then every 15. Drains 60 per sweep (Capability 10, 8.7). |
| Path check | Sweeper Lambda + Wire II Submitter | A region has proven nothing for a minute | One message a minute; tried once; deleted unread after 2 minutes; three failures in a row open the breaker (Capability 10, 8.7). |
| Lease expiry | SQS | A Submitter task stopped mid-attempt | Visibility timeout and lease TTL both exceed the worst-case Turing + OPF attempt. |
| Parking | SQS DLQ; Sweeper Lambda at the deadline | Maximum receives reached (about 2 hours of retries), or the submit deadline | Alarm when a message arrives in the DLQ. At maximum receives the API marks the record PARKED with `MAX_RECEIVES`. At the submit deadline the Sweeper marks it PARKED with `NOT_SENT` or `OUTCOME_UNKNOWN`. |
| Recovery sweep | Sweeper Lambda (each region) | Stalled or unreported record; unfinished record of a region that is out | Every minute; stall threshold about 20 min; takeover when the other region has been out for 5 min and this region is able, 60 records per sweep (Capability 10, 8.3 and 8.4). |
| Operator replay | Operations | Investigated PARKED record | `POST /internal/wires/{uetr}/dispatches`. The DLQ message itself is not needed: it holds only the UETR and expires by itself (DLQ retention 24 hours, working queue 12 hours). |
| PayX call retry | PayX | No 202 received (timeout or 5xx) | Per PayX policy; idempotent on the UETR. |

*The starting values are proposals, to be tuned in performance and resilience testing.*

**The DLQ and the Sweeper.** A queue message is only a pointer; the payment is in the table. SQS redelivery handles normal retries. The Sweeper acts only after the stall threshold, which is longer than the longest backoff, so the two never race. A message in the DLQ means a defect or a payment that failed for two hours while others succeeded; the record is PARKED and an operator replays the record, not the message.

**PARKED is for non-terminal records only.** If the report to PayX keeps failing for a record that is already ACCEPTED or REJECTED, the record stays terminal. Its message may reach the DLQ; the Sweeper keeps dispatching report-only attempts until PayX has been told.

## 6. Wire II Submitter runtime

- Consumer concurrency is the back-pressure limit on Turing and OPF; bursts wait in the queue.
- Scales on queue depth and the age of the oldest message, independently of the API.
- On deploy or scale-in, tasks stop receiving on SIGTERM and finish in-flight attempts within the stop timeout. An attempt that is cut off leaves its message to reappear and its lease to expire.
- Holds the only clear-text BAN, in memory, and has no DynamoDB permissions.
- Handles two kinds of message from the same queue: a payment (the bare UETR, from the Pipe) and a path check (a small JSON body, from the Sweeper). A Submitter that fails on a path check lets the message reach the DLQ like any other; the DLQ alarm then points at a broken Submitter, and the region proves nothing and counts as out.

## 7. Security and least privilege

| Identity | Allowed | Not allowed |
|---|---|---|
| EventBridge Pipe | Read the table stream; send to the submit queue | Anything else |
| Wire II Submitter | Receive / delete / change visibility on its queue; Turing; OPF; PayX status API; the Wire II internal endpoints | Any DynamoDB access |
| Sweeper (Lambda) | Send path-check messages to the submit queue of its region. Its table access is described on Capability 10, 8.5. | Receive, delete or change visibility on the queue; Turing; OPF; PayX status API; the Wire II internal endpoints |

Queues and DLQs are encrypted with a customer-managed KMS key, and messages carry the UETR only. Logs carry the UETR for correlation, and never a clear-text BAN or a full request or response body.

## 8. Consequences

- Retry, backoff and parking come from SQS; the circuit breaker keeps a vendor outage out of the DLQ.
- The component that holds clear-text BANs has no data-store access.
- Each submission step makes an internal call to the API, so the Submitter depends on the API being up in its region.
- More managed resources per region: a Pipe, a queue, a DLQ and the Pipe's own DLQ.
- The Submitter also proves, once a minute, that its region can reach OPF. That is what lets the other region take over the sending without a routing switch.

## 9. Technical debt incurred

Transitional. The Submitter and the dispatch resources exist to satisfy OPF's interim API and are retired at the full OPF exit (Phase 2).

## 10. Open questions

Tracked on [[LINK: Capability 10 (Wire II — pain.001 generation)]], section 12: OQ-A, OQ-B, OQ-H, OQ-J, OQ-K, OQ-M and OQ-P for this page.
