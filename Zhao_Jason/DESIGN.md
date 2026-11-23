 # Duplicate-Safe Event Delivery for Monitoring Student-AI Interactions

<!-- Copy this file into your project directory as DESIGN.md. Replace prompts
with your own explanations as sections become due. Create assets and evidence
directories for files you include. Remove instructional comments at final submission. -->

**Author:** Jason Zhao

**Course:** CSC 59866

**Selected project and option:** 
Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education 

**Current assignment:** 3

**Friday commit deadline:** November 22, 2026, at 11:59 p.m. New York time

**Monday presentation:** November 23, 2026, at 6:30 p.m. New York time

**Last updated:** 11/21/2026

**Document status:** Proposed
<!-- Commit and push by Friday. Supply the exact commit identifier through the
course submission channel. Every Monday, present your own work and orally review
the same selected group mate for the remainder of the semester. No written
peer-review report, review link, or review transcript is required. -->

## Progress note

Following earlier feedback, I narrowed the project to one distributed-systems question: when an accepted event is lost between intake and analysis, can the system find it and retrieve it from its saved record? This week I selected separate services with a queue over a single-application baseline, added a reconciliation sweep that re-sends any accepted event not yet analyzed, recorded two decisions, drew three architecture views, and specified the intake and sweep contracts. In experiment E2, losing 10 of 100 messages left 10 events unanalyzed and 2 flags missed in every queue-only trial; with the sweep, all 100 events and all 5 flags were recovered in 10 of 10 trials. Experiment E1 showed the worker must ignore repeats, because the sweep re-sends late events. Unresolved: real RabbitMQ loss and recovery time, since both experiments simulated the queue. Next: run E3 with RabbitMQ, then build intake.

## Contents

