# Wire II stories after the asynchronous design: what stays, what changes, what is new

The 22 issues under the epic were written for one synchronous service. The design is now asynchronous: the **Wire II API** stores the payment and answers 202, a managed dispatch (table stream, EventBridge Pipe, SQS) hands the work to the **Wire II Submitter**, a **Sweeper Lambda** is the safety net, and a circuit breaker in its own table protects against an OPF outage. This file says what to do with each existing issue and gives the new stories, ready to paste.

Titles of new and rewritten stories start with **[Cap 10]** or **[Cap 3]**. Everything is created under the Capability 10 epic for now. The tag lets you move the [Cap 3] stories in one step when the Capability 3 epic exists.

## 1. The result in numbers

| | Issues | Points |
|---|---|---|
| Done, nothing to do | 6 | already accepted |
| Keep as they are | 7 | 26 |
| Update (small change to text or criteria) | 8 | 32 |
| Rewrite (same key, new title and content) | 3 | 15 (were 9) |
| New, [Cap 10] | 8 | 36 |
| New, [Cap 3] | 6 | 24 |
| **Remaining work** | **32** | **133** |

**The planned end of the epic, 31 December 2026, no longer holds.** At about 24 points a sprint, 133 points need the current sprint and five more. The build finishes in sprint 27.1.3, about 18 February 2027, if the first sprint of the new year starts on 7 January. The stories that need the real FIS sandbox still wait for the FIS statement of work.

## 2. Sprint calendar

Two-week sprints, Thursday to Wednesday. The 27.1 dates are assumed from the pattern and are to be confirmed.

| Sprint | Dates | Notes |
|---|---|---|
| 26.4.4 | 12 Nov to 26 Nov | current; ends on Thanksgiving |
| 26.4.5 | 26 Nov to 10 Dec | |
| 26.4.6 | 10 Dec to 24 Dec | holiday-reduced, about 17 points |
| 27.1.1 | 7 Jan to 21 Jan | assumed |
| 27.1.2 | 21 Jan to 4 Feb | assumed |
| 27.1.3 | 4 Feb to 18 Feb | assumed |

Capacity: three developers, five points about a week each, about 24 points a sprint.

## 3. Sprint plan

V = Vasanth, T = Tejaswi, K = Kaushik P. Assignments are a suggestion. Tejaswi also carries stories of the OPF connection epic, which are not in this file.

| Sprint | Stories | Points |
|---|---|---|
| 26.4.4 (current) | 146193 design (5, V), 145515 field validation (5, K), 143562 XML to JSON (5, V), 146044 publish the mapper library (3, V), 144735 data standardisation (3, K), 146048 approvals follow-up (2, K) | 23 |
| 26.4.5 | 143560 wire the mapper in (3, V), 143564 header (3, V), 143578 intake API (5, V), 143580 transaction record (5, K), NEW tables for the asynchronous design (5, K), 145619 API pipeline green (3, T) | 24 |
| 26.4.6 | NEW internal API for the Submitter (5, V), NEW dispatch: Pipe, queue and DLQs (5, T), NEW Submitter service scaffold (5, T) | 15 |
| 27.1.1 | 143577 Submitter sends to OPF (5, V), NEW Turing detokenisation (3, K), NEW circuit breaker at the lease (5, K), NEW Submitter reports to PayX, retries and parking (5, T), NEW Sweeper Lambda (5, V) | 23 |
| 27.1.2 | NEW Sweeper path check and takeover (5, V), NEW Submitter path check (3, T), NEW operator replay and manual breaker control (3, K), NEW alarms (3, K), 143585 observability and end-to-end test against the stub (5, T), 145621 QA environment (5, K) | 24 |
| 27.1.3 | NEW second region and failover game day (5), 143579 clearing stages in QA (3), 145623 end-to-end test in UAT (5), 145622 FIS sample alignment (5), NEW FIS sandbox repeat tests (3), 145624 move to the Wire II ASV (3) | 24 |

One move on the board: 143580 sits in the current sprint today. With its new content it belongs in 26.4.5, after the tables story.

Order that must hold: tables before the transaction record and the intake API; internal API before the Submitter stories; dispatch before the Submitter can be tested end to end; circuit breaker before the Sweeper; both Sweeper stories before the failover game day. The last sprint holds everything that needs FIS or the new ASV, so it moves if they move.

## 4. Every existing issue and what to do with it

