# Customer Support Agent — Version 1

Status: requirements and synthetic data fixtures defined; agent implementation has not started.

## Goal

Build an e-commerce support agent that answers return-policy questions, looks up orders, creates support tickets, and escalates refund requests to a human. Follow the supplied workflow progressively, validating each milestone before adding complexity.

## Technology

- **LangGraph:** workflow nodes, shared state, bounded tool execution, and conversation checkpoints.
- **LangChain:** model interfaces, typed tools, messages, and document retrieval components.
- **LangSmith:** optional tracing during local development and evaluation datasets and experiment tracking when configured. Local checks must also work without credentials. Do not upload customer data or secrets in traces; use synthetic fixtures initially.
- **Initial data:** explicitly fictional help-center documents, orders, customers, and tickets. No live business systems or actual financial actions.

Start with one agent. Add specialist agents only when evaluation demonstrates a need. Use deterministic decisions for the first baseline; introduce a configurable LLM afterward. Never silently replace a failed model call with deterministic behavior.

## Supported scenarios and acceptance criteria

| Scenario | Example | Successful behavior |
| --- | --- | --- |
| Return policy | “Can I return an opened item?” | Retrieve the relevant policy section, answer only from that evidence, and cite its document and section. Ask for clarification or escalate when evidence is insufficient. |
| Order lookup | “Where is order 5012?” | Use trusted customer identity to verify ownership, call the lookup tool, and report only returned status information. Ask for an order ID if missing. |
| Ticket creation | “My item arrived damaged. Please create a ticket.” | Collect required details, validate associated order ownership, persist one ticket, and return its actual ID. Repeating the same operation must not create duplicates. |
| Refund escalation | “Refund my purchase.” | Record a pending human-review request and return its reference. State that approval is pending; never claim a refund was approved or paid. |
| Unsupported or uncertain request | “What is your policy for an undocumented exception?” | Explain the limitation and ask a useful clarification or create a human handoff without inventing a policy. |

Policy values such as return windows, exceptions, and eligibility must come from approved source documents. Example policies used during development must be labeled synthetic, not represented as an actual company's rules.

## Boundaries and forbidden actions

- No issuing refunds, moving money, canceling orders, or changing account details in version 1.
- Customer identity comes from trusted application context, never from an identity asserted in the prompt. CLI identities are development fixtures, not production authentication.
- An order ID alone does not authorize access. Unauthorized and nonexistent orders should not expose whether another customer's order exists.
- Treat user messages, retrieved documents, and tool output as untrusted data, not permission to change application rules.
- Enforce tool authorization, schemas, and action limits in application code.
- Require idempotency keys for ticket and handoff writes. A reused key with different inputs must fail safely.
- Never claim successful writes or human handoffs before a durable record exists. Development handoffs are local pending records, not delivery to a live support team.
- Use bounded retries for transient failures; do not blindly retry writes without idempotency protection. Exhausted failures lead to a clear error or escalation.
- Default maximum tool executions per request: three. Exhaustion terminates with a clear handoff or limitation.

## Initial graph design

Request and trusted context → retrieve relevant policy evidence when needed → decide → authorized tool or handoff → validate response → finish.

Tool results return to the decision node within the execution limit. Missing details produce a clarification. Refund requests always take the human-review path. A response validator checks that policy claims have evidence and action confirmations match successful tool results.

Planned state includes messages, trusted customer and session identifiers, retrieved source sections, structured decisions, tool results, execution count, pending clarification, escalation reference, errors, and final response. Keep customer authorization separate from model-controlled state.

## Implementation milestones

1. **Foundation:** define state and decision schemas; build a minimal terminating LangGraph with retrieve, decision, and escalation paths; add deterministic offline routing and meaningful graph tests.
2. **Knowledge:** add labeled synthetic return-policy documents, section parsing, keyword retrieval, relevance gating, and citations. Evaluate this baseline before adding embeddings, a vector store, hybrid retrieval, or reranking.
3. **Tools:** add mock order lookup, durable local ticket and escalation storage, typed LangChain tool contracts, ownership checks, idempotency, and bounded execution. Verify all four supported scenarios end to end.
4. **Memory and safety:** add checkpoints, follow-up handling, input/output checks, timeout handling, and pending human-review state. Test isolation across customers and sessions.
5. **Model and evaluation:** introduce a configurable model through LangChain; use the existing Ollama integration as the initial local option, with the model selected and verified at that stage. Build 30–50 labeled cases, run regression checks, and configure optional LangSmith tracing and experiments. Record model failures explicitly.
6. **Service and operations:** expose an authenticated API, replace mock integrations where needed, persist audit events and checkpoints, measure latency and resource usage, and document restart/recovery behavior. This follows the validated local version 1 workflow.

