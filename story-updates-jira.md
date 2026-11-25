# Sprint 26.4.5: the existing stories to use, and what to change in each

No new story is created. Each planned item takes an existing story. Where the planned scope is smaller or different, the title, the points and the acceptance criteria change as given below. Tejaswi's Okta entitlements story is not in this file.

## 1. The list

| # | Planned item | Who | Existing story to use | Capability | Points |
|---|---|---|---|---|---|
| 1 | V9 mapper in the mapping library | Vasanth | [Cap 10] Add pain.001 V9 output to bank-wire-object-mapper behind a version argument | 10 | 5, no change |
| 2 | Integrating the mapping library | Vasanth | ENTPMTS-143560, Wire the pain.001 mapper into the Wire Rail II service as the generation step | 10 | 3, no change |
| 3 | Update the spec for the async design | Vasanth | ENTPMTS-143564, Implement Business Application Header (BAH) wrapping per the OPF API contract | 10 | 3, no change |
| 4 | GitHub onboarding of the Submitter | Tejaswi | [Cap 3] Scaffold the Wire II Submitter service with the FIS client and its pipeline | 3 | 5 becomes 2 |
| 5 | Turing API integration code | Tejaswi | [Cap 3] Detokenise account numbers through Turing in the Submitter | 3 | 3 becomes 5 |
| 6 | DevExchange onboarding of the Submitter, health check only | Kaushik | ENTPMTS-145619, Merge the FIS client prototype into the scaffolded Wire Rail II service and get the pipeline green | 3 | 3 (5 becomes 3 if it still shows 5) |
| 7 | Code for the database entries | Kaushik | [Cap 10] Tables for the asynchronous design: global table, stream, pending index and the Breaker table | 10 | 5, no change |
| 8 | SQS queue setup and onboarding | Kaushik | [Cap 3] Dispatch without code: EventBridge Pipe, submit queue and dead-letter queues | 3 | 5 becomes 3 |

Totals: Vasanth 11, Kaushik 11, Tejaswi 7 plus her entitlements story of 5, which makes 12.

Three stories keep their key and get new content (3, 6 and 7). The reason for each is given with the story.

## 2. Message for Dylan

```
Stories for sprint 26.4.5. No new tickets; three existing ones get a new title and criteria, I am updating them today.

Vasanth (11)
- [Cap 10] Add pain.001 V9 output to bank-wire-object-mapper behind a version argument (5)
- ENTPMTS-143560 Wire the pain.001 mapper into the Wire Rail II service (3)
- ENTPMTS-143564, retitled: Update the Wire II API specification for the asynchronous design (3)

Tejaswi (12)
- [Cap 3] Scaffold the Wire II Submitter service, retitled: Create the Wire II Submitter repository (2, was 5)
- [Cap 3] Detokenise account numbers through Turing in the Submitter (5, was 3)
- Okta entitlements onboarding (5), Tejaswi's own story

Kaushik (11)
- ENTPMTS-145619, retitled: Onboard the Wire II Submitter to DevExchange and deploy it to dev with a health check (3)
- [Cap 10] Tables for the asynchronous design, retitled: Transaction record: data access code of the Wire II API (5)
- [Cap 3] Dispatch without code, retitled: Submit queue and dead-letter queue for Wire II (3, was 5)
```

## 3. The stories

### Story 1, Vasanth

Story: **[Cap 10] Add pain.001 V9 output to bank-wire-object-mapper behind a version argument**
Capability 10. Points: 5, no change. Title, description and acceptance criteria: no change. Only move it into the sprint.

### Story 2, Vasanth

Story: **ENTPMTS-143560, Wire the pain.001 mapper into the Wire Rail II service as the generation step**
Capability 10. Points: 3, no change. Title: no change.

Description, replace with:

```
The mapping library is published to Artifactory and now produces both the pain.001.001.11 document and the FIS JSON request with its header. This story makes the Wire II API use it: the library becomes a dependency of the service, and one generation step takes the payment object and returns the request that will be stored and sent. The step is deterministic: the same payment always gives the same output, because the message is stored once and sent unchanged on every attempt. Storing the result and answering the caller belong to the intake story.
```