| Key | Title today | Status | Points | Decision | Capability |
|---|---|---|---|---|---|
| 143493 | Wire II API and Utility repo local project skeleton setup | Accepted | | Done | 10 |
| 144060 | Analyzing of converting existing Payment Object to v11 pain.001 xml | Accepted | | Done | 10 |
| 144081 | Registering Wire II API in developer exchange with an Open API spec | Accepted | | Done | 10 |
| 144884 | Implement wire-ii-pain001-mapper to convert the Payment Object to pain.001 V11 XML | Accepted | | Done | 10 |
| 145516 | Stand up a WireMock stub of the FIS OPF API for development | Accepted | | Done | 10 |
| 145620 | Provision the DynamoDB table and access for Wire Rail II | Accepted | | Done. What the new design adds is a new story | 10 |
| 144735 | Complete data standardization for Payx-Wire-Payment-Processor API attributes | In Review | 3 | Keep | 10 |
| 145515 | Add field-level validation to the Bank Wire Rail II API contract | In Progress | 5 | Keep | 10 |
| 146048 | Follow-up: Architect approval and external data share approval | In Progress | 2 | Keep | 10 |
| 146193 | Finalize the asynchronous design for Wire Rail II and update the ADRs for approval | In Progress | 5 | Keep. This file closes its last open criterion | 10 |
| 146044 | Fix the Jenkins pipeline so bank-wire-object-mapper publishes to Artifactory | Backlog | 3 | Keep | 10 |
| 145622 | Align the message with the FIS sample requests and the data-mapping elaboration | Backlog | 5 | Keep | 10 |
| 145624 | Move Wire Rail II from the PIE ASV to the Wire II ASV | Backlog | 3 | Keep | 10 |
| 143560 | Wire the pain.001 mapper into the Wire Rail II service as the generation step | Backlog | 3 | Update | 10 |
| 143562 | Implement XML-to-JSON conversion for the generated pain.001.001.11 message | Backlog | 5 | Update | 10 |
| 143564 | Implement Business Application Header (BAH) wrapping per the OPF API contract | Backlog | 3 | Update | 10 |
| 145619 | Merge the FIS client prototype into the scaffolded Wire Rail II service and get the pipeline green | Backlog | 5, becomes 3 | Update | 10 |
| 145621 | Add the QA environment to the Wire Rail II pipeline and deploy to QA | Backlog | 5 | Update | 10 |
| 143585 | Observability for Wire Rail II and automated end-to-end test against the stub | Backlog | 5 | Update | 10 |
| 145623 | End-to-end test in UAT: PayX integration service to Bank Wire Rail II to the OPF sandbox | Backlog | 5 | Update | 10 |
| 143578 | Implement the Bank Wire Rail II API response contract for the PayX integration service | Backlog | 3, becomes 5 | Rewrite | 10 |
| 143580 | Implement DynamoDB transaction record persistence for Wire II submissions | Backlog | 3, becomes 5 | Rewrite | 10 |
| 143577 | Implement direct submission to the OPF API | Backlog | 3, becomes 5 | Rewrite | 3 |
| 143579 | Verify in QA that the PayX integration service produces the clearing stages from the Bank Wire Rail II response | Backlog | 3 | Update | 3 |

One note on 145515, which is in progress and stays as it is: its last criterion says nothing is persisted on an invalid request. That still holds for a request the contract rejects. A request that passes the contract and then fails in the conversion is a different case and is covered by 143560 below.

## 5. Stories to update

### Update 143560, Wire the pain.001 mapper into the Wire Rail II service as the generation step

Points: 3 (unchanged). Title: unchanged.

Description, replace the last sentence ("Mapping failures return a distinguishable error to the caller with the field that caused them, and nothing is persisted or sent onward.") with:
A mapping failure returns a 400 that names the field. The payment is recorded as rejected so a repeat of the same call gets the same answer, and nothing is queued or sent to OPF. The generation is deterministic: the same payment always produces the same document, because the message is stored once and re-sent byte for byte on every attempt.

Acceptance criteria, replace with:

- [ ] The service calls the mapper library and produces a schema-valid pain.001.001.11 document for a GPN wire and a corporate wire sample
- [ ] UETR from the request is populated on the payment identification
- [ ] Supplementary data is carried into the V11 per-context envelopes per the mapping document
- [ ] A mapping or validation failure returns a 400 naming the field, the payment is recorded as rejected, and nothing is queued or sent to OPF
- [ ] Building the same payment twice gives byte-identical output; no value comes from the wall clock
- [ ] Unit tests cover the generation step with golden payment objects

### Update 143562, Implement XML-to-JSON conversion for the generated pain.001.001.11 message

Points: 5 (unchanged). Title and description: unchanged.

Acceptance criteria, add one line at the end:

- [ ] The conversion is deterministic: the same XML always gives byte-identical JSON, with a stable order of elements

### Update 143564, Implement Business Application Header (BAH) wrapping per the OPF API contract

Points: 3 (unchanged). Title: unchanged.

Description, add at the end:
The header is built from the payment, not from the moment of the call. The business message identifier is the UETR with the hyphens removed, and the creation date is the creation timestamp of the PayX payment. A header built again for the same payment is therefore identical, which the idempotent repeat at OPF depends on.

Acceptance criteria, replace with:

- [ ] businessApplicationHeader with from, to, businessMessageIdentifier, messageDefinitionIdentifier and creationDate is present on every message
- [ ] businessMessageIdentifier is the UETR with hyphens removed (32 characters), and the group-header message identifier uses the same value
- [ ] creationDate and the group-header creation date-time come from the PayX payment's creation timestamp, never from the wall clock
- [ ] from and to identifiers and the characterSet setting are configuration per environment, not code
- [ ] Schema validation covers the header
- [ ] Placeholder values are replaced by Discover's values from FIS without a code change

### Update 145619, Merge the FIS client prototype into the scaffolded Wire Rail II service and get the pipeline green

Points: 3 (was 5). The FIS client no longer belongs in this service; it moves to the Submitter, in the new story "[Cap 3] Scaffold the Wire II Submitter service with the FIS client and its pipeline".

Title, replace with: Get the scaffolded Wire II API building with the mapper library and the pipeline green

Description, replace with:
The Wire II API repository was generated by the DevNav Hub Fargate API template with the enterprise Java libraries, a pipeline file and dev-only environments. This story makes it a working base for the API: add the mapper library as a dependency from Artifactory, align the Java version with the library, replace plaintext placeholders with secrets references, write the first unit tests so the Sonar coverage gate passes, run the container locally, and complete the first successful pipeline run and deployment to the dev environment. The API calls neither OPF nor Turing, so no FIS client is added here.

Acceptance criteria, replace with:

- [ ] Mapper library is a dependency resolved from Artifactory, and the Java version is consistent across the service and the library
- [ ] No plaintext credentials remain; secrets are referenced through the secrets library
- [ ] Unit tests exist and the Sonar quality and coverage gates pass
- [ ] Pipeline builds, scans and deploys the service to the dev environment and the health check passes
- [ ] Service runs locally in a container with one command, against a local DynamoDB

### Update 145621, Add the QA environment to the Wire Rail II pipeline and deploy to QA

Points: 5 (unchanged).

Title, replace with: Add the QA environment to the Wire II pipelines and deploy the API, the Submitter and the Sweeper to QA

Description, add at the end:
The design now has three deployables: the Wire II API, the Wire II Submitter and the Sweeper Lambda, with the tables, the Pipe and the queues between them. All of them get a QA environment in this story, so the end-to-end test runs on the full path.

Acceptance criteria, replace with:

- [ ] QA environment blocks exist in the pipeline files of the API, the Submitter and the Sweeper, modeled on the reference service
- [ ] Tables, Pipe, submit queue and dead-letter queues exist in QA
- [ ] Secrets for QA are injected from Secrets Manager; no placeholders remain
- [ ] Rollback alarms and the on-call service are configured; the governance tracker is enabled
- [ ] Pipeline approval status is confirmed and all three are deployed to QA with passing health checks
- [ ] Outbound path to the FIS gateway is verified from the Submitter in the QA runtime

### Update 143585, Observability for Wire Rail II and automated end-to-end test against the stub

Points: 5 (unchanged). Title: unchanged.

Description, replace with:
Give the on-call engineer what they need before cut-off, for the asynchronous path. Structured JSON logs with correlation and trace ids in the API, the Submitter and the Sweeper, attributable to a UETR and propagated onto the OPF request headers. A metric for every status change of a payment, and metrics for conversion failures, OPF latency and OPF answers by code. Dashboards for the above. And an automated end-to-end test in the pipeline that runs the whole flow against the WireMock stub: call the API, receive 202, wait for the Submitter to send, and check the stored outcome and the status reported to PayX. The alarms of the breaker, the regions and the parked records are a separate story. The UAT test against the real sandbox is a separate story.

Acceptance criteria, replace with:

- [ ] Structured logs in the API, the Submitter and the Sweeper carry correlation id, trace id and UETR, with no account data
- [ ] A metric exists for every status change, tagged with the region, plus conversion failures, OPF latency and OPF answers by code
- [ ] Automated end-to-end test covers an accepted payment, a rejected payment and a retried payment against the stub, from the 202 to the status reported to PayX
- [ ] The test runs in the pipeline and fails the build when a step does not complete in the expected time
- [ ] Dashboards for the above are available in the team's monitoring tool

### Update 145623, End-to-end test in UAT: PayX integration service to Bank Wire Rail II to the OPF sandbox

Points: 5 (unchanged). Title: unchanged.

Description, replace the second sentence (it starts "With Wire Rail II deployed in QA") with:
With the Wire II API, the Submitter and the Sweeper deployed under their own ASV, the PayX integration service calling the API, and the OPF sandbox configured by FIS, run outbound GPN and corporate wires end to end: payment object from PayX, 202 from the Wire II API, dispatch to the Submitter, submission, 202 from OPF, the outcome on the record, the final status reported to PayX and the clearing stage produced.

Acceptance criteria, replace with:

