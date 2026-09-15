# Agent Behavior Trace Model: shared contract

Status: Living working group document (v0.1-draft) - last reviewed 2026-09-15

This document is the draft shared contract for the Agent Behavior Trace Model workstream. It records which identities and relationships the two agent examples ([Task 2](https://github.com/aaif/wg-observability-and-traceability/issues/40)) should export, how those relationships should be interpreted, and how they map to AAIF terminology and OpenTelemetry. It follows the [execution plan](https://github.com/aaif/wg-observability-and-traceability/pull/48) and is tracked in [issue #41](https://github.com/aaif/wg-observability-and-traceability/issues/41).

Recording this draft does not make these semantics an adopted standard. Unresolved choices are collected in [section 9](#9-unresolved-choices) rather than silently decided. Once the Task 2 baselines are captured, the draft is expected to be revised against them before any version is proposed as a recommendation.

## 1. Purpose

The contract answers three questions for every exported record:

1. **What work does this record identify?** (identities, section 3)
2. **How does that work connect to other work?** (relationships, section 4)
3. **What may a consumer conclude from the records, and what must remain unknown?** (interpretation rules, section 6)

The four priority questions from the execution plan drive the scope:

- **Continuity:** which turns belong together, including after a pause or restart
- **Calls and retries:** which model calls served each turn, without counting the same work twice
- **Approvals:** which decision concerned an action, and did that action execute
- **Effects:** which external change can be connected to that execution

## 2. Reading rules

- **Contract identities** (for example `C1`, `T2`, `P1`) are illustrative fixture identities, not proposed OTel attribute names. This follows the convention in the task issues.
- **Contract terms** (Conversation, Turn, Proposed action, ...) are the versioned terms this document defines. Section 8 crosswalks each to AAIF taxonomy status.
- **Native labels** are what a given runtime calls the same thing (thread, session, chat, run). Each example maps its native labels to the contract identities in its own mapping file; mappings are declared, never assumed.

## 3. Identities

| Identity | Meaning | Fixture example |
| --- | --- | --- |
| **Conversation** | The user-visible continuity scope: the unit that survives pauses, restarts, and process boundaries. | `C1` |
| **Runtime session** | A transport- or runtime-level session (protocol session, process-local session object). A restart opens a new runtime session; the conversation continues. Distinct from Conversation and never merged with it. | `S1`, `S2` after restart |
| **Turn** | One user-initiated exchange within a conversation: request, the work it triggers, and the reply. | `T1`, `T2` |
| **Model call** | One logical inference operation, including any internal retries by the caller. | `M1` |
| **Proposed action** | A record that the agent proposed to perform an action with described arguments. Exists whether or not it is approved or executed. | `P1` |
| **Approval decision** | A human or policy decision on exactly one proposed action. Recorded when denied as well as when approved. | `A1` |
| **Tool execution** | The execution of an action through a tool. | `E1` |
| **External effect** | An independently observed change in the world, reported by a source outside the agent (for example a service-side receipt). | `ticket-42` receipt |
| **Agent trajectory** | The path of one agent instance across conversations. Named here for AAIF vocabulary alignment; not required by the four priority questions. | - |

Identity rules:

1. Identifiers are assigned by the system that owns the entity and are opaque to consumers.
2. No synthesized fallback identifiers. If a runtime owns no conversation id, the record carries none; it does not carry a new UUID, trace id, or content hash. This mirrors the OTel rule for `gen_ai.conversation.id`.
3. Records carry the native label and the contract identity together. The mapping replaces nothing; it annotates.

## 4. Relationships

| # | Relationship | Meaning | How it is expressed |
| --- | --- | --- | --- |
| R1 | Turn → Conversation | Membership: which turns belong together. | Shared conversation identity. Never trace-tree ancestry or timestamps alone. |
| R2 | Model call → Turn | Service: which calls served a turn. | Reference from the call record to the turn it served. |
| R3 | Proposed action → Turn | Origin: within which turn the proposal was made. | Reference from the proposal to its turn. |
| R4 | Approval → Proposed action | Decision: which action a decision concerned. | Reference from the decision to exactly one proposal; present for denials too. |
| R5 | Tool execution → Proposed action | Realization: which proposal an execution carries out. | Reference from the execution to the proposal (and, where applicable, to the approval it followed). |
| R6 | External effect ↔ Tool execution | Correlation: which observed change connects to which execution. | Correlation key shared between the receipt and the execution (for example the ticket id returned by the service and by the tool). |

Each relationship record carries: the source identity, the target identity, the relationship kind, and how the link was established (span link, attribute reference, causal flag, or external correlation key). Consumers must not reconstruct a relationship that was not exported, except to report it as missing.

## 5. Turn boundaries

These are candidate rules, to be tested by the examples in Tasks 5 and 6, not existing OTel requirements.

1. **One trace per turn is the candidate standalone default.** A completed turn is a closed trace; continuity comes from identity (R1), not from an open session-long span.
2. **Turn-entry span.** Each turn trace opens with a turn-entry span that anchors the turn identity. Which span plays this role is an open choice (section 9, item 1).
3. **Caller context is preserved.** When an agent runs inside an existing caller trace, the turn must not break that context; the turn trace links to it rather than discarding it. The exact mechanism is an open choice (section 9, item 4).
4. **Approval waits.** A turn may suspend while awaiting an approval decision. The suspension is recorded; the turn is not closed by it.
5. **Resumption.** When work resumes (the next morning, after a process restart, in a new runtime session and a new trace), the resumed work joins the same conversation and turn. The later turn asking "what was the ticket number?" is a new turn of the same conversation.

## 6. Interpretation rules

Rules for consumers answering the four priority questions from exported records alone:

**Continuity.** Group turns by conversation identity (R1). Do not group by trace parent, process, or wall-clock proximity. Ordering must not depend only on timestamps or a single parent-child span tree, especially for concurrent, queued, suspended, or resumed work.

**Calls and retries.** A logical model call may contain failed attempts and one success (R2). Reported usage is counted once per logical call. Retries the runtime does not export remain unknown; they are not zero and they are not silently counted.

**Approvals.** An approval decision refers to exactly one proposed action (R4). A denial produces no tool execution record; a missing decision record means the decision is unknown - it does not mean the action was unapproved, and it does not mean it was approved. Observed approval is not proof of enforcement.

**Effects.** An external effect correlates to a tool execution through a shared correlation key (R6).

- A missing observation is **unknown**, not zero and not success. A removed ticket-creation receipt makes the answer "creation unconfirmed," not "no ticket was created."
- A receipt delivered twice is **one** confirmed effect, not two. Consumers deduplicate by correlation key.
- Delayed or reordered arrival must not change the answer. Records are interpreted as a set, not as a stream.
- Conflicting observations (two receipts that disagree) are reported as conflicts, not resolved silently.
- A service-side receipt is correlation evidence, not cryptographic attestation.

## 7. OpenTelemetry crosswalk

Mapping of contract identities and relationships onto the OTel GenAI semantic conventions. All `gen_ai.*` attributes and spans below are at **Development** status as of this draft; the contract pins what it was drafted against and expects to track changes.

| Contract concept | OTel mapping | Notes |
| --- | --- | --- |
| Conversation | `gen_ai.conversation.id` on inference and agent spans | Conditionally set: only when the instrumented library natively owns an id, or the application provides one via context. No synthesized fallbacks (see identity rule 2). |
| Turn | One trace per turn (candidate default, section 5); no `gen_ai` turn attribute exists today | Turn identity is carried by the example's mapping; see the OTel [turn-entry discussion](https://github.com/open-telemetry/semantic-conventions-genai/issues/356). |
| Model call | GenAI client inference span: name `{gen_ai.operation.name} {gen_ai.request.model}`, kind `CLIENT` (`INTERNAL` acceptable in-process) | Automatic retries stay inside one span; usage via `gen_ai.usage.*` attributes. |
| Continuation between turns | `gen_ai.request.previous_response.id` where the provider supports it; `gen_ai.conversation.compacted` for compacted context views | Provider-specific; examples document availability. |
| Agent operations | Agent and framework spans: `create_agent`, `invoke_agent` (CLIENT or INTERNAL), `invoke_workflow`, `plan`; tool execution via the `execute_tool` operation span | Per the OTel GenAI agent spans conventions; an LLM call that produces a plan is a child of the `plan` span. |
| R1–R6 across traces | Span links plus shared identity attributes | Not parent-child, except where work is genuinely nested (the `plan` case above). |
| Proposed action, Approval decision, External effect | No OTel convention yet | Recorded as example-local spans/attributes through each example's mapping. Upstream proposals are [Task 9](https://github.com/aaif/wg-observability-and-traceability/issues/47), each backed by a concrete example. |

## 8. AAIF terminology crosswalk

| Contract term | AAIF taxonomy status | Native labels seen so far |
| --- | --- | --- |
| Conversation | Pending (definitions deferred by the taxonomy workstream; "Session" definition under discussion) | thread, session, chat, conversation |
| Runtime session | Pending; must stay distinct from Conversation in any taxonomy entry | session, connection |
| Turn | Pending; the plan proposes "interaction turn" as a term | turn, run, interaction |
| Proposed action, Approval decision, Tool execution, External effect | To be fed back into the taxonomy by this WG | proposal, approval, tool call, receipt |

Rules:

1. Contract documents use published AAIF terms where they exist and label pending terms as pending.
2. Each example documents its native labels and maps them to contract identities (an SDK that calls conversations "threads" says so in its mapping).
3. The crosswalk is coordinated through [issue #10](https://github.com/aaif/wg-observability-and-traceability/issues/10) and the AAIF Taxonomy & Landscape workstream. This contract is input to that crosswalk, not a definition source for the taxonomy.

## 9. Unresolved choices

1. **Turn-entry span.** Which span anchors the turn identity: the `invoke_agent` span, the first agent-owned span, or a dedicated span.
2. **Standalone default vs. caller trace.** How one-trace-per-turn composes with an agent embedded in an existing caller trace.
3. **Suspend/resume marking.** How an approval wait and a later resumption are recorded without a session-long span, so a consumer can distinguish suspended, resumed, and closed turns.
4. **Caller-context mechanism.** Span link to the caller trace versus parenting under it.
5. **Attribute names** for proposed action, approval decision, and external effect. Deferred to Task 9 upstream proposals, evidence first.
6. **Baseline revisions.** The Task 2 baselines ([#40](https://github.com/aaif/wg-observability-and-traceability/issues/40)) are not yet captured; identities and relationships here are expected to be revised against them.
7. **Conversation identity when a runtime owns none.** Whether the contract treats it as mandatory for examples, or conditional as OTel does.
8. **Usage deduplication across providers.** How usage is counted once when retries span providers or hidden retries are partially visible.
9. **Second agent/SDK.** Selection affects which native labels and resumption paths the first revision must cover.

## 10. Fixture example

The support workflow from the execution plan, expressed in fixture identities. This is the shape the Task 4 test kit ([#42](https://github.com/aaif/wg-observability-and-traceability/issues/42)) is expected to turn into fixtures with expected answers.

| Record | Identity | Related to | Notes |
| --- | --- | --- | --- |
| User asks to look up the delayed order | `T1` | `C1` | First turn; model call `M1` serves it. |
| User requests a ticket | `T2` | `C1` | Model call `M2` fails once, retries, succeeds; usage counted once. |
| Agent proposes creating a ticket | `P1` | `T2` | Proposed action with ticket fields. |
| User approves | `A1` | `P1` | Approval decision, recorded as approved. |
| Agent executes the ticket tool | `E1` | `P1`, `A1` | Tool execution. |
| Test service records ticket-42 | receipt | `E1` | External effect; correlation key is the ticket id. |
| Next morning, user asks for the ticket number | `T3` | `C1` | New turn, new trace, possibly new runtime session; joins `C1` by identity. |

Expected answers for the degraded cases:

- Receipt delivered twice: still one confirmed ticket.
- Receipt removed: "creation unconfirmed," not "no ticket was created."
- `M2` failure exported, success missing: the call and its usage are unknown, not zero.

## 11. Versioning and references

- This document is versioned with the workstream: **v0.1-draft**. Changes land through pull requests against [issue #41](https://github.com/aaif/wg-observability-and-traceability/issues/41); the review window follows the [working methods](../WORKING-METHODS.md) (specifications: at least 2 weeks).
- [Execution plan](https://github.com/aaif/wg-observability-and-traceability/pull/48) and its nine-task overview; task issues [#39](https://github.com/aaif/wg-observability-and-traceability/issues/39)–[#47](https://github.com/aaif/wg-observability-and-traceability/issues/47).
- [OTel GenAI semantic conventions: spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md) and [agent spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md), including the [turn-entry discussion](https://github.com/open-telemetry/semantic-conventions-genai/issues/356).
- [AAIF Taxonomy & Landscape workstream](https://github.com/aaif/ws-taxonomy-landscape) and [taxonomy/trace-model crosswalk, issue #10](https://github.com/aaif/wg-observability-and-traceability/issues/10).