Acceptance criteria, replace with:

```
- [ ] bank-wire-object-mapper is a dependency of the Wire II API, resolved from Artifactory at a published version
- [ ] One generation step takes the payment object and returns the pain.001.001.11 document and the FIS JSON request with the business application header
- [ ] The header's from and to identifiers are service configuration per environment, not code
- [ ] The UETR is read from the configured payment object field and appears on the payment identification
- [ ] A payment that cannot be mapped gives one error that names every field at fault, and nothing partial is returned
- [ ] Building the same payment twice gives byte-identical output; no value comes from the wall clock
- [ ] Unit tests in the service run the step for a customer wire and a bank wire sample and compare with the expected JSON
- [ ] The service builds in the pipeline with the library and the Sonar gate passes
```

### Story 3, Vasanth

Story: **ENTPMTS-143564, Implement Business Application Header (BAH) wrapping per the OPF API contract**
Capability 10. Points: 3, no change.

Why this key: the header was built inside the mapping library with the XML to JSON story, so nothing is left to do here. Its one open line, the identifiers per environment, is in story 2. The key takes the specification update at the same points.

Comment to add before the change:

```
The business application header was delivered with ENTPMTS-143562 inside bank-wire-object-mapper. The identifiers per environment are covered by ENTPMTS-143560. This story now carries the update of the API specification for the asynchronous design, at the same 3 points.
```

Title, replace with:

```
[Cap 10] Update the Wire II API specification for the asynchronous design
```

Description, replace with:

```
As the PayX integration service, I want the published Wire II API specification to describe the asynchronous design, so that my integration is built against the contract we agreed and not against the synchronous one.

The registered specification still describes one synchronous call that returns the OPF result. Update it: the payment is accepted, stored and answered at once, and the final status follows later. The request is the one agreed with the PayX integration team. The field-level validation rules already in the specification stay.
```

Acceptance criteria, replace with:

```
- [ ] The request matches the contract agreed with PayX: payment lifecycle id, exchange object id, saga type and the exchange object
- [ ] POST /wires is specified with 202 for an accepted payment, and with the answers for a repeat: 202 while in flight, 200 with the stored outcome when finished, 409 for different content
- [ ] 400 is specified with the list of fields at fault
- [ ] The status request is specified with the status values and, when finished, the outcome
- [ ] The identifier that makes a repeat safe, and the header that carries it, are specified
- [ ] The final status reported to PayX is described: the request as received plus the rail status, final results only
- [ ] The field-level validation rules already in the specification are unchanged
- [ ] Every answer has an example, and a test checks each example against the specification
- [ ] The specification passes the lint check, is reviewed by the team and is published on Exchange as a new version
```

### Story 4, Tejaswi

Story: **[Cap 3] Scaffold the Wire II Submitter service with the FIS client and its pipeline**
Capability 3. Points: 5 becomes 2.

Title, replace with:

```
[Cap 3] Create the Wire II Submitter repository from the Fargate template
```

Description, replace with:

```
As the Wire II developer, I want the Submitter to have its own repository, so that the component that sends to OPF is separate from the API and the work on it can start.

Create the repository from the Fargate template through DevNav Hub and bring it to a state the team can build on. The onboarding to DevExchange and the first deployment are a separate story. The FIS client, the scaling and the shutdown behaviour follow in later stories.
```

Acceptance criteria, replace with:

```
- [ ] Repository is created through DevNav Hub from the Fargate template with the agreed name
- [ ] Team access, branch protection and the development branch are set as on the other Wire II repositories
- [ ] The generated project builds locally and its tests pass
- [ ] The pipeline file is present and the first pipeline build runs
- [ ] No plaintext credentials are in the repository
- [ ] The README says what the service is and how to run it locally
```

### Story 5, Tejaswi

Story: **[Cap 3] Detokenise account numbers through Turing in the Submitter**
Capability 3. Points: 3 becomes 5. Title: no change.