- [ ] A GPN wire and a corporate wire submitted from PayX in UAT get 202 from the Wire II API, 202 from the OPF sandbox, and produce WIRECLEARING after the status report
- [ ] A payload OPF rejects produces WIRECLEARING.STOP with OPF's error recorded against the transaction
- [ ] A replayed UETR with the same payload gets the repeat answer and creates no duplicate at OPF
- [ ] A payment sent while the sandbox is unreachable waits and completes by itself when the sandbox returns
- [ ] Transaction records match the submitted and returned data for every test
- [ ] Evidence and the list of defects are recorded on the capability pages and reviewed with the tech lead and the PO

### Update 143579, Verify in QA that the PayX integration service produces the clearing stages from the Bank Wire Rail II response

Points: 3 (unchanged). This story belongs to Capability 3.

Title, replace with: [Cap 3] Verify in QA that PayX produces the clearing stages from the final status reported by Wire II

Description, replace with:
With the asynchronous design the clearing stage does not come from the response to the first call. The PayX integration service calls the Wire II API after OFAC and gets a 202. The stage is produced later, when the Wire II Submitter reports the final status through the PayX status API: WIRECLEARING on an accepted outcome and WIRECLEARING.STOP on a rejection. On a retryable technical failure nothing is reported; Wire II retries by itself, and PayX replays the call once with the same UETR only if the stage misses its service level. This is a joint test with the PayX integration team in QA against the WireMock stub, before the real sandbox is available. The stage names and the status-report contract are confirmed with that team first.

Acceptance criteria, replace with:

- [ ] Stage names and the status-report contract for accepted and rejected outcomes are confirmed with the PayX integration team and recorded on the capability page
- [ ] A test payment on an outbound GPN wire in QA gets 202 from the Wire II API and produces WIRECLEARING after a stubbed OPF 202
- [ ] A stubbed 400 or 422 produces WIRECLEARING.STOP
- [ ] A stubbed 500 or timeout produces no stage; Wire II retries by itself and the stage appears once the stub answers 202
- [ ] A replay of the call with the same UETR gets the repeat answer and produces no second stage
- [ ] Results recorded with the request and response pairs

## 6. Stories to rewrite (same key, new title and content)

### Rewrite 143578

**Story: [Cap 10] Accept a payment idempotently: POST /wires answers 202 and GET /wires/{uetr} returns the status**
Points: 5 (was 3)

Description

As the PayX integration service, I want Wire II to accept a payment, store it and answer at once, so that my call never waits for Turing or OPF and a repeat of the call is always safe.

The API builds the pain.001.001.11 JSON in memory with the account numbers still tokenised, then stores the payment with one conditional write keyed on the UETR: the payment hash, the built message, the request header value, status DUE, the owning region and the request for dispatch. A UETR that already exists is a repeat: the same content in flight gets 202 again, a finished payment gets 200 with the stored outcome, and different content gets 409. A build or validation failure gets 400 and is recorded as rejected. GET returns the current status, so PayX can check a payment whose status report it did not receive. The contract is agreed with the PayX integration team and published in the registered API specification.

Acceptance criteria

- [ ] New UETR: the payment is stored with status DUE and the call returns 202 with a Location header
- [ ] Same UETR and same content: 202 while in flight, 200 with the stored outcome when finished, and OPF is never called again
- [ ] Same UETR and different content: 409, record unchanged, alert raised
- [ ] Two first calls at the same moment store one record; the other gets the repeat answer
- [ ] The Idempotency-Key header must equal the UETR in the body; a mismatch is a 400
- [ ] GET /wires/{uetr} returns the current status and, when finished, the outcome
- [ ] Contract is agreed with the PayX integration team and published on Exchange; contract tests cover every answer

### Rewrite 143580

**Story: [Cap 10] Transaction record: the data access layer of the Wire II API with conditional writes**
Points: 5 (was 3)

Description

As the Wire II developer, I want one data access layer for the transaction record in which every write is conditional, so that a payment has exactly one record, two writers can never overwrite each other, and the stored message is never changed after it was written.

Implement the record and its access in the Wire II API: create only if the UETR is new, read by UETR, and update only when the version is unchanged. The record holds the status, the payment hash, the stored message with tokens only, the request header value, the owning region, the dispatch flag, the keys of the pending index, the lease, the attempt counters, the submit deadline, the parked reason, OPF's last response, the reported flag, audit timestamps, the version and the expiry. Status values are DUE, SUBMITTING, ACCEPTED, REJECTED, FAILED_RETRYABLE and PARKED. A record leaves the pending index when it is finished and reported.

Acceptance criteria

- [ ] Create is a conditional put: a second create for the same UETR fails and returns the existing record
- [ ] Every update checks the version and raises it; a stale update is refused
- [ ] The stored message and the payment hash cannot be changed after the first write
- [ ] The keys of the pending index are present while the record is unfinished or unreported, and removed when it is finished and reported
- [ ] No clear-text account number is ever written; the record holds tokens only
- [ ] Table design is reviewed with the team
- [ ] Integration tests run against a local DynamoDB and cover create, repeat, concurrent update and removal from the index

### Rewrite 143577

**Story: [Cap 3] Wire II Submitter: lease the payment, send the stored pain.001 to the OPF API and record the outcome**
Points: 5 (was 3)