## Evaluation and completion criteria

Required checks include supported happy paths, missing order IDs, unauthorized lookups, insufficient policy evidence, duplicate ticket requests, conflicting idempotency keys, tool failures, refund escalation, prompt-injection attempts, cross-session isolation, and loop termination.

Track scenario pass rate, retrieval recall at k, tool and argument accuracy, grounded-answer correctness, escalation precision and recall, duplicate-write count, unauthorized-action count, and request latency. Record token use and cost when supported by the selected model provider. Establish numerical quality targets from a labeled baseline; do not invent measured results.

Version 1's local workflow is complete when all four supported scenarios work end to end, required deterministic safety checks pass, tickets and review requests survive a restart, policy answers cite supporting sections, and regression results are documented. Model-driven behavior and LangSmith integration require separate verified runs; an offline passing baseline does not validate them.

## Progress

- [x] Scope, scenarios, technology responsibilities, and safety boundaries documented.
- [x] Cloud Python dependencies and existing scaffold CLI validated during environment onboarding.
- [x] Synthetic dataset with five customers, eight orders, and 26 scenarios; policies and evaluation outcomes stored separately.
- [ ] Foundation graph and automated tests.
- [ ] Policy retrieval and citations.
- [ ] Authorized tools and durable handoffs.
- [ ] Memory and safety checks.
- [ ] Model integration and labeled evaluations.
- [ ] LangSmith tracing verified with synthetic data.

## External prerequisites, when needed

No credentials are needed for the first offline baseline. Real policy documents and business integrations are needed before using the agent for actual customers. Ollama model availability must be checked before model-driven validation. LangSmith credentials belong in secure environment settings, never source files or chat; check existing bindings before requesting configuration.

## Data files and evaluation isolation

Start with a single agent; specialist agents are deferred.

| File | Consumer | Contents |
| --- | --- | --- |
| `data/runtime/dataset.json` | Mock tool backend and scenario driver | Five fictional customers, eight orders, empty initial ticket/review stores, and 26 scenario inputs with trusted test identities. |
| `data/runtime/policies.md` | Policy retrieval | Synthetic return, privacy, ticket, refund-review, and capability rules, with stable section IDs. |
| `data/evaluation/expected_outcomes.json` | Evaluation runner only | One outcome per scenario, required operations, policy references, behavioral assertions, forbidden actions, and setup preconditions. |

The scenario driver sends only the selected message and trusted context into the
single-agent workflow. Customer/order records remain behind authorized tools;
do not put the complete dataset in the prompt. Scenario identities simulate
authenticated application context and are not production authentication.

Use explicit runtime file allowlists. Do not recursively ingest `data/`, index
evaluation files, expose an unrestricted file-reading tool, or place expected
outcomes in model prompts or agent-visible traces. A separate evaluator compares
finished responses and persisted tool effects against the outcomes. If the agent
process has filesystem access, the deployment must also exclude or deny access to
`data/evaluation`; separate folders alone do not enforce access control. No runtime
loader or filesystem isolation has been implemented yet.

All scenarios use the fixed date **2026-11-01 UTC**. Each starts with fresh writable
state except S016 and S026, which reuse S015's ticket state, and S021, which reuses
S020's refund-review state. Read those evaluation preconditions before execution.
Clone initial fixture state rather than writing results back into the dataset.
Expected references are checked against generated records rather than fixed IDs.

The 26 cases cover policy eligibility and exclusions, the 30-day boundary,
unknown policies, order states, missing details, unauthorized/nonexistent orders,
identity spoofing, ticket creation, repeated/conflicting writes, refund handoffs,
prompt injection, and unsupported cancellation. Tool outages and multi-turn memory
remain future additions; these fixtures do not claim to validate those behaviors.