- [1 Problem research and scope](#1-problem-research-and-scope)
- [2 Requirements and system specification](#2-requirements-and-system-specification)
- [3 Alternatives and selected solution](#3-alternatives-and-selected-solution)
- [4 Architectural design](#4-architectural-design)
- [5 Component design and interfaces](#5-component-design-and-interfaces)
- [6 Data structure design](#6-data-structure-design)
- [7 Algorithm design and execution](#7-algorithm-design-and-execution)
- [8 Failures and security](#8-failures-and-security)
- [9 Implementation and feasibility](#9-implementation-and-feasibility)
- [10 Testing and verification](#10-testing-and-verification)
- [11 Operation maintenance and future work](#11-operation-maintenance-and-future-work)
- [12 Revision history and references](#12-revision-history-and-references)

## 1 Problem research and scope
 
**Problem and affected people:** In a large introductory Python course, an instructor cannot read every student's conversations with an AI tutor. A simple rule can point the instructor to interactions worth reading, for example "three full-solution requests within ten minutes." That rule is only right if the records behind it are complete. In a system of separate services, an event can be accepted and saved, and then its message can be lost on the way to analysis. My engineering question is therefore narrow:
 
> When an accepted event is lost between intake and analysis, can the system detect the loss and retrieve the event from its saved record, so that every accepted event is eventually analyzed and shown on the instructor dashboard exactly once?
 
Retrieval means sending a lost event again, which creates a second risk: the "lost" message may only have been late and arrive twice. So the design must also make repeated deliveries harmless.
 
This is a question about **recovering lost data in a distributed system**, not about whether the tutor teaches well, whether a flag correctly identifies a struggling student, or whether students learn more.
 
| Affected people | Need | Status |
|---|---|---|
| Instructor or teaching assistant (TA) | See which students a stated rule flagged in the last day, and read the history behind each flag | **Assumed (A1).** Not yet confirmed with an instructor; I will show one instructor or TA the rule and dashboard and revise from their answer. |
| Student | Keep working while analysis runs in the background; an accepted request is never rejected because analysis is down | Design goal (Q3) |
| Course and institution | A flag is information for review, never an automatic judgment of cheating or struggling | Constraint C1 |
 
**Current workflow:**
 
1. A student asks the AI tutor for help with an assignment.
2. The interaction is recorded as an event.
3. An analysis step applies a review rule to the student's events.
4. The instructor opens a dashboard and reads the history behind each flag.
 
The difficulty is between steps 2 and 4. Recording, analysis, and the dashboard are separate parts that can fail independently, which is called **partial failure**.
 
*Concrete scenario.* Fictional student G-S1 asks for a full solution three times in eight minutes. All three requests are accepted and saved, but the queue message for the third one (G003) is lost. The worker never sees G003, so G-S1 is never flagged, and the dashboard shows G003 as `pending analysis` forever. Nothing reports an error. The fix is a **reconciliation sweep**: a periodic check of the saved records that finds G003 has no analysis result and sends it again. If the original G003 message was only delayed and arrives later, it must be ignored rather than counted twice.
 
| Delivery failure | Effect on the rule | Effect on the instructor at step 4 |
|---|---|---|
| Accepted event's message is lost | Event never analyzed; flag missed | A student the rule should surface is not shown, and nothing reports an error |
| Event counted twice | Count too high; false flag | Time spent on a student who did not meet the rule |
| Analysis down and nothing kept | Dashboard blank or stale | Cannot tell "nothing happened" from "nothing analyzed" |
 
**Evidence:**
 
- **Documented.** Qian, Liu, and Zhu (2026) rated AI feedback in an introductory Python course at an average of 1.84 out of 5, with more than 40 percent at the lowest rating, and students receiving it did worse on debugging and the final exam than students receiving data-driven feedback ([Humanities and Social Sciences Communications](https://www.nature.com/articles/s41599-026-07920-7)). Heickal and Lan (2026), with 215 students and 6,693 submissions, found natural-language hints helped completion, while generated test cases helped only when valid ([PMLR](https://proceedings.mlr.press/v339/heickal26a.html)). Both support giving instructors records to review. Neither measures lost or duplicated events.
- **Documented, existing approaches.** Moodle supports rule-based analytics over course activity but does not interpret AI-tutor conversations ([Moodle](https://docs.moodle.org/502/en/Using_analytics)). OpenAI's [function calling](https://developers.openai.com/api/docs/guides/function-calling) and [agent orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration) are building blocks, not durable delivery to an instructor view. The NSF award ([2540315](https://www.nsf.gov/awardsearch/show-award/?AWD_ID=2540315)) covers tutoring agents, instructor analytics, and orchestration; this project studies one small slice of it.
- **Observed.** In experiment E2 (Section 9), with 10 of 100 queue messages lost, a queue-only design left 10 events unanalyzed and silently missed 2 of 5 flags in 10 of 10 trials. Adding the reconciliation sweep recovered all 100 events and all 5 flags in 10 of 10 trials. In E1, a worker that counted messages instead of saved events created false flags whenever a message was repeated after a crash.
- **Assumed.** That instructors want this review task (A1), and that real RabbitMQ redelivers as E1 simulated (A6).
 
**Six-week contribution:** One working path and one reliability property. A simulator sends a fictional event to an intake service, which stores it and returns a fixed tutor reply. A queue carries it to one analysis worker that applies rule R1. A minimal dashboard shows each event, whether it was analyzed, and any flag with its reason. A reconciliation sweep re-sends any accepted event that has not been analyzed. The property: every accepted event is eventually analyzed and appears on the dashboard exactly once, even when its message is lost, delayed, or repeated.
 
| Week | Deliverable | Evidence |
|---|---|---|
| 1 | Event contract, rule R1, fixtures, feasibility experiments E1 and E2 | Section 2 fixtures; E1 and E2 results (done) |
| 2 | Intake service stores events and returns fixed replies | T1, T8 pass |
| 3 | Outbox publisher, queue, duplicate-safe worker, and reconciliation sweep | T2, T3, T10 pass |
| 4 | Dashboard shows records, status, and flags | T1, T9 pass |
| 5 | Worker outage and crash tests | T4, T5 pass |
| 6 | 100-event reconciliation and timing | T6, T7 recorded |
 
**Semester II scope:** Only after the six-week path passes: several workers on one queue, larger load tests, a measured comparison with the single-application baseline, a retry and dead-letter policy, and confirming the instructor task (A1). Optional: an LLM-based tutor or analyzer added as a second consumer of the same events.
 
**Exclusions:** LLM tutoring or analysis in Semester I; code execution; judging whether a student has learned or is struggling; comparing rule-based with LLM analysis; real authentication; real student data; and failure of the database or queue themselves. Learning-effect questions would need real classroom data and privacy review.
 
## 2 Requirements and system specification
 
| ID | Required behavior and conditions | Acceptance check | Design and test reference |
|---|---|---|---|
| F1 | A valid event sent to the intake service is stored once and answered with HTTP 202 and a fixed tutoring reply. | Send fixture event E1. Expect 202, one row in `events`. | Section 5 `POST /events`; T1 |
| F2 | Every accepted event is delivered to the analysis worker and marked analyzed. | After fixture F-A, `processed_events` has 3 rows for S1 within 5 seconds. | Worker in Appendix A; T1 |
| F3 | The worker applies rule R1 exactly as defined below. | Fixtures F-A to F-E produce exactly the expected flag counts in the fixture table below. | Worker in Appendix A; T2 |
| F4 | The dashboard shows every accepted event with status `pending analysis` or `analyzed`, and each flag with its reason text. | For fixture F-A: 3 event rows, all `analyzed`, 1 flag with reason. | Section 5 Dashboard; T1 |
| F5 | Invalid events are rejected with HTTP 400 and leave no stored data. | Send events missing `event_id`, with a bad `event_type`, and with `text` over 2,000 characters. Expect 3 rejections and 0 new rows. | Section 5 `POST /events` errors; T8 |
| F6 | **Lost-event retrieval.** Any accepted event that has not been analyzed 30 seconds after acceptance is found by a reconciliation sweep, which reads the saved `events` records and sends the event again. | Drop the queue message for E3 (fixture F-J). Within 60 seconds the sweep finds E3, E3 is analyzed, and S1 is flagged. | Reconciler (Sections 4, 5); T10; E2 |
| C1 | A flag is information for review. No page, field, or log may say a student cheated or is struggling. | Every flag reason contains the exact sentence "Review suggested. This is not a finding of misconduct." The dashboard source and database schema contain no field or label named cheating, struggling, or similar. | Section 5 Dashboard; T9 |
| C2 | Sending the same `event_id` repeatedly never creates more than one stored event, one processed record, or one flag. | Retry E2 five times. `events` still has 3 rows for S1 and the flag count is still 1. | Unique keys in Appendix A `schema.sql`; T3; E1 |
| Q1 | For a set of 100 known accepted events, the dashboard shows each exactly once: 0 missing and 0 repeated. (Proposed target, not yet measured) | Reconcile the submitted ID list against dashboard rows (fixture F-G). | T6 |
| Q2 | When the worker is stopped, accepted events are kept. After the worker restarts, every kept event reaches `analyzed` within 60 seconds with no loss. (Proposed target, not yet measured) | Fixture F-A while the worker is stopped: 3 accepted, dashboard shows 3 `pending analysis` and 0 flags. After restart, within 60 seconds: 3 `analyzed` and 1 flag. | T4 |
| Q3 | Intake keeps accepting events while the worker is stopped. 95th-percentile response time is at most 500 ms with 10 concurrent simulated clients sending 100 events. Normal-path delay from acceptance to `analyzed` is at most 5 seconds at the 95th percentile. (Proposed target, not yet measured) | Time each request and each acceptance-to-analyzed interval in fixture F-G. | T7 |
| Q5 | **Recovery from lost messages.** With 10 of 100 queue messages lost and 5 delayed, all 100 accepted events are analyzed within 60 seconds of the last acceptance, with 0 duplicate records and exactly the 5 expected flags. (Counts observed in E2 with a simulated queue; the 60-second timing is a proposed target, not yet measured.) | Fixture F-J: compare accepted, analyzed, and flagged students against the expected lists. | T10; E2 |
| Q4 | If analysis stops after saving its result but before reporting that it finished, repeating that event's analysis changes nothing. | Fixtures F-H and F-I: stop the worker at that point and restart. F-H: exactly one E17 entry and one flag. F-I: two entries and no flag. | T5; E1 (worker logic passed, broker simulated) |
 
**Planned tests referenced above.** These names tie each requirement to a test. Their full setup and procedures will be written in Section 10 (Assignment 4).
 
| Test | Checks | Fixture |
|---|---|---|
| T1 | Normal path: events accepted, analyzed, and shown | F-A |
| T2 | Rule R1 flag counts, including both window boundaries | F-A to F-E |
| T3 | Repeated `event_id` creates no extra record or flag | F-F |
| T4 | Worker stopped: events kept, analyzed after restart | F-A |
| T5 | Worker crash after commit, before acknowledgement | F-H, F-I |
| T6 | 100-event reconciliation: 0 missing, 0 repeated | F-G |
| T7 | Response time and acceptance-to-analyzed delay | F-G |
| T8 | Invalid input rejected with no stored data | Invalid events (F5) |
| T9 | Flag wording and dashboard access | F-A |
| T10 | Lost messages found and recovered by the sweep; delayed duplicates ignored | F-J |
 
**Inputs and outputs:**
 
*Ordinary example.* Fictional student S1 sends three full-solution requests within ten minutes (fixture F-A). Each is accepted with HTTP 202 and a fixed tutor reply. The worker processes all three and creates one review flag for S1. The dashboard shows 3 `analyzed` events and 1 flag reading "Review suggested. This is not a finding of misconduct. Rule R1: 3 or more full-solution requests within 600 seconds."
 
*Unsuccessful examples.* (1) An event missing `event_id` is rejected with HTTP 400 and nothing is stored. (2) A student who asks five ordinary questions (fixture F-C) must **not** be flagged; flagging them would be a false positive. (3) If the worker crashes after saving S7's second request and the message is redelivered (fixture F-I), a naive design flags S7 for three requests when only two were made. The required behavior is no flag.
 
The event format, the exact rule, and all fixtures follow.
 
### Event contract
 
Every event is one JSON object submitted by the simulator or student interface.
 
| Field | Type and allowed values | Meaning | Example |
|---|---|---|---|
| `event_id` | string, UUID or fixed fixture ID, 1 to 64 characters | Unique key chosen by the sender. Reusing it means "this is a retry." | `"E2"` |
| `student_id` | string, 1 to 32 characters | Fictional student identifier | `"S1"` |
| `session_id` | string, 1 to 32 characters | One working session | `"sess-1"` |
| `occurred_at` | ISO 8601 UTC timestamp, whole seconds | When the student sent the request | `"2026-01-15T10:04:00Z"` |
| `event_type` | `"question"` or `"full_solution_request"` | Label supplied by the simulator. The prototype does not classify text. | `"full_solution_request"` |
| `text` | string, 1 to 2,000 characters | The student's message | `"Write the whole function for me"` |
 
**Accepted** means the intake service returned HTTP 202 after the event row and its outbox row were committed in one database transaction. Everything in this document that says "accepted event" uses this definition.
 
### Rule R1 (the exact meaning of "repeated")
 
> For a student and a session, raise a **review flag** when at least **3 distinct** events with `event_type = "full_solution_request"` have `occurred_at` values within a window of **600 seconds, inclusive** (last timestamp minus first timestamp is 600 or less). At most one R1 flag exists per `(student_id, session_id)`.
 
Two implementers following this text must give the same result on every fixture below. The rule uses `occurred_at`, not arrival time, so retries and delays do not change the result. Events of type `question` are never counted. The threshold and window are placeholders that make the rule testable; they are not an educational finding and may change after the instructor interview (A1).
 
### Fixtures and exact expected results
 
All timestamps below are on 2026-01-15 UTC. "Full" means `full_solution_request`.
 
| Fixture | Students and events | Expected flags | Expected stored events |
|---|---|---|---|
| F-A ordinary flag | S1: E1 10:00:00, E2 10:04:00, E3 10:08:00, all Full | 1 flag for S1 | 3 |
| F-B below threshold | S2: E4 10:00:00 and E5 10:05:00 Full, E6 10:11:00 Full (E4 to E6 spans 660 s, so only 2 within any 600 s window) | 0 flags | 3 |
| F-C genuine learner (the unsuccessful-flag case) | S3: five events E7 to E11 of type `question` within 10 minutes | 0 flags | 5 |
| F-D boundary included | S4: E12 10:00:00, E13 10:05:00, E14 10:10:00, all Full (span exactly 600 s) | 1 flag for S4 | 3 |
| F-E boundary excluded | S5: E15 10:00:00, E16 10:05:00, E18 10:10:01, all Full (span 601 s) | 0 flags | 3 |
| F-F duplicate retry | Run F-A, then resend E2 five times unchanged | 1 flag for S1 | 3 (not 8) |
| F-H crash point | S6: E19 10:00:00, E20 10:04:00, E17 10:08:00, all Full, sent in that order so E17 is processed last and triggers the flag. The worker is stopped right after committing E17's result and before acknowledging it. | 1 flag for S6 | 3, with exactly one E17 entry |
| F-I crash on a non-triggering event | S7: E21 10:00:00 and E22 10:04:00, both Full. The worker is stopped right after committing E22's result and before acknowledging it. A correct system must **not** flag S7, who made only two requests. | 0 flags | 2 |
| F-J lost and delayed messages | F-G's 100 events. Ten queue messages are dropped and never delivered, including the flag-triggering third request of G-S1 (G003) and G-S2 (G008). Five other messages are delayed until after the sweep has run, so each arrives twice. | 5 flags (G-S1 to G-S5) | 100, all analyzed once |
| F-G reconciliation set | 20 students, 5 events each, 100 distinct IDs (`G001` to `G100`). Students G-S1 to G-S5 each have 3 Full events in a 600 s window plus 2 `question` events. The other 15 students have only `question` events. Ten randomly chosen IDs are resent. | 5 flags | 100 |
 
Event IDs are labels, not sequence numbers. E17 is reserved for the crash fixture F-H because that is the event named in earlier feedback and in test T5.
 
**Assumptions:**
 
| ID | Condition I rely on | How I will check it |
|---|---|---|
| A1 | Instructors want to review R1 flags and the history behind them | Interview one instructor or TA with the dashboard mock-up |
| A2 | The sender reuses `event_id` when it retries, so `event_id` works as the duplicate key | T3 resends E2 five times; the simulator logs every `event_id` it sends |
| A3 | PostgreSQL and RabbitMQ stay available during tests | Health checks before each test; a test run is discarded if either was down |
| A4 | Fixed tutor replies are enough for testing delivery | Not checked; tutor quality is outside the question (Exclusions) |
| A5 | Simulator labels (`event_type`) are correct | Fixture files are reviewed by hand; labeling real text is excluded |
| A7 | 30 seconds is longer than normal processing takes, so the sweep rarely re-sends events that are merely slow | Compare with acceptance-to-analyzed times measured in T7 |
| A6 | Real RabbitMQ redelivers an unacknowledged message after a worker crash, as E1 simulated | Experiment E3 with RabbitMQ in Docker |
 
**Constraints:**
 
- **Time:** six weeks for the working path, as one student.
- **Tools:** Python, PostgreSQL, RabbitMQ, and a small web dashboard, all run locally with Docker Compose.
- **Data:** fictional records only. No real student data.
- **Resources:** one laptop; no paid services; no LLM calls in Semester I.
- **Evaluation:** fixtures test delivery behavior and rule correctness. They cannot show that flags identify real learning difficulty or improve outcomes.
 
## 3 Alternatives and selected solution
 
Both approaches are judged against the same requirements (F1 to F5, C1, C2, Q1 to Q4) and fixtures (F-A to F-I).
 
| Approach and source | Useful behavior | Limitation | Evidence or assumption |
|---|---|---|---|
| **Simple baseline: A. Single application.** One Python app does intake, rule analysis, and the dashboard. It saves each event as `pending`, runs R1 in a background thread, and rescans `pending` rows every 10 s. Basis: rule-based analytics as in [Moodle](https://docs.moodle.org/502/en/Using_analytics). | Fewest parts. Easy to run and debug. A unique key on `event_id` blocks duplicate events. The rescan recovers from an analysis crash. | Analysis shares the intake process, so a slow or crashing analyzer can slow or stop intake (Q3 risk). Recovery waits for the next scan. | **Assumed:** may meet Q1, Q2, Q4. **Observed (E1):** the duplicate guard lives in the database, so A can use it too. |
| **Candidate alternative: B. Separate services with a queue.** Intake service, RabbitMQ, analysis worker, dashboard. Intake saves the event and an outbox row together; a publisher sends outbox rows to the queue; the worker saves its result and a `processed_events` row together, then acknowledges; a reconciliation sweep re-sends any saved event that has not been analyzed after 30 s. Basis: [transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html), [RabbitMQ reliability guide](https://www.rabbitmq.com/docs/reliability), [PostgreSQL `ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html). | Intake does not depend on analysis speed or availability. The worker can be stopped, replaced, or multiplied. An LLM analyzer can later be added as another consumer without changing intake. | Three programs plus a queue. The queue may deliver a message twice, so the worker must be duplicate-safe. More to configure. | **Observed (E2):** the sweep recovered 10 of 10 lost events in 10 of 10 trials. **Observed (E1):** duplicate-safe worker correct in 60 of 60 crash and concurrent trials. **Assumed:** intake stays fast during an outage (Q3); real RabbitMQ redelivers as simulated (A6). |
 
The same comparison by requirement. Entries are expectations unless marked observed.
 
| Criterion (requirement) | A. Single application | B. Services and queue | Evidence so far |
|---|---|---|---|
| Lost queue message is found and re-sent; all 100 events analyzed, 5 flags (F6, Q5, fixture F-J) | Expected pass: its rescan of `pending` rows is also a sweep | **Observed pass** for sweep logic, queue simulated; queue-only design failed | E2, 10/10 with sweep, 0/10 without |
| Retrying E2 five times leaves 3 events and 1 flag (C2) | Expected pass | Expected pass | Planned T3 |
| Crash after saving, before reporting completion: counts stay correct (Q4) | Expected pass **if** it uses the same database guard | **Observed pass**, worker logic, queue simulated | E1, 20/20 per fixture |
| Same message processed twice at once (C2) | Not applicable (one analysis thread) | **Observed pass** | E1, 20/20 |
| Worker stopped: 0 lost, all analyzed within 60 s of restart (Q2) | Expected pass; depends on scan interval | Expected pass; depends on queue redelivery | Planned T4 on both |
| p95 accept time at most 500 ms while analysis is stopped (Q3) | **At risk** | Expected pass | Planned T7 on both |
| Complexity | 1 program, 1 store | 3 programs, 2 stores | Counted at end of week 6 |
 
**Choice and reason:** I select **B, and keep A as a measured baseline.**
 
1. **Q3 is what separates the designs.** E1 showed duplicate safety (C2, Q4) comes from a database guard either design can use, so it does not justify extra services. Keeping intake fast while analysis is stopped or slow does: in A, analysis shares the intake process; in B, it does not.
2. **The planned LLM analyzer will be slow and unreliable by nature** (network calls, rate limits, outages). B lets it be added as a second consumer without touching intake.
3. **B's added risks are controlled.** A queue can lose a message or deliver it twice. E2 showed the reconciliation sweep retrieves lost events from the saved records, and E1 and E2 showed the guarded worker ignores repeats. Neither result favors B over A by itself, because A's rescan is also a sweep. What they show is that B can be made as safe as A while keeping intake independent of analysis.
4. **The time constraint has a fallback.** If B is not passing T1 to T3 by the end of week 3, I build A with the same database guard and report A-only results.
 
The two decision records below follow the usual architecture-decision-record elements of context, options considered, outcome, and tradeoffs, with a status ([Microsoft, Architecture decision record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record)).
 
### Decision 1
 
**Context and requirement:** Students' events must be accepted whether or not analysis is running (Q2, Q3), and analysis will later include slow LLM calls. Should intake and analysis run in the same program?
 
**Options:** (A) one application with a status column and rescan; (B) intake service, durable queue, separate worker.
 
**Decision and reason:** B, with A built as a baseline. B separates the step most likely to be slow or down (analysis) from the step that must stay up (intake), and E1 showed its added duplicate risk is manageable.
 
**Consequences:** More parts to run and test. An outbox is needed so an accepted event cannot be lost between saving and publishing. The dashboard may show an event as `pending analysis` for a while: it catches up eventually rather than instantly.
 
**Status and revisit condition:** Accepted for the prototype. Revisit if T7 shows A also meets Q3, if B is not passing T1 to T3 by week 3, or if E3 shows real RabbitMQ differs from E1's simulation.
 
### Decision 2
 
**Context and requirement:** Accepted events must never be lost (Q1, F6, Q5) and never counted twice (C2, Q4). In design B, a queue message can be lost (dropped, or the publisher stops after saving but before sending) or delivered more than once (redelivery after a crash, or a late original after a re-send). How should lost events be retrieved, and repeats handled?
 
**Options:** (1) trust the queue alone, so whatever it loses stays lost; (2) queue redelivery plus a worker that keeps a running count per student; (3) treat the saved `events` table as the source of truth: an outbox re-sends unpublished rows, a **reconciliation sweep** re-sends any accepted event with no analysis record after 30 s, and the worker records `event_id` in `processed_events` in the same transaction as its result so repeats have no effect.
 
**Decision and reason:** Option 3. In E2, option 1 left 10 of 100 events unanalyzed and missed 2 of 5 flags in every trial. With the sweep, all 100 events and all 5 flags were recovered in 10 of 10 trials, and the 5 events that then arrived twice were ignored. In E1, option 2 was wrong in 40 of 40 crash trials, including flagging a student with only two requests, while option 3's worker was correct in all 60 crash and concurrent trials.
 
**Consequences:** Retrieval depends on the saved record: an event must be in `events` before its message is sent, and kept at least until it is analyzed. The sweep deliberately creates repeats, so every worker write must be inside the transaction that records `processed_events`, and running counters are not allowed. The sweep adds one database query every 10 s. A threshold that is too short would re-send events that are merely slow (A7).
 
**Status and revisit condition:** Accepted, supported by E1 and E2 with a simulated queue. Revisit if E3 shows real RabbitMQ behaving differently, if T7 shows normal processing often takes longer than 30 s, or if a later consumer cannot write its result in the same transaction as `processed_events`.
 
## 4 Architectural design
 
The architecture is shown in three views: **context** (what is inside and outside the system), **containers** (the running parts, their messages, and saved state), and **deployment** (where each part runs). The context and container views follow the C4 model's system context and container levels ([C4 model](https://c4model.com/diagrams)). The context and deployment views are plain-text diagrams, each with a caption; the container view is a table.
 
**Distributed systems concepts this design relies on**
 
| Concept | Plain meaning | How it shapes this design |
|---|---|---|
| Partial failure | One part fails while others keep running | The worker can stop while intake keeps accepting events, so events wait in the queue (Q2, Q3) |
| At-least-once delivery | A message is redelivered until acknowledged, so it may arrive twice | Nothing accepted is lost, but the worker must handle repeats (Decision 2) |
| Idempotence | Repeating an action has no extra effect | `event_id` is the duplicate key at intake (200 on retry) and in the worker (`processed_events`) |
| Atomic transaction | Related writes are saved together or not at all | Event + outbox row together; result + `processed_events` row together |
| Eventual consistency | A view catches up after a delay | The dashboard may show `pending analysis` until the worker finishes |
| Reconciliation (anti-entropy) | Periodically compare what should exist with what does, and repair the difference | The sweep compares `events` (what was accepted) with `processed_events` (what was analyzed) and re-sends the difference. The saved record, not the queue, is the source of truth |
 
**A lost event traced through the parts (G003, fixture F-J).** (1) The simulator posts G003; intake saves it with an outbox row and replies 202, so G003 is accepted. (2) The publisher sends `{"event_id":"G003"}`, but the message is lost before reaching the worker. (3) The dashboard shows G003 as `pending analysis` and G-S1 has no flag. (4) On the next sweep after 30 s, the reconciler finds G003 has no `processed_events` row and re-sends it from the saved record (8a, 8b). (5) The worker analyzes G003, finds G-S1's third full-solution request within 600 s, and saves the flag. (6) If the original G003 message was only delayed and arrives now, the worker finds it already processed and writes nothing.
 
**A repeated event traced through the parts (E17, fixture F-H).** (1) The simulator posts E17 and intake saves it with an outbox row, then replies 202. (2) The publisher sends `{"event_id":"E17"}` to the queue. (3) The worker saves E17 in `processed_events`, finds three full-solution requests from S6 within 600 s, and saves one flag, all in one transaction. (4) If the worker crashes before acknowledging, the queue delivers E17 again; the worker sees it already processed and writes nothing. (5) The dashboard shows E17 once, as `analyzed`, and one flag for S6.
 
### Boundary and outside participants (context view)
 
```text
 +--------------------+   HTTP POST /events      +--------------------------------+
 | Simulator script   | -----------------------> |  MONITORING PROTOTYPE          |
 | (stands in for     | <----------------------- |  (system boundary)             |
 |  students)         |   202 + fixed reply      |  accepts events, applies R1,   |
 +--------------------+                          |  re-sends lost events,         |
                                                 |  shows review flags            |
 +--------------------+   HTTP GET /dashboard    |                                |
 | Instructor or TA   | -----------------------> |                                |
 | (web browser)      | <----------------------- |                                |
 +--------------------+   events, status, flags  +--------------------------------+
                                                      :                 :
                                    not connected in  :                 :  not connected in
                                    Semester I        v                 v  Semester I
                                           +---------------+   +--------------------+
                                           | LLM API       |   | Code evaluation    |
                                           | (deferred)    |   | (deferred)         |
                                           +---------------+   +--------------------+
```
 
*Figure 1. Context view. A simulator standing in for students sends events to the prototype and receives a fixed tutor reply; the instructor's browser reads the dashboard. Dotted lines lead to the LLM API and code-evaluation service, which are not connected in Semester I.*
 
| Outside participant | Role | Crosses the boundary by |
|---|---|---|
| Simulator script | Stands in for students and the student interface. Sends fictional fixture events, repeats some, and stops and restarts the worker during tests. | HTTP POST `/events` |
| Instructor or TA | Reads which events were analyzed and which students have review flags, with the reason for each flag. | HTTP GET `/dashboard` in a browser |
| LLM API, code-evaluation service | Part of the larger NSF vision. **Not connected** in this prototype. | Nothing in Semester I |
 
The boundary excludes real students, real student data, authentication beyond a test token, and any AI service. Everything inside the boundary is built, reused, or simulated as listed under What I build, reuse, or simulate.
 
### Main parts and messages (container view)
 
*Table: Main parts and messages for one event, in order. Steps 3b and 4 may repeat. Steps 2 and 5 are all-or-nothing database writes. Steps 8a and 8b are the lost-event retrieval path.*
 
| Step | From | To | Message or action | Saved state afterward | Can repeat or fail? |
|---|---|---|---|---|---|
| 1 | Simulator (simulated) | Intake API (build) | HTTP POST `/events` with the event JSON | None yet | Sender may retry with the same `event_id` |
| 2 | Intake API | PostgreSQL (reuse) | SQL: insert the event and its outbox row in one transaction; reply 202 | `events` + `outbox` row; the event is now **accepted** | A repeat finds the existing row and replies 200 |
| 3a | Outbox publisher (build) | PostgreSQL | SQL: read outbox rows not yet published | No change | Runs every 1 s |
| 3b | Outbox publisher | RabbitMQ (reuse) | AMQP publish `{"event_id": ...}`; mark the row published | Message in the durable queue; `published_at` set | May publish twice; the message may be lost |
| 4 | RabbitMQ | Analysis worker (build) | AMQP delivery with manual acknowledgement | Message held until acknowledged | Redelivered after a worker crash |
| 5 | Analysis worker | PostgreSQL | SQL: insert `processed_events` row and any flag in one transaction, then acknowledge | `processed_events` row; maybe a `flags` row | A repeat finds the `processed_events` row and writes nothing |
| 6 | Instructor browser | Dashboard (build) | HTTP GET `/dashboard` | No change | Safe to repeat |
| 7 | Dashboard | PostgreSQL | SQL: read events, status, and flags (read only) | No change | Safe to repeat |
| 8a | Reconciler (build) | PostgreSQL | Every 10 s, SQL: find accepted events with no `processed_events` row after 30 s | No change | Runs every 10 s |
| 8b | Reconciler | RabbitMQ | AMQP re-publish each lost `event_id` | New message in the queue | A late original is ignored at step 5 |
 
**Responsibilities**
 
| Part | Responsibility | Does not |
|---|---|---|
| Intake API | Validates an event against the contract (Section 2), stores it once, returns a fixed tutor reply | Analyze, or contact the broker directly |
| Outbox publisher (thread in the intake program) | Sends each unpublished outbox row to the broker; marks it published after the broker confirms | Decide anything about the event's content |
| Reconciler (thread in the intake program) | Every 10 s, finds accepted events that have no `processed_events` row 30 s after acceptance, and re-sends them. This is how a lost message is retrieved from the saved record. | Decide whether an event is a duplicate; the worker does that |
| RabbitMQ | Holds messages until a worker acknowledges them; redelivers unacknowledged messages | Guarantee single delivery |
| Analysis worker | Applies R1 once per event, writing results and the `processed_events` row together | Write to `events` or `outbox` |
| Dashboard | Shows each accepted event, its derived status, and flags with reasons | Write anything |
| PostgreSQL | Authoritative saved state (below) | Run application logic |
 
**Communication**
 
| Step | From → to | Protocol and content | Sync or async | If the receiver is down |
|---|---|---|---|---|
| 1 | Simulator → Intake API | HTTP POST, event JSON | Synchronous | Simulator gets a connection error or 503 and retries with the same `event_id` |
| 2 | Intake API → PostgreSQL | SQL transaction: insert event and outbox row | Synchronous | 503 to simulator; event **not** accepted |
| 3a, 3b | Publisher → PostgreSQL, Publisher → RabbitMQ | SQL read; AMQP publish of `{"event_id": ...}` with publisher confirms | Asynchronous to the student | Row stays unpublished and is retried every 1 s |
| 4 | RabbitMQ → Worker | AMQP delivery with manual acknowledgement | Asynchronous | Message waits in the queue |
| 5 | Worker → PostgreSQL | SQL transaction: insert `processed_events` and any flag | Synchronous for the worker | Worker does not acknowledge; message is redelivered |
| 6, 7 | Browser → Dashboard → PostgreSQL | HTTP GET; SQL read | Synchronous | Dashboard shows an error page; nothing is lost |
| 8a, 8b | Reconciler → PostgreSQL, Reconciler → RabbitMQ | SQL query for unanalyzed events older than 30 s; AMQP re-publish of `{"event_id": ...}` | Asynchronous, every 10 s | Queue down: the event stays unanalyzed and the next sweep tries again |
 
**Saved state** (what survives a restart of each part)
 
| Where | What is saved | Owner | Survives |
|---|---|---|---|
| PostgreSQL `events` | Every accepted event | Intake | Restart of any container (volume `pgdata`) |
| PostgreSQL `outbox` | Which accepted events have been published | Intake | Same |
| PostgreSQL `processed_events`, `flags` | Which events were analyzed, and R1 results | Worker | Same |
| RabbitMQ queue | Published but unacknowledged messages (persistent) | Broker | Broker restart (volume `mqdata`) |
| Intake, publisher, worker, dashboard processes | **Nothing.** They hold no state that must survive. | — | They can be killed at any time; state is rebuilt from the stores |
 
Because the four programs keep no saved state, a crash of any of them is recovered by restarting it. That is the property E1 tested for the worker.
 
### Where parts run (deployment view)
 
```text
 ONE LAPTOP
 +-------------------------------------------------------------------------+
 |  Host: simulator + tests (Python)          Host: browser                |
 |        |                                         |                      |
 |  DOCKER COMPOSE                                  |                      |
 |  +-----v------------------+            +---------v---------+           |
 |  | intake  :8000          |            | dashboard  :8080  |           |
 |  | (+ publisher,          |            +---------+---------+           |
 |  |    reconciler)         |                      |                     |
 |  +----+-------------+-----+                      |                     |
 |       |             |                            |                     |
 |  +----v-----+  +----v-------------+              |                     |
 |  | rabbitmq |  | postgres  :5432  | <------------+                     |
 |  | :5672    |  |                  | <------------+                     |
 |  +----+-----+  +--------+---------+              |                     |
 |       |                 |               +--------+---------+           |
 |       +---------------------------------> worker           |           |
 |       |                 |               | (restarted by    |           |
 |  [volume mqdata]   [volume pgdata]      |  tests)          |           |
 |   survives restart  survives restart    +------------------+           |
 +-------------------------------------------------------------------------+
```
 
*Figure 2. Deployment view. One laptop runs the simulator and browser on the host and five containers under Docker Compose; the pgdata and mqdata volumes keep PostgreSQL and RabbitMQ data across container restarts.*
 
- **One machine.** All parts run on one laptop under Docker Compose, so there is no real network failure between machines. That is an excluded condition (Section 8, Assignment 4).
- **Shared dependencies.** Every program depends on PostgreSQL. Intake's publisher and the worker depend on RabbitMQ. If PostgreSQL is down, nothing can be accepted (503).
- **Separate containers on purpose.** The tests stop and restart only the worker container (T4, T5) while intake keeps running.
- **Ports** are fixed so test scripts can reach intake (8000) and the dashboard (8080).
 
### What I build, reuse, or simulate
 
| Item | Build, reuse, or simulate | Notes |
|---|---|---|
| Intake API and outbox publisher | **Build** | Python web framework (FastAPI or Flask, to be chosen in week 2) |
| Reconciler | **Build** | Sweep logic tested in E2 |
| Analysis worker | **Build** | E1's [`worker.py`](#appendix-a-experiments-e1-and-e2-code-and-evidence) is the first version of its core logic |
| Dashboard | **Build** | One server-rendered HTML page |
| Simulator and fixture files | **Build** | Plays students and injects failures |
| PostgreSQL 16 | **Reuse** | Official image; used for real in E1 |
| RabbitMQ 3.13 | **Reuse** | Official image; **not yet tested** (E3) |
| psycopg 3, an AMQP client (pika) | **Reuse** | Python libraries |
| Students and their messages | **Simulate** | Fictional fixtures F-A to F-I |
| AI tutor replies | **Simulate** | Fixed text |
| Broker redelivery in E1 | **Simulate** | The E1 harness resends unacknowledged messages; E3 will use real RabbitMQ |
| Lost and delayed messages in E2 | **Simulate** | The E2 harness drops or holds back chosen messages; the sweep and worker logic are real |
| LLM, code evaluation, real authentication | **Deferred** | See Section 1 scope |
 
## 5 Component design and interfaces
 
| Component | Responsibility | Input | Output or state change | Requirements served |
|---|---|---|---|---|
| Intake API | Validate an event, store it exactly once, answer the sender | HTTP POST `/events` with event JSON and `X-Sender-Token` | Inserts one `events` row and one `outbox` row (new event only); returns 202, 200, 400, 401, 409, or 503 | F1, F5, C2, Q3 |
| Outbox publisher | Move accepted events onto the queue | `outbox` rows where `published_at` is empty | Message `{"event_id": ...}` on `events.to_analyze`; sets `published_at` after broker confirm | F2, Q1 |
| Reconciler | Find accepted events that were never analyzed and send them again | Timer every 10 s; saved `events` and `processed_events` records | Re-publishes each unanalyzed event older than 30 s; logs the IDs it re-sent; no table changes | F6, Q1, Q5 |
| Analysis worker | Apply R1 once per event | Delivered message `{"event_id": ...}` | Inserts one `processed_events` row; may insert one `flags` row; acknowledges message | F2, F3, C2, Q2, Q4 |
| Dashboard | Show events, derived status, and flags to an instructor | HTTP GET `/dashboard` with instructor token | HTML page; no state change | F4, C1 |
| Simulator | Send fixtures, resend duplicates, stop and restart the worker, record timings | Fixture files | HTTP requests; test log and timing file | All tests |
 
**Critical operation:** `POST /events`, which accepts one student-AI interaction event. Its purpose is to durably accept each event exactly once. It defines when an event becomes **accepted**. Every later guarantee (Q1: nothing lost; Q2: kept during an outage) applies only to accepted events. It directly implements **F1** (store once and reply), **F5** (reject invalid input with no stored data), and **C2** (a repeated `event_id` never creates a second record).
 
**Input:** One JSON body with the six fields of the event contract (Section 2). **Caller identity:** the simulator (later, the student interface), identified by header `X-Sender-Token`. In the prototype this is one fixed test token. Per-student authentication is deferred (Section 8, Assignment 4).
 
| Field | Type and allowed values | Required |
|---|---|---|
| `event_id` | string, 1 to 64 characters | Yes |
| `student_id` | string, 1 to 32 characters | Yes |
| `session_id` | string, 1 to 32 characters | Yes |
| `occurred_at` | ISO 8601 UTC timestamp, whole seconds | Yes |
| `event_type` | `"question"` or `"full_solution_request"` | Yes |
| `text` | string, 1 to 2,000 characters | Yes |
 
Example input (successful):
 
```http
POST /events
X-Sender-Token: test-token
Content-Type: application/json
 
{"event_id":"E2","student_id":"S1","session_id":"sess-1",
 "occurred_at":"2026-01-15T10:04:00Z","event_type":"full_solution_request",
 "text":"Write the whole function for me"}
```
 
**Output:**
 
| Field | Meaning |
|---|---|
| HTTP status | 202 new and accepted; 200 already accepted with identical content; 400, 401, 409, 503 errors (below) |
| `event_id` | Echo of the input key |
| `status` | `"accepted"` for 202 and 200; otherwise `"rejected"` |
| `tutor_reply` | Fixed text for 202 and 200 |
| `error` | For 4xx and 5xx: a short code and the field name, if any |
 
Example output for the request above, as a new event:
 
```http
HTTP/1.1 202 Accepted
{"event_id":"E2","status":"accepted","tutor_reply":"Try tracing the loop by hand first."}
```
 
The same request sent again (a retry after a timeout) returns:
 
```http
HTTP/1.1 200 OK
{"event_id":"E2","status":"accepted","tutor_reply":"Try tracing the loop by hand first."}
```
 
**State changes:**
 
| Case | `events` | `outbox` | Queue |
|---|---|---|---|
| 202 new event | +1 row | +1 row (`published_at` empty) | Nothing yet; the publisher sends it within about 1 s |
| 200 same ID, same content | No change | No change | No change |
| 400, 401, 409, 503 | No change | No change | No change |
 
Both inserts in the 202 case happen in **one transaction**, so there is never an `events` row without its `outbox` row.
 
After the first request, `events` has E2 and `outbox` has E2 unpublished. After the retry, nothing has changed and there is still one E2 row. This is the C2 check in T3.
 
**Errors:** Unsuccessful examples and the appropriate caller action for each:
 
| Request | Response | State change | Caller action |
|---|---|---|---|
| Same body as above but `"event_type":"give_me_code"` | `400 {"status":"rejected","error":"invalid_value","field":"event_type"}` | None | Fix the event; do not retry unchanged |
| Body with no `event_id` | `400 {"status":"rejected","error":"missing_field","field":"event_id"}` | None | Fix the event |
| `event_id` `"E2"` again but with `"text":"Explain loops"` | `409 {"status":"rejected","error":"id_reused_with_different_content"}` | None; original E2 kept | Sender bug: use a new `event_id` for new content |
| Valid body, header `X-Sender-Token: wrong` | `401 {"status":"rejected","error":"unauthorized"}` | None | Fix the token |
| Valid body while PostgreSQL is stopped | `503 {"status":"rejected","error":"store_unavailable"}` | None; transaction rolled back | Retry later with the **same** `event_id` |
| Valid body, but no response arrives before the sender's timeout | None received; **outcome unknown** to the sender | Event may or may not have been saved | Retry with the **same** `event_id`: a 202 or 200 then confirms it is accepted, never twice |
 
**Repeated or overlapping calls:** Using a client-chosen request identifier so a retry is recognized as the same action follows Amazon's guidance on idempotent APIs ([Amazon Builders' Library](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)). Two identical requests for E2 arriving at the same moment both try `INSERT ... ON CONFLICT (event_id) DO NOTHING`. PostgreSQL lets one insert succeed (202). The other sees the conflict and returns 200. E1 tested this same database mechanism for the worker's `processed_events` table (20 of 20 concurrent trials correct). It has not yet been tested for the intake endpoint (planned T3).
 
**Details still being worked out:**
 
| Open detail | Current proposal | Why it is not final |
|---|---|---|
| What "identical content" means for 200 versus 409 | Exact match of all six fields, compared as stored | Unclear whether whitespace or timestamp format differences should count as different content |
| Timeout ambiguity: the sender cannot tell whether a timed-out request was committed | Sender always retries with the same `event_id`; the 200 path makes this safe | Retry limit and backoff for the simulator not yet chosen |
| `occurred_at` far in the future or past | Accept any valid timestamp | A wrong clock could create or hide R1 flags; a limit such as "no more than 5 minutes ahead of server time" is under consideration |
| Request size and rate limits | 2,000-character text limit only | No rate limit yet; needed before any real use |
| Event stuck because the broker is unreachable for a long time | Outbox retries every 1 s indefinitely | No alert yet when unpublished rows grow (Section 12 open questions) |
 
The worker's operation was tested in E1. Its code is in [Appendix A](#appendix-a-experiments-e1-and-e2-code-and-evidence), and its pseudocode will be written in Section 7 (Assignment 4).
 
### Lost-event retrieval: `RECONCILE` (reconciliation sweep)
 
This second operation is how the design retrieves a lost event. It implements **F6** and supports **Q1** and **Q5**.
 
| Part of the contract | Definition |
|---|---|
| Trigger and caller | A timer inside the intake program, every 10 s. No outside caller. |
| Input | The saved records: every row in `events`, and every row in `processed_events`. |
| Selection rule | Events in `events` that have **no** `processed_events` row and were received more than **30 s** ago. |
| Output | One queue message `{"event_id": ...}` per selected event, and a log line listing the re-sent IDs. |
| State changes | None to `events`, `outbox`, `processed_events`, or `flags`. Only queue messages are added. |
| Repeats | Re-sending is always safe: if the original message was only late, the worker ignores the second copy (`processed_events` key). A still-unanalyzed event is re-sent on every sweep until it is analyzed. |
| Errors | Database unreachable: skip this sweep and try in 10 s. Queue unreachable: the event stays selected and the next sweep tries again. |
 
*Successful example (fixture F-J).* At the first sweep, 15 events have no analysis record: the 10 whose messages were lost and the 5 still delayed. All 15 are re-sent. Afterward, all 100 events have a `processed_events` row and the 5 late originals are ignored. This was observed in E2.
 
*Unsuccessful example.* If the sweep threshold were 0 s, an event accepted a moment ago and still on its way would be re-sent needlessly. That is harmless for correctness but wastes work, which is why the threshold must exceed normal processing time (A7).
 
**Still being worked out:** the exact threshold, which needs T7 timing data; a limit on how many times one event is re-sent before it is reported as stuck; and whether a stuck event should appear on the dashboard as a warning.

## 6 Data structure design

<!-- Assignment 4. Explain information and meaning, not only storage syntax. -->

| Record or structure | Fields and example values | Relationships and rules | Persistence |
|---|---|---|---|
| Replace | Replace | Replace | What survives restart? |

**Ownership:** Identify the authoritative place for important facts.

**Invalid combinations:** Explain how they are prevented or detected.

**Retention and deletion:** Explain how long data is needed and what deletion changes.

## 7 Algorithm design and execution

<!-- Assignment 4. Replace this block with your pseudocode. Explain any step
that hides a guarantee, such as a protected write. -->

```text
ALGORITHM NAME(inputs)
    replace with input checks
    replace with decisions and state changes
    replace with output and error behavior
```

**Normal sequence:** Numbered steps or an explained sequence diagram.

**Interrupted sequence:** Failure point, surviving state, and recovery.

**Overlapping activity:** A conflicting execution and the mechanism that handles it.

**Resource estimate:** A small justified estimate if it affects the choice.

## 8 Failures and security

<!-- Assignment 4. Include three adverse scenarios and two misuse cases. -->

| Scenario | State that may survive | Expected behavior | Planned check |
|---|---|---|---|
| Replace | Replace | Replace | Replace |

**Selected adverse case for the prototype:** Name the case and why it matters.

**Excluded conditions and consequences:** Replace.

| Possible misuse and affected asset | Response | Remaining limitation and check |
|---|---|---|
| Replace | Replace | Replace |

## 9 Implementation and feasibility

<!-- Begin the feasibility experiment and path in Assignment 3; refine in Assignment 4;
report implementation status in Assignments 5â€“6. -->

**Minimal path:** The smallest useful path is **one event, accepted once, analyzed once, and shown once**. Each step must pass before the next is added.
 
| Step | What runs | Done when | Requirement |
|---|---|---|---|
| 1 | PostgreSQL with the E1 tables (Appendix A) | Tables load and the E1 harness runs | — |
| 2 | Intake API, no publisher yet | E1 returns 202 and creates one `events` and one `outbox` row; resending returns 200 with no new rows | F1, C2 |
| 3 | Outbox publisher and RabbitMQ | E1 appears on the queue and is marked published | F2 |
| 4 | Worker (E1 logic attached to RabbitMQ) | `processed_events` has E1 and the message is acknowledged | F2 |
| 5 | Dashboard | E1 shows as `analyzed` | F4 |
| 6 | Rule R1 with fixtures F-A to F-E | T2 flag counts match | F3 |
| 7 | Reconciler (E2 sweep logic as a timer in the intake program) | With one queue message dropped, the event is analyzed within 60 s | F6 |
 
Failure tests T3 to T6 and T10 follow step 7. Step 1 and the core of step 4 already exist from E1.
 
**Feasibility experiment:** E1, crash after commit, before acknowledgement
 
**Question:** If the worker dies after saving its result but before acknowledging the message, and the message is delivered again, do counts and flags stay correct? Does the planned guarded worker do better than a worker that counts messages?
 
**Why it is high impact:** Decision 2 rests on accepting repeated delivery. If repeats corrupt R1 counts, the dashboard shows false or doubled flags, which is the exact harm in the Section 1 scenario.
 
**Procedure:**
 
1. **Real:** PostgreSQL 16.15 with the Appendix A tables, and a separate operating-system process for each delivery. The crash is a hard exit (`os._exit(137)`) placed right after the database commit and before the acknowledgement.
2. **Simulated:** the queue. The test harness plays RabbitMQ's role: if a worker process exits without printing `ACK <event_id>`, it delivers the same message again, up to 5 times.
3. **Two workers compared.** *Guarded* (Decision 2): records `event_id` in `processed_events` in the same transaction as the R1 result, and R1 counts stored events. *Naive:* adds 1 to a per-student counter per message and flags at 3 or more.
4. **Scenarios,** 20 trials each with the database reset every trial: control F-A with no crash; F-H with a crash after saving E17 (correct answer: 1 flag); F-I with a crash after saving E22 (correct answer: 0 flags); and, for the guarded worker, each F-A message given to two workers released at the same instant.
5. **Measured:** processed rows or counter value, flag rows, deliveries per run, and each delivery's exit code and output.
 
**Evidence:** Observed 2026-11-21 (Python 3.13.18, psycopg 3.3.9, PostgreSQL 16.15, Linux x86_64).
 
| Scenario | Worker | Correct answer | Observed in all 20 trials | Correct trials |
|---|---|---|---|---|
| F-A, no crash | Guarded | 3 processed, 1 flag | 3 processed, 1 flag | **20/20** |
| F-A, no crash | Naive | count 3, 1 flag | count 3, 1 flag | **20/20** |
| F-H, crash after saving E17 | Guarded | 3 processed, 1 flag | 3 processed, 1 flag | **20/20** |
| F-H, crash after saving E17 | Naive | count 3, 1 flag | **count 4, 2 flags** | **0/20** |
| F-I, crash after saving E22 | Guarded | 2 processed, 0 flags | 2 processed, 0 flags | **20/20** |
| F-I, crash after saving E22 | Naive | count 2, 0 flags | **count 3, 1 false flag** | **0/20** |
| F-A, two workers per message at once | Guarded | 3 processed, 1 flag | 3 processed, 1 flag | **20/20** |
 
Guarded worker, F-H, trial 1:
 
```text
E19 attempt 1: exit=0   out='ACK E19'
E20 attempt 1: exit=0   out='ACK E20'
E17 attempt 1: exit=137 out=''                          <- died after commit, no ACK
E17 attempt 2: exit=0   out='DUPLICATE E17\nACK E17'    <- repeat detected, nothing written
```
 
This log also shows the crash happened at the intended point. On redelivery the worker found E17 already in `processed_events`, which is only possible if the first attempt's commit completed. The full log and code are in [Appendix A](#appendix-a-experiments-e1-and-e2-code-and-evidence).
 
**Finding:**
 
1. The guarded worker was correct in all 60 crash and concurrent trials. The database's unique key, not timing, decided which delivery did the work.
2. **Negative result:** the message-counting worker was wrong in all 40 crash trials, and in F-I it flagged a student who made only two requests.
3. The naive worker passed all 20 normal trials, so **normal tests alone would not have revealed the fault.**
4. The protection came from a database transaction and unique key that the single-application baseline could also use. E1 shows design B's added risk is manageable. It does **not** show B is more reliable than A.
 
**Design consequence:** Keep the guarded worker and "count stored events, not messages" (Decision 2). Ban running counters in any consumer. Keep the crash test permanently (T5). Narrow the case for separate services to Q3 (Section 3). Start step 4 of the path from E1's worker code.
 
**Limits:** The queue was simulated, so E1 says nothing about how real RabbitMQ redelivers or behaves when it restarts. Only one crash point was tested. Intake and the publisher were not tested. Timings mostly reflect starting a new Python process per delivery, not worker speed. All data was fictional.
 
**Next experiment (E3):** Repeat F-H, F-I, and F-J with real RabbitMQ in Docker, stopping the worker container and running the sweep on its 10 s timer, to measure real recovery time.
 
**Feasibility experiment:** E2, lost messages and the reconciliation sweep
 
**Question:** If some queue messages are lost, does a queue-only design silently miss events? Does a reconciliation sweep that reads the saved records retrieve every lost event, without counting late duplicates twice?
 
**Why it is high impact:** Retrieving lost events is the project's main question. If the sweep cannot find lost events from the saved records, Decision 2 and requirements F6 and Q5 fail.
 
**Procedure:**
 
1. **Real:** PostgreSQL 16.15 with the Appendix A tables, and E1's guarded worker logic (`process_guarded`) for every delivery.
2. **Simulated:** the queue, as a list in the test harness, so chosen messages can be dropped or held back on purpose.
3. **Fixture F-J:** 100 accepted events from 20 students; 5 students should be flagged. **Ten messages are lost**, including the flag-triggering third requests of G-S1 (G003) and G-S2 (G008). **Five are delayed** until after the sweep, so each arrives twice.
4. **Two designs:** *queue only* delivers what the queue holds, then the delayed messages. *Sweep* does the same, but after the queue drains it runs `RECONCILE` (events with no `processed_events` row) and re-sends what it finds before the delayed originals arrive. In the experiment the sweep runs once with no 30 s wait, because time is not simulated.
5. **10 trials per design,** with the database reset each time and delivery order shuffled.
 
**Evidence:** Observed 2026-11-22, with the same software versions as E1.
 
| Measure | Correct system | Queue only | With sweep |
|---|---|---|---|
| Accepted events | 100 | 100 | 100 |
| Events analyzed | 100 | **90** | 100 |
| Events never analyzed | 0 | **10** | 0 |
| Flags | 5 (G-S1 to G-S5) | **3** (G-S1 and G-S2 missed) | 5 (G-S1 to G-S5) |
| Events re-sent by sweep | — | 0 | 15 (the 10 lost and 5 delayed) |
| Late duplicates ignored by worker | — | 0 | 5 |
| Correct trials | — | **0/10** | **10/10** |
 
Every trial gave the same counts regardless of delivery order. The full log and code are in [Appendix A](#appendix-a-experiments-e1-and-e2-code-and-evidence).
 
**Finding:**
 
1. **Negative result for the queue-only design:** every lost message meant an event that was accepted but never analyzed, and in 10 of 10 trials two students who met the rule were never flagged. No error was raised.
2. **The sweep retrieved every lost event** from the saved records in 10 of 10 trials, and both missed flags appeared.
3. The sweep also re-sent the 5 merely delayed events. When their originals arrived, the worker ignored them, so no event was counted twice. **Retrieval and duplicate safety must be designed together.**
4. As with E1, the protection comes from the database (the saved `events` table and the `processed_events` key), so the single-application baseline's rescan could do the same.
 
**Design consequence:** Add the reconciler to the architecture (Section 4) and make retrieval part of Decision 2. Keep `events` as the source of truth, so an event must be saved before its message is sent. Add F6, Q5, fixture F-J, and test T10.
 
**Limits:** The queue and the passage of time were simulated, so E2 does not measure how quickly a real sweep recovers (the 60 s target in Q5) or how real RabbitMQ loses messages. Only lost and delayed messages were tested; losing a saved record in the database is excluded. All data was fictional.
 
**Current implementation:** Distinguish implemented, reused, simulated, blocked, and deferred.
 
**Run instructions and CI:** Link README.md and the recorded check.

## 10 Testing and verification

<!-- Plan in Assignment 4; add observations in Assignments 5â€“6. Expected outcomes
are not results. Link raw evidence. -->

| Test and requirement | Setup and procedure | Expected result | Status and evidence |
|---|---|---|---|
| T1 normal operation | Replace | Replace | Planned until run |
| T2 correctness rule | Replace | Replace | Planned until run |
| T3 adverse scenario | Replace | Replace | Planned until run |

**Baseline and comparison question:** Replace.

**Workload and environment:** Inputs, counts, timing, overlap, versions, machine, and reset.

**Measurements:** Units and populations, including unsuccessful operations.

**Actual results:** Explain what happened and what the evidence supports.

**Limits and next test:** State remaining uncertainty.

## 11 Operation maintenance and future work

<!-- Draft in Assignment 4; add operating instructions in Assignment 5;
complete in Assignment 6. -->

**Startup and diagnosis:** How to run the system and recognize a problem.
 
**Recovery:** Action, preserved state, and verification of recovery.
 
**One future change:** Effects on components, interfaces, data, and tests.
 
**Update and rollback:** How to change versions and recover a working configuration.
 
**Responsible design:** Connect people, data, access, reuse, and resource use to choices.
 
| Risk | Early sign | Response or fallback |
|---|---|---|
| Replace | Replace | Replace |
 
| Semester II milestone | Dependency | Completion evidence | Essential or optional |
|---|---|---|---|
| Replace | Replace | Replace | Replace |
 
## 12 Revision history and references
 
### Response to feedback
 
| # | Prior feedback (source) | Design revision made | Sections changed | Status |
|---|---|---|---|---|
| 1 | Studies do not show why reliability matters to a stakeholder; connect it to an instructor task (Assignment 1) | Added the instructor review task and a table showing how a lost event hides a flag and a duplicate creates a false flag. E1 then demonstrated the false flag with a naive worker. | 1, 9 | Done. Instructor need still an assumption (A1). |
| 2 | Scope combines too many success criteria; choose one path and one property (Assignment 1) | One path (intake → queue → rule worker → dashboard) and one property (durable, duplicate-safe delivery). LLM, code evaluation, and learning claims deferred. | 1, 2, 4 | Done |
| 3 | Test a worker crash and retry with fake data, e.g., event E17 (Assignment 1) | Fixture F-H with E17, test T5, and feasibility experiment E1 actually run | 2, 9 | Done for worker logic; real broker pending (E3) |
| 4 | Explain why the workflow needs service separation (Assignment 2) | Separation now justified by Q3 only, after E1 showed duplicate safety does not depend on it | 1, 3 | Done; to be measured in T7 |
| 5 | Define an event contract and an exact rule for "repeated" (Assignment 2) | Six-field event contract, definition of "accepted", and rule R1 with threshold, window, and inclusive boundary | 2, 5 | Done |
| 6 | Replace qualitative goals with numerical targets and exact fixtures (Assignment 2) | Q1 to Q5 have numeric targets; fixtures F-A to F-J give exact counts | 2 | Done |
| 7 | Comparison table is a placeholder; compare both approaches on the same requirements (Assignment 2) | Completed approach table (simple baseline and candidate alternative), same-requirement comparison, choice and reason, and Decision 1 and Decision 2 | 3 | Done |
| 8 | Stale metadata, unfinished references and revision record, overlong progress note (Assignment 2) | Updated metadata, 100 to 150 word progress note, this response table, and references | Header, progress note, 12 | Done |
| 9 | Presentation: lead with one event reaching the dashboard and separate "recorded correctly" from "analysis is accurate" (Assignments 1 and 2) | Section 1 states the delivery question first; educational accuracy is explicitly out of scope. Presentation will follow E17's path through the Main parts and messages table, then E1's results. | 1, 4, 9 | Planned for Monday |
 
**Unresolved items and why**
 
- **Instructor need (A1)** is unvalidated because I have not interviewed an instructor or TA yet. The flag rule and dashboard may change after that conversation.
- **Real RabbitMQ behavior and real recovery time** are untested because E1 and E2 simulated the queue. E3 will test both.
- **Approach A versus B on Q3** has no evidence yet. It needs both implementations (weeks 2 to 6).
- **Retry and dead-letter limits** and the "identical content" rule for 409 are still being worked out (Section 5).
 
### Revision history
 
| Revision date | Change to my design | Reason and evidence |
|---|---|---|
| 11/6/2026 | First draft of problem, scope, and initial requirements (Assignment 1) | Initial submission |
| 11/21/2026 | Narrowed to one path and one reliability property; removed LLM analysis, code evaluation, and learning-outcome claims from the prototype | Assignment 1 and 2 feedback (items 2, 9 above) |
| 11/21/2026 | Linked delivery failures to the instructor task; added assumption A1 | Feedback item 1 |
| 11/21/2026 | Defined event contract, rule R1, numeric targets, and fixtures | Feedback items 5, 6 |
| 11/21/2026 | Removed the Katona and Gyonyoru review from the evidence | I could not find a verifiable link for that citation |
| 11/22/2026 | Narrowed the project to one distributed-systems question: retrieving lost events. Rewrote Sections 1 and 2 (instructor task, concrete scenario, labeled evidence, event contract, rule R1, numeric targets, fixtures F-A to F-J, planned tests T1 to T10). Completed Section 3 (baseline and alternative compared, choice and reason, Decisions 1 and 2), Section 4 (context and deployment diagrams, main-parts table, responsibilities, communication, saved state, build/reuse/simulate), Section 5 (component table, `POST /events` and reconciliation sweep contracts), and Section 9 (minimal path, experiments E1 and E2). Reorganized to the official template and left Sections 6 to 8, 10, and 11 for later; removed the unverified Katona and Gyonyoru citation. | Assignment 1 and 2 feedback (response table above) and the course handbook. E1: guarded worker 60/60 correct, naive worker 0/40. E2: queue only 0/10 correct (10 events never analyzed, 2 flags missed); with the sweep, 10/10 correct. |
 
### Open questions
 
| Open question | Why it matters | Next action |
|---|---|---|
| What sweep threshold and interval should be used, and when is an event declared stuck? | Too short re-sends slow events; too long delays recovery; an event re-sent forever hides a real fault | Choose from T7 timings; add a re-send limit and a dashboard warning |
| Does real RabbitMQ redeliver an unacknowledged message after the worker container is killed, as E1 assumed? | E1's conclusion depends on it | Run E3 with RabbitMQ in Docker |
| Does Approach A already meet Q3? | If it does, the extra services are not justified for this workload | Build A and run T7 on both |
| What retry limit and dead-letter handling should apply to a message that keeps failing? | Without a limit, one bad message can loop; a bad limit can drop real events | Inject a failing message; choose a limit in week 5 |
| Do instructors want R1 flags and this dashboard (A1)? | The instructor-task argument rests on it | Interview one instructor or TA |
| Is 3 requests in 600 s a useful threshold? | It is a placeholder that makes the rule testable, not an educational finding | Ask in the same interview; rerun T2 if changed |
 
**Assistance and reuse:** I used Claude (Anthropic) to help revise this document's wording, structure, and tables in response to instructor feedback, to draft the pseudocode and diagrams, and to write and run the E1 and E2 experiment scripts. I reviewed the design and the experiment output and am responsible for both. Reused software: PostgreSQL 16, psycopg 3, and Mermaid (for diagrams). No code, images, or data from other projects are reused.
 
### References
 
- National Science Foundation. [Award 2540315, "Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education."](https://www.nsf.gov/awardsearch/show-award/?AWD_ID=2540315) Supports the project context.
- Qian, Yizhou, Meishan Liu, and Liye Zhu. [Exploring the Quality and Effectiveness of AI-Generated Feedback in Introductory Programming.](https://www.nature.com/articles/s41599-026-07920-7) Humanities and Social Sciences Communications, 2026. Supports the claim that AI feedback quality varies.
- Heickal, Hasnain, and Andrew Lan. [A Classroom Study of LLM-Generated Feedback Intervention in Introductory Programming.](https://proceedings.mlr.press/v339/heickal26a.html) Proceedings of Machine Learning Research, 2026. Supports the claim that feedback type and validity affect usefulness.
- Moodle. [Using Analytics.](https://docs.moodle.org/502/en/Using_analytics) Moodle 5.2 documentation. Supports rule-based analytics as the baseline approach.
- OpenAI. [Agents: Orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration) and [Function Calling](https://developers.openai.com/api/docs/guides/function-calling). Support LLM orchestration as a building block deferred here.
- Richardson, Chris. [Pattern: Transactional outbox.](https://microservices.io/patterns/data/transactional-outbox.html) microservices.io. Supports writing the event and its outgoing message in one transaction (Decision 1).
- RabbitMQ. [Reliability Guide.](https://www.rabbitmq.com/docs/reliability) Supports acknowledgements, publisher confirms, and redelivery (Decision 2, E3).
- PostgreSQL Global Development Group. [INSERT, ON CONFLICT clause.](https://www.postgresql.org/docs/current/sql-insert.html) PostgreSQL 16 documentation. Supports the duplicate-safe insert tested in E1.
- Psycopg. [Psycopg 3 documentation.](https://www.psycopg.org/psycopg3/docs/) Database driver used in E1.
- Mermaid. [Flowchart syntax](https://mermaid.js.org/syntax/flowchart.html) and [accessibility options](https://mermaid.js.org/config/accessibility.html). Used for Figures 1 and 2 and their text descriptions.
- Brown, Simon. [The C4 model: diagrams.](https://c4model.com/diagrams) Basis for the context and container views in Section 4.
- Microsoft. [Architecture decision record.](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) Azure Well-Architected Framework. Basis for the format of Decisions 1 and 2.
- Featonby, Malcolm. [Making retries safe with idempotent APIs.](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) Amazon Builders' Library. Supports using `event_id` as a client-chosen duplicate key (Section 5, Decision 2).
 
### Supporting media and evidence
 
| Item | Location | Purpose |
|---|---|---|
| Figure 1, context view | Section 4, Boundary and outside participants (Mermaid) | Boundary and outside participants |
| Container view table | Section 4, Main parts and messages | Running parts, messages, saved state |
| Figure 2, deployment view | Section 4, Where parts run (Mermaid) | Where each part runs |
| E1 and E2 code, run instructions, full logs | [Appendix A](#appendix-a-experiments-e1-and-e2-code-and-evidence) | Reproduce and check experiment E1 |

### Final document check

- [ ] Terms and examples make the project understandable to a beginner.
- [ ] All twelve sections contain project-specific content.
- [ ] Requirements connect to design and tests.
- [ ] Architecture, components, data, and algorithms agree.
- [ ] Targets and plans are separate from observations.
- [ ] Tables, images, pseudocode, references, and evidence links render correctly.
- [ ] Images have alternative text, captions, and readable labels.
- [ ] Run instructions and evidence support reproduction.
- [ ] I am prepared to present my work and orally review the same selected group mate.
- [ ] Remaining work and maintenance needs are explicit.
- [ ] Instructional placeholders are removed from the final submission.