Description

As the Wire II developer, I want the Submitter to take a UETR from the queue, claim the payment through the Wire II API, send the stored message to OPF and record what OPF answered, so that submission runs in the background and every attempt sends exactly the same bytes.

The Submitter has no database access. For each message it asks the API for a lease, receives the stored message and header value, replaces the tokens through Turing in memory (separate story), posts to the customer credit transfer initiations endpoint with the uuid header and the idempotency key set to the UETR, and reports the result to the API: accepted on 202, rejected on 400 or 422, retryable on 401, 403, 5xx or timeout. With a retryable result it states the cause and whether the request left Capital One. A timeout or a 5xx is never retried inside the attempt, because OPF may still be working on the first request.

Acceptance criteria

- [ ] A granted lease leads to exactly one POST with the stored message, the stored uuid header and idempotency-key = UETR
- [ ] 202 is recorded as ACCEPTED; 400 and 422 as REJECTED with OPF's error; 401, 403, 5xx and timeout as FAILED_RETRYABLE with the cause
- [ ] A 422 raises an alert, because a genuine repeat cannot produce one
- [ ] No retry happens inside an attempt; the next attempt is the redelivered message
- [ ] A refused lease is handled as the API says: delete the message, or leave it to come back
- [ ] The Submitter role has no DynamoDB permission
- [ ] Integration tests run against the WireMock stub for every stubbed answer

## 7. New stories, [Cap 10]

**Story: [Cap 10] Tables for the asynchronous design: global table, stream, pending index and the Breaker table**
Points: 5 · Sprint 26.4.5

Description

As the Wire II developer, I want the data stores of the asynchronous design provisioned through the pipeline, so that the API, the dispatch and the Sweeper have what they are built on.

The transaction table that exists today is a single-region table keyed by UETR. The design needs it as a global table with replicas in us-east-1 and us-west-2, a change stream with the new image for the dispatch, and a sparse index on the pending region and the next check time for the Sweeper. It also needs a second, small global table for the circuit breaker, keyed by region and dependency, with four records. Both tables are encrypted with a customer-managed key in each region and are reachable only through the VPC endpoint.

Acceptance criteria

- [ ] Transaction table is a global table with replicas in us-east-1 and us-west-2 in dev, with the stream enabled with new images
- [ ] Sparse index on pending region and next check time exists and lists only unfinished records
- [ ] Breaker table exists as a global table, on-demand, keyed by region and dependency, with no index
- [ ] Both tables are encrypted with the approved customer-managed key in each region; point-in-time recovery is on
- [ ] Task and function roles are limited per table to the operations each component needs
- [ ] Table names and regions are configuration per environment; expiry is enabled on the transaction table

**Story: [Cap 10] Internal API for the Submitter: lease, outcome and reported**
Points: 5 · Sprint 26.4.6

Description

As the Wire II Submitter, I want three internal endpoints to claim a payment, record its outcome and mark it reported, so that I can do my work without any access to the table.

The lease endpoint claims the payment for one attempt and returns the stored message and header value. It is granted only if this region owns the payment, no lease is live, the payment is not reported and its submit deadline has not passed. A granted lease sets the status to SUBMITTING, clears the dispatch flag, counts the attempt and lasts two minutes. A finished payment that PayX has not been told about returns its outcome, so the Submitter only reports. The outcome endpoint records the result, releases the lease and keeps the count of unanswered attempts right; the first final outcome wins. The reported endpoint marks a finished payment as reported and takes it out of the pending index. The endpoints are private, authenticated service to service, and not published on Exchange.

Acceptance criteria

- [ ] Lease is granted only when all conditions hold, and each refusal returns its reason: not the owner, lease held, already reported, deadline passed
- [ ] A granted lease sets SUBMITTING, clears the dispatch flag, raises the attempt counters and expires after the configured time
- [ ] A finished, unreported payment returns "terminal, not yet reported" with its outcome
- [ ] Outcome is accepted only with the current lease id; the first final outcome wins and a later one is refused
- [ ] The unanswered count is taken back on a clear answer, or when the Submitter reports that nothing was sent
- [ ] Reported is accepted only on a finished payment and removes it from the pending index
- [ ] Only the Submitter's identity can call the three endpoints

**Story: [Cap 10] Circuit breaker at the lease for OPF and Turing**
Points: 5 · Sprint 27.1.1

Description

As the on-call engineer, I want Wire II to stop sending by itself when OPF or Turing is down and to start again by itself when it is back, so that an outage of hours costs delay and no manual replay.

The API keeps one breaker record per region and dependency in the Breaker table. Each retryable outcome adds its UETR to the record; the fifth different payment in a row opens the breaker and pages. While it is open the API refuses leases, so nothing is sent and no retry is used up. One waiting payment at a time is let through as a probe; any clear answer from OPF, accepted or rejected, closes the breaker. The record also carries the proof that the region can reach OPF: the time of the last clear answer or the last passing path check. A new internal endpoint takes the result of a path check: a pass refreshes the proof, three fails in a row open the breaker.