Why 5: the story now also builds the client that gets its token from the gateway, and a stub of Turing, because the code is written before the Submitter's onboarding is approved.

Description, add at the end:

```
The code is written in the new Submitter repository before the service's onboarding is approved. Until then the client is tested against a stub of Turing, and the call to the real endpoint follows as soon as the service is allowed to call it.
```

Acceptance criteria, replace with:

```
- [ ] A Turing client in the Submitter gets its token from the DevExchange gateway, keeps it and renews it before it expires
- [ ] Tokens are replaced in memory for the attempt; the stored message, the queue and the logs never hold a clear-text account number
- [ ] Answers 105, 106, 108 and 400 block the payment as not retryable and raise an alert
- [ ] Answers 109, 401, 403 and 500 are reported as retryable with the cause
- [ ] Token types BAN, GLOBALBAN, UKBAN and NONUSBAN are supported by configuration
- [ ] Timeouts are configuration; wire logging of the client is off and bodies are excluded from tracing
- [ ] Tests cover every answer against a stub of Turing
- [ ] The real Turing endpoint is called from dev once the Submitter's onboarding is approved, or the open step is noted on the story
```

### Story 6, Kaushik

Story: **ENTPMTS-145619, Merge the FIS client prototype into the scaffolded Wire Rail II service and get the pipeline green**
(If the title was already changed, it reads: Get the scaffolded Wire II API building with the mapper library and the pipeline green.)
Capability 3. Points: 3.

Why this key: its content was the first green pipeline and deployment of a scaffolded service. That is now needed for the Submitter. The mapping library as a dependency of the API is in story 2.

Title, replace with:

```
[Cap 3] Onboard the Wire II Submitter to DevExchange and deploy it to dev with a health check
```

Description, replace with:

```
As the Wire II developer, I want the Submitter registered in DevExchange and running in dev, so that it has an identity, can ask for access to the APIs it calls, and the code that follows has a place to run.

The service exposes a health check only. Register it, complete the onboarding steps, and bring the pipeline to the first deployment in dev. The request to call Turing is raised from this registration. This story starts when the repository exists.
```

Acceptance criteria, replace with:

```
- [ ] The Submitter is registered in DevExchange with an OpenAPI specification that holds the health check endpoint only
- [ ] The onboarding approvals are requested and their status is recorded on the story
- [ ] The pipeline builds, scans and deploys the service to dev, and the health check passes
- [ ] The Sonar quality and coverage gates pass
- [ ] The service has its gateway credentials in dev, kept in the secrets store
- [ ] Access to the Turing API is requested for the service
```

### Story 7, Kaushik

Story: **[Cap 10] Tables for the asynchronous design: global table, stream, pending index and the Breaker table**
Capability 10. Points: 5, no change.

Why this key: the two tables were created under ENTPMTS-143580, and the code criteria taken out of that story are the work planned here. What is left of the table setup is listed in section 4.

Title, replace with:

```
[Cap 10] Transaction record: data access code of the Wire II API with conditional writes
```

Description, replace with:

```
As the Wire II developer, I want one data access layer for the transaction record in which every write is conditional, so that a payment has exactly one record, two writers can never overwrite each other, and the stored message is never changed after it was written.

Implement the records and their access in the Wire II API for the transaction table and the Breaker table, as in the reviewed table design: create only if the UETR is new, read by UETR, and update only when the version is unchanged. Locally the code is tested against an in-memory or local DynamoDB, so the story does not wait for access to the real tables.
```

Acceptance criteria, replace with:

```
- [ ] Record classes and a repository exist in the Wire II API for the transaction table and the Breaker table, matching the reviewed table design
- [ ] Create is a conditional put: a second create for the same UETR fails and returns the existing record
- [ ] Every update checks the version and raises it; a stale update is refused
- [ ] The stored message and the payment hash cannot be changed after the first write
- [ ] No clear-text account number is written; the record holds tokens only
- [ ] Table names and the region are configuration per environment
- [ ] Tests run against an in-memory or local DynamoDB and cover create, repeat, read and concurrent update
- [ ] The code is run against the dev tables once the service can reach them, or the open step is noted on the story
```