Acceptance criteria

- [ ] Five different payments failing in a row open the breaker; one payment failing five times does not
- [ ] While open, lease requests are refused with the reason "breaker open" unless the UETR is the named probe
- [ ] A clear answer, accepted or rejected, empties the failed set, sets the last-proven time and closes the breaker
- [ ] A failed probe reopens the breaker and doubles the wait: 1, 2, 4, 8 minutes, then every 15
- [ ] The path-check endpoint records pass or fail; three fails in a row open the breaker; a pass closes an open breaker when no payment is waiting
- [ ] Two API tasks handling the fifth failure at once leave one consistent record
- [ ] Thresholds and waits are configuration; opening pages the on-call service

**Story: [Cap 10] Sweeper Lambda: put stalled payments back on the path, probe, drain and park**
Points: 5 · Sprint 27.1.1

Description

As the Wire II developer, I want a small scheduled function that finds payments that are stuck and puts them back on the normal path, so that no payment depends on a queue message that was lost.

The Sweeper is a Go Lambda in each region, started every minute, one run at a time. It reads the pending index and requests dispatch again for payments of its region that have made no progress for about 20 minutes, and for finished payments PayX was never told about. While the breaker of its region is open it holds, and when a probe is due it names one waiting payment as the probe. After the breaker closes it drains the waiting payments, oldest first, 60 per run. A payment whose submit deadline has passed is marked parked, with the reason: never sent, or outcome unknown. The Sweeper writes the tables directly through a short list of conditional updates and never touches the stored message or an outcome. It never calls Turing, OPF, PayX or the API.

Acceptance criteria

- [ ] Runs every minute in each region with at most one run at a time
- [ ] A stalled payment of its region gets a new dispatch request, and the write always changes the item so the stream fires
- [ ] A finished, unreported payment gets a new dispatch request until PayX has been told
- [ ] While the breaker is open nothing is dispatched except one named probe when a probe is due; a probe whose lease ran out is treated as failed
- [ ] After the breaker closes, waiting payments are dispatched oldest first, at most 60 per run
- [ ] A payment past its deadline with no live lease is parked with reason NOT_SENT or OUTCOME_UNKNOWN and an alert
- [ ] The function role allows only the listed attributes to be updated, and a contract test checks them against the record schema

**Story: [Cap 10] Sweeper: path check and regional takeover of sending**
Points: 5 · Sprint 27.1.2

Description

As the on-call engineer, I want the standby region to take over the sending when the serving region cannot reach OPF, without a routing switch and without a person, so that payments are delayed by minutes and not by an incident call.

A region with no traffic has a closed breaker without having proven anything. So when a region has proven nothing for a minute, its Sweeper puts a path-check message on the submit queue; the Submitter makes one call to FIS that creates no payment and reports the result. A region is able when its breaker is closed and proven within three minutes. A region is out when its breaker has been open for five minutes, or it has proven nothing for five minutes. When the other region is out and this region is able, this region's Sweeper adopts the other region's unfinished payments that have no live lease: oldest first, 60 per run. Its own Pipe, queue and Submitter then send the stored message with the same key. If both breakers are open nobody adopts, because OPF itself is down.

Acceptance criteria

- [ ] A path-check message is queued when the region has proven nothing for a minute, in both regions, all the time
- [ ] A region is treated as able or out exactly by the two rules, and never as both
- [ ] Adoption happens only when the other region is out and this region is able; it is one conditional write per payment
- [ ] At most 60 payments are adopted per run, oldest first, and none that has a live lease
- [ ] Adoption stops as soon as the other region is able again; adopted payments are never moved back
- [ ] With both breakers open no payment is adopted
- [ ] A test with two regions shows a payment accepted in the cut-off region being sent by the other region about six minutes later

**Story: [Cap 10] Operator replay and manual breaker control, with the runbook**
Points: 3 · Sprint 27.1.2

Description

As the on-call engineer, I want a controlled way to replay a parked payment and to open or close a breaker by hand, so that I can act on an alert without touching the table.

Two internal endpoints for the operations role. The replay endpoint requests dispatch again for a parked payment and sets a new deadline; it is refused for a payment that is finished and reported. The breaker endpoint opens or closes a breaker, for planned FIS maintenance or to force a state. Both are audited. The runbook says what each parked reason means and when a replay is safe: always for a payment that was never sent, and only after OPF has been checked for a payment whose outcome is unknown.

Acceptance criteria

- [ ] Replay of a parked payment sets a new deadline and requests dispatch; the payment then follows the normal path
- [ ] Replay is refused on a payment that is finished and reported
- [ ] A breaker can be opened and closed by hand; while it is open by hand no payment is sent
- [ ] Both endpoints accept only the operations role, and every call is written to the audit log with who and when
- [ ] Runbook covers the three parked reasons, the checks before a replay, and the use of the breaker for planned maintenance

**Story: [Cap 10] Alarms for the breaker, the regions, the takeover, parked payments and the Sweeper**
Points: 3 · Sprint 27.1.2

Description

As the on-call engineer, I want an alarm for each way the asynchronous path can stop moving, named so that I know what happened without reading logs.

Add the alarms that the breaker, the two regions and the Sweeper need, on top of the queue and service alarms that come with the dispatch and Submitter stories. Each alarm goes to the on-call service with the region and the dependency in its name.

Acceptance criteria

- [ ] Alarm when a breaker opens, by dependency and region
- [ ] Alarm when a region is not able, including the standby
- [ ] Alarm when path checks fail, by region
- [ ] Alarm when a takeover starts
- [ ] Alarm on parked payments, by reason
- [ ] Alarm when no sweep has run in a region for five minutes
- [ ] Alarm on payments that are finished but not reported within the expected time, and on stalled payments

**Story: [Cap 10] Second region for Wire II and a failover game day in non-production**
Points: 5 · Sprint 27.1.3

Description

As the tech lead, I want Wire II running in both regions and its failure handling shown in a planned exercise, so that the first real outage is not the first test.

Deploy the API, the Submitter, the Sweeper, the Pipe and the queues in us-west-2 at minimum capacity, with the credentials, secrets and keys the region needs, and confirm that path checks pass from both regions. Then run the exercise in a non-production environment with the stub in place of OPF: a vendor outage of an hour, a region cut off from the vendor, and the loss of a region with payments in flight. Record what happened against what the design says should happen.

Acceptance criteria

- [ ] All components run in us-west-2 with their own Pipe, queue and dead-letter queues, deployed before the exercise and not during it
- [ ] Credentials, secrets and keys exist in both regions, and path checks pass from both
- [ ] Vendor outage: the breaker opens, nothing reaches the dead-letter queue, and the backlog drains by itself afterwards
- [ ] Region cut off from the vendor: the other region takes over the sending, and no payment is sent twice
- [ ] Region lost with payments in flight: the other region adopts them and reports the final status
- [ ] Results, timings and defects are recorded and reviewed with the tech lead

## 8. New stories, [Cap 3]

**Story: [Cap 3] Dispatch without code: EventBridge Pipe, submit queue and dead-letter queues**
Points: 5 · Sprint 26.4.6

Description

As the Wire II developer, I want a stored payment to reach the Submitter through managed services only, so that there is no second write that could be lost between storing a payment and queueing it.

Provision, per region, an EventBridge Pipe from the stream of the transaction table to an SQS queue. The Pipe passes only records owned by its region that request dispatch, and sends the UETR alone. The queue is a standard queue with a dead-letter queue; the Pipe has its own dead-letter queue. If the pipeline cannot deploy EventBridge Pipes, a small stream-triggered function applies the same filter and sends to the queue, and nothing else changes.

Acceptance criteria

- [ ] A stored payment that requests dispatch appears on the submit queue of its own region as a message holding the UETR only
- [ ] A record owned by the other region, or one that does not request dispatch, produces no message
- [ ] Submit queue and dead-letter queue exist with the agreed retention, maximum receives and visibility timeout, encrypted with the approved key
- [ ] Undeliverable events go to the Pipe's own dead-letter queue
- [ ] Alarms exist for the age of the oldest message, a message arriving in a dead-letter queue, and Pipe delivery failures
- [ ] Support for Pipes in the deployment pipeline is confirmed, or the function fallback is built and recorded as the decision

**Story: [Cap 3] Scaffold the Wire II Submitter service with the FIS client and its pipeline**
Points: 5 · Sprint 26.4.6

Description

As the Wire II developer, I want the Submitter as its own deployable service with the FIS client inside, so that the component that sees clear-text account numbers is separate from the API and has no access to the data store.

Create the service from the Fargate template as a queue consumer. Move the FIS client prototype, its configuration and its tests into it. Reference secrets through the secrets library. Scale on queue depth and the age of the oldest message. On shutdown stop receiving and finish the attempts in flight. Complete the first pipeline run and deployment to the dev environment.

Acceptance criteria

- [ ] Service exists in its own repository with pipeline, Sonar gates passing and a deployment to dev with a passing health check
- [ ] FIS client, configuration and tests live in the service; the separate prototype is retired or kept by a recorded decision
- [ ] No plaintext credentials remain
- [ ] The service role can use its queue, call Turing, OPF, the PayX status API and the Wire II internal endpoints, and has no DynamoDB permission
- [ ] Scaling follows queue depth and message age; consumer concurrency is configuration
- [ ] On a stop signal the service stops receiving and finishes or releases the attempts in flight
- [ ] Runs locally in a container against the WireMock stub and a local queue with one command

**Story: [Cap 3] Detokenise account numbers through Turing in the Submitter**
Points: 3 · Sprint 27.1.1

Description