### Story 8, Kaushik

Story: **[Cap 3] Dispatch without code: EventBridge Pipe, submit queue and dead-letter queues**
Capability 3. Points: 5 becomes 3.

Title, replace with:

```
[Cap 3] Submit queue and dead-letter queue for Wire II
```

Description, replace with:

```
As the Wire II developer, I want the submit queue and its dead-letter queue provisioned through the team's pipeline, so that the Submitter has the queue it reads from.

Complete the onboarding for SQS and provision a standard queue with a dead-letter queue in dev. The Pipe that fills the queue from the table follows in a later story.
```

Acceptance criteria, replace with:

```
- [ ] The onboarding for SQS is completed through the team's provisioning path and the steps are recorded
- [ ] Submit queue and dead-letter queue exist in dev, encrypted with the approved key
- [ ] Retention, maximum receives and visibility timeout are set to the agreed values and are configuration
- [ ] Only the Wire II roles can send to and receive from the queue
- [ ] A test message sent to the queue is received and deleted; a message received more often than the maximum moves to the dead-letter queue
- [ ] Alarms exist for the age of the oldest message and for a message arriving in the dead-letter queue
```

## 4. What comes off these stories, and where to keep it

Three stories get smaller or change content. The work that leaves them is still needed. Add each block as extra acceptance criteria to the story named, so it is not lost. Two of those stories then hold more than 5 points and need a new estimate when they are planned.

**From the Submitter scaffold story, add to ENTPMTS-143577 ([Cap 3] Wire II Submitter: lease the payment, send the stored pain.001 to the OPF API and record the outcome):**

```
- [ ] FIS client, its configuration and its tests live in the Submitter; the separate prototype is retired or kept by a recorded decision
- [ ] The service role can use its queue, call Turing, OPF, the PayX status API and the Wire II internal endpoints, and has no DynamoDB permission
```

**From the Submitter scaffold story, add to [Cap 3] Submitter: report the final status to PayX, and handle retries, the dead-letter queue and parked payments:**

```
- [ ] Scaling follows queue depth and message age; consumer concurrency is configuration
- [ ] On a stop signal the service stops receiving and finishes or releases the attempts in flight
```

**From the dispatch story, add to ENTPMTS-143577 as well:**

```
- [ ] The transaction table has its stream enabled with new images
- [ ] An EventBridge Pipe passes a stored payment that requests dispatch to the submit queue of its own region as a message holding the UETR only
- [ ] A record owned by the other region, or one that does not request dispatch, produces no message
- [ ] Undeliverable events go to the Pipe's own dead-letter queue, with an alarm
- [ ] Support for Pipes in the deployment pipeline is confirmed, or the function fallback is built and recorded as the decision
```

**From the tables story, add to [Cap 10] Sweeper Lambda: put stalled payments back on the path, probe, drain and park:**

```
- [ ] Sparse index on pending region and next check time exists and lists only unfinished records
- [ ] The keys of the pending index are written while a record is unfinished or unreported, and removed when it is finished and reported
```

**From the tables story, add to [Cap 10] Second region for Wire II and a failover game day in non-production:**

```
- [ ] Transaction table and Breaker table are global tables with replicas in us-east-1 and us-west-2
- [ ] Both tables are encrypted with the approved customer-managed key in each region; point-in-time recovery is on
- [ ] Task and function roles are limited per table to the operations each component needs; expiry is enabled on the transaction table
```

**From ENTPMTS-143560, add to ENTPMTS-143578 ([Cap 10] Accept a payment idempotently):**

```
- [ ] A build or validation failure returns 400 naming the field, the payment is recorded as rejected, and nothing is queued or sent to OPF
```

In ENTPMTS-143578, remove the line "Contract is agreed with the PayX integration team and published on Exchange; contract tests cover every answer". It is now story 3.