As the Wire II developer, I want the Submitter to replace the account tokens in the stored message with the real values just before sending, so that a clear-text account number exists only in memory and only for one attempt.

Call the Turing detokenise endpoint with a token from the DevExchange gateway, for the token types in use. Replace the tokens in the message in memory and change nothing else. Treat each Turing answer by its meaning: a malformed request or an invalid token blocks the payment and raises an alert, a lookup failure or a server error is retryable and counts toward the Turing breaker. Wire logging of the HTTP client is off, and request and response bodies are excluded from tracing.

Acceptance criteria

- [ ] Tokens are replaced in memory for the attempt; the stored message, the queue and the logs never hold a clear-text account number
- [ ] Answers 105, 106, 108 and 400 block the payment as not retryable and raise an alert
- [ ] Answers 109, 401, 403 and 500 are reported as retryable with the cause, so the API can count them for the Turing breaker
- [ ] Token types BAN, GLOBALBAN, UKBAN and NONUSBAN are supported by configuration
- [ ] Tests cover every answer against a stub

**Story: [Cap 3] Submitter: report the final status to PayX, and handle retries, the dead-letter queue and parked payments**
Points: 5 · Sprint 27.1.1

Description

As the PayX integration service, I want the final status of every payment reported once, so that I can produce the clearing stage; and as the on-call engineer, I want failed attempts spaced out and stopped after a limit.

For a final outcome the Submitter calls the PayX status API, marks the payment reported through the Wire II API and deletes the message. If the report fails the message comes back, the lease answers "terminal, not yet reported", and the Submitter only reports. For a retryable outcome it leaves the message and sets its visibility from the receive count: 30 seconds, 1, 2, 5, 10, then 15 minutes. After the maximum number of receives, about two hours, the message moves to the dead-letter queue and a payment that is not finished is parked with the reason "maximum receives". A finished payment is never parked.

Acceptance criteria

- [ ] An accepted or rejected payment is reported to the PayX status API, marked reported and its message deleted
- [ ] A failed report leaves the message; the next attempt reports without calling OPF again
- [ ] A retryable outcome sets the next visibility from the receive count as listed
- [ ] After the maximum receives the message is in the dead-letter queue and an unfinished payment is PARKED with reason MAX_RECEIVES
- [ ] A finished payment whose report keeps failing stays finished and is reported later
- [ ] Each lease answer is handled as defined: granted, terminal not reported, refused with delete, refused with leave
- [ ] Tests cover the report, the failed report, the backoff steps and the move to the dead-letter queue

**Story: [Cap 3] Submitter path check: prove once a minute that the region can reach FIS**
Points: 3 · Sprint 27.1.2

Description

As the on-call engineer, I want each region to prove that its road to FIS works even when it has no payments, so that a broken route in the standby is found before it is needed.

The Submitter handles a second kind of message from the submit queue: a path check, put there by the Sweeper. It makes one authenticated call to FIS that creates no payment and reports pass or fail to the Wire II API. The call is configuration: a FIS heartbeat or status URL if FIS has one, otherwise a token request to the FIS gateway. A check message is deleted after one attempt, and a check older than two minutes is deleted unread.

Acceptance criteria

- [ ] A path-check message leads to one call to FIS and one report of pass or fail to the Wire II API
- [ ] No lease is taken, nothing is detokenised and no payment is created
- [ ] The message is deleted after one attempt whatever the result; a message older than two minutes is deleted unread
- [ ] The FIS call used for the check is configuration, and the choice agreed with FIS is recorded
- [ ] A payment message and a path-check message cannot be confused; tests cover both

**Story: [Cap 3] Prove in the FIS sandbox that a repeated submission creates no second payment**
Points: 3 · Sprint 27.1.3 · Blocked until FIS has configured the sandbox

Description

As the tech lead, I want to see how the OPF sandbox treats a repeat of a submission, so that the design's promise of no duplicate wire rests on a test and not on an assumption.

Every recovery path in the design re-sends the same stored message with the same idempotency key. Run that against the real sandbox: the same request twice in a row, twice at the same moment, and again after a wait. Ask FIS how long a key is remembered and whether the comparison is byte-exact. Record the answers, since the submit deadline is set from them.

Acceptance criteria

- [ ] The same request sent twice gets an answer that creates one payment at OPF, confirmed with FIS
- [ ] Two identical requests sent at the same moment create one payment
- [ ] The same key with a different message is rejected
- [ ] FIS's answers on key retention and on what is compared are recorded
- [ ] The submit deadline value is set from the result, or the open point is escalated

## 9. Outside this file

Three gaps are named on the epic itself and are not covered here, because the asynchronous design does not change them.

1. **Drawdown mapping.** No story exists for the drawdown wire types, and no owner is assigned.
2. **Behaviours carried over from the earlier converter.** The epic asks for a recorded decision on each one. No story exists.
3. **The epic's own text.** Its description and its fifth acceptance criterion still say one synchronous service with no queue and no retries of its own. That needs rewording by whoever owns the epic.
