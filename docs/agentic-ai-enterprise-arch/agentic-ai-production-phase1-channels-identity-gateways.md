---
layout: default
title: "Agentic AI in Production — Phase 1: The Front Door (Channels, Identity, API & AI Gateway)"
description: "Phase 1 of a series on taking agentic AI from proof-of-concept to enterprise production: Layers 1–2 — channels, identity, API gateway, AI gateway, guardrails and approvals — with cloud-provider options, build-vs-buy decisions, and the cross-cutting concerns that apply at each layer."
---

# Agentic AI in Production — Phase 1: The Front Door

<p class="article-meta">Part 1 of a new series on turning agentic AI proofs-of-concept into production-grade enterprise systems. This series walks an enterprise reference architecture layer by layer. Phase 1 covers <strong>Layer 1 — Experience &amp; Channels</strong> and <strong>Layer 2 — Identity, Gateway &amp; Governance</strong>. The question: <em>who is allowed to reach the agent, through which door, on whose authority, and what does that door enforce before any model or tool is touched?</em></p>

## The POC has one caller. Production has many kinds.

A typical agent POC has a single entry point: a notebook, a Streamlit page, a `curl` command. The caller is the developer, the API key sits in an environment variable, and the model is whichever one looked best in the demo.

Production breaks each of those assumptions:

- The caller may be an employee in a copilot, a partner's system, a schedule, a Kafka event, or a customer on a phone.
- The identity may be a user, a service, an *agent acting on behalf of a user*, or a chain of all three.
- The model will change, the provider will throttle, and the budget is owned by someone who was not in the demo.
- One request can fan out into many model calls and tool calls, so a single unauthenticated hit can cost real money or trigger a real action.

None of this is "model quality". It is the **front door**, and in agentic systems it is where a large share of security, cost and accountability incidents begin. A chatbot that answers wrongly is a quality problem. An agent that acts wrongly, under the wrong identity, with no record of who authorized it, is an incident.

### The forces that shape every layer

The RAG series used four forces: scalability, resilience, cost and latency, then added quality and security. Agentic systems add a seventh: **accountability**. Every action needs an answer to *"who caused this, under whose authority, with what evidence, and can we prove it later?"*

### How this series is organized

The reference architecture has seven layers plus a set of **cross-cutting concerns** that run through all of them. Each phase of this series takes one or two layers and, for each, asks what the cross-cutting concerns look like *at that layer* rather than treating them as a separate chapter at the end.

| Layer | Focus | Phase (planned) |
|---|---|---|
| 1. Experience & Channels | How users and systems reach the solution | **Phase 1 (this article)** |
| 2. Identity, Gateway & Governance | Access, policy enforcement, model gateway, guardrails, approvals | **Phase 1 (this article)** |
| 3. Agent Orchestration | Supervisor/router, domain agents, workflows, state and memory | Later phase |
| 4. Intelligence & Tools | LLM gateway, tool gateway/MCP, memory, RAG | Later phase |
| 5. Knowledge & Enterprise Systems | Data, apps, event bus, external systems | Later phase |
| 6. Platform & Delivery | Runtime, DevSecOps, registries, networking, reliability | Later phase |

### The cross-cutting concerns we track at every layer

| Concern | The question it asks at each layer |
|---|---|
| **Security** | What can an attacker or a mistake reach here, and how far? |
| **Observability** | Can we see, trace and explain what happened here? |
| **Evaluation** | How do we measure that this layer behaves correctly, including under attack? |
| **FinOps** | What does this layer cost, and who is accountable for it? |
| **Audit** | Can we reconstruct who did what, when, and under what authority? |
| **Governance** | Who owns the policy for this layer, and how does it change safely? |
| **Data privacy** | What personal or confidential data passes through, and for how long? |
| **Business outcomes** | Is this layer helping the business measure value, not just activity? |

---

## Layer 1: Experience & Channels

The architecture lists four channel types: **Chat & Voice**, **Apps & APIs**, **Event Triggers** and **External Users**. They look like UI variants. They are really different operating environments with different failure modes.

### Channel comparison

| Channel | What is different in production | Typical failure | Design question |
|---|---|---|---|
| **Chat / Copilot** | Long-lived streaming, conversation state, human patience measured in seconds | Proxy buffers or times out a stream; user retries and duplicates an action | Who owns session state: client, gateway or orchestrator? |
| **Voice** | Hard latency budget, barge-in, partial transcripts, speech-recognition errors feeding the agent | 1.5 s of silence feels broken; a misheard number triggers a wrong action | Where does the latency budget go: recognition, gateway, model, speech synthesis? |
| **Apps & APIs** | Versioned contracts, SDK consumers, bursty load | Response schema changes break a client; one client saturates shared quota | Is the response a stable contract or "whatever the model said"? |
| **Event triggers** | No human waiting, at-least-once delivery, replays | Duplicate event causes a duplicate refund; poison message loops through retries | Is every agent action idempotent, and what does the dead-letter path look like? |
| **External users** | Untrusted, adversarial, multi-tenant, possibly anonymous | Prompt abuse, quota exhaustion, cross-tenant leakage | What is the identity and abuse-control model before the first token? |

### Decisions that look small and are not

**1. Streaming protocol.** Server-sent events are simpler and pass through most proxies and CDNs. WebSockets support bidirectional flows (voice, interruption) but are harder to scale, authenticate per message and load balance. Whichever you pick, check the *entire* path (CDN, WAF, API gateway, ingress, service mesh) for response buffering and idle or absolute timeouts. Streaming that works on a laptop and dies at a 30-second gateway limit is one of the most common POC-to-production surprises.

**2. Async by default for long-running work.** An agent that plans, calls tools and waits for approval is not a single HTTP request. Model it as *submit → job id → status / stream / webhook*. Synchronous chat is the special case.

**3. Event triggers need agent-safe semantics.** Queue-driven agents inherit at-least-once delivery. Require idempotency keys on every side-effecting tool call, cap the maximum number of agent steps, and route repeated failures to a dead-letter queue with the full trace attached. An event-driven agent with no step cap is an unbounded cost generator.

**4. Identity must travel with the channel.** A chat UI that authenticates the user and then calls the backend with a shared service credential has silently discarded the user's identity. Everything downstream (data access control, tool authorization, audit) then runs against the wrong principal. Layer 1 must hand a verifiable identity to Layer 2.

**5. Not every channel deserves every capability.** A voice channel with speech recognition errors and weak caller authentication should not be able to approve a payment, even if the same agent can do so from an authenticated web app. Define per-channel action ceilings.

**6. External users change the threat model.** Plan per-tenant quotas, abuse detection and bot protection from day one. External traffic should not share a rate-limit pool with internal users.

### Cross-cutting lens: Layer 1

| Concern | What it looks like at the channel layer |
|---|---|
| **Security** | WAF and bot protection at the edge. Verify webhook signatures and channel-platform tokens (Teams, Slack, WhatsApp). Authenticate event sources, not just event content. Voice channels face spoofing and synthetic-voice risk, so avoid voice-only authentication for sensitive actions. Treat *content that arrives through a channel* (inbound email, attachments, pasted text) as untrusted input, because it can carry indirect prompt injection into an agent that was triggered by it. Enforce request size limits and CSRF/CORS rules for browser channels. |
| **Observability** | Generate the **correlation/trace id at the channel edge** so it follows the request through every later layer. Track per-channel SLIs: time to first token, stream drop rate, speech-recognition error rate, event lag, dead-letter depth. Client-side telemetry matters because server-side "200 OK" says nothing about a stream that stalled on the user's device. |
| **Evaluation** | Build evaluation sets *per channel*. Voice needs transcripts with realistic recognition errors and accents. Mobile chat skews to short, ambiguous queries. Event triggers need replay and duplicate-delivery tests. Red-team each channel separately, since the attack surface differs. |
| **FinOps** | Channels have very different cost profiles: voice adds recognition and synthesis charges plus long-held connections; events can produce cost storms after an upstream incident; anonymous external traffic can be abused for free model access. Tag every request with channel and tenant so cost can be attributed, and set per-channel budgets and circuit breakers. |
| **Audit** | Record channel, client/device context, session id and consent state alongside the request. If an approval or confirmation happens in a channel, record how it was authenticated, since "the user said yes in a chat window" is weaker evidence than "the user confirmed with step-up authentication". |
| **Governance** | Publish a channel onboarding standard: what a new channel must provide (identity, trace id, rate limits, response contract, data-handling statement) before it can reach an agent. Version the response contract for API consumers. Define which agent capabilities each channel is permitted to reach. |
| **Data privacy** | Voice recordings and transcripts can be biometric or highly sensitive. Chat history is personal data with retention limits and deletion rights. Edge/CDN placement affects data residency. Show consent and disclosure ("you are talking to an AI") where required. Consider client-side or edge redaction before content leaves the trust boundary. |
| **Business outcomes** | Measure outcomes per channel: task completion, containment/deflection where relevant, time to resolution, abandonment, escalation to a human, satisfaction. Activity metrics like "messages sent" do not show value. |

### Cloud options for Layer 1

| Concern | Azure | AWS | GCP |
|---|---|---|---|
| Web edge and WAF | Front Door, Application Gateway + WAF | CloudFront, AWS WAF | Cloud CDN / Load Balancing, Cloud Armor |
| Chat/bot channel plumbing | Azure Bot Service, Teams integration | API Gateway (WebSocket), Amazon Lex, Amazon Connect | Dialogflow CX, Contact Center AI |
| Voice / real-time media | Azure Speech, Azure Communication Services | Transcribe / Polly / Connect | Speech-to-Text / Text-to-Speech, Live API |
| Event triggers | Event Grid, Service Bus, Functions | EventBridge, SQS/SNS, Lambda | Pub/Sub, Eventarc, Cloud Scheduler |
| External-user identity | Entra External ID | Cognito | Identity Platform |

**Where custom code is usually needed:** the **channel adapter**. Managed bot frameworks are strong for scripted or intent-driven flows but constrain agentic, streaming, tool-using interactions. Most teams end up with a thin custom adapter that normalizes web chat, Teams/Slack, voice and events into one internal request envelope: identity, tenant, channel, correlation id, idempotency key, consent flags. It is a small amount of code that pays for itself as soon as the second channel is added.

---

## Layer 2: Identity, Gateway & Governance

The architecture places five components here: **Identity & Access**, **API Gateway**, **AI Gateway**, **Guardrails** and **Approvals**. Together they answer: *who is this, how much may they use, which model handles it, what is allowed, and does a human need to say yes?*

### 2a. Identity & Access — the hardest problem in agentic AI

Classic applications have two identity questions: who is the user, and what may they do. Agents add a third: **what may this agent do, and on whose behalf?**

| Model | How it works | Pros | Cons |
|---|---|---|---|
| **Service identity only** | The agent calls tools with its own credential | Simple; easy to operate | Loses the user; over-privileged by design; audit says "the agent did it"; confused-deputy risk |
| **User delegation (on-behalf-of)** | The user's token is exchanged for a scoped downstream token (OAuth token exchange / OBO) | Preserves user permissions; audit ties to a person; least privilege | Token lifetime vs long-running work; every downstream system must support it; consent complexity |
| **Hybrid: agent identity + delegated user context** | Agent has its own workload identity *and* carries the user's context | Clear audit trail; separates "what the agent may ever do" from "what this user may do through it" | Most moving parts; standards and tooling are still maturing |

**A useful rule:** the effective permission of an agent action should be the **intersection** of what the agent is allowed to do and what the user is allowed to do, never the union.

Further decisions:

- **RBAC vs ABAC.** Roles are easy to reason about but multiply with per-tenant, per-document and per-context rules. Agent scenarios ("finance user, managed device, own accounts only, under a threshold") push toward attribute- or policy-based decisions. A policy engine (OPA, Cedar-based services, cloud-native equivalents) keeps rules out of prompts and out of scattered `if` statements.
- **Token passthrough is an anti-pattern.** Do not forward the inbound user token to arbitrary tools or MCP servers. Exchange it for a narrowly scoped, audience-bound token per downstream service.
- **Long-running agents outlive tokens.** Design for refresh, re-consent or re-authorization at approval points rather than storing a long-lived credential.
- **Secrets custody.** Provider API keys and tool credentials belong in a vault behind the gateway, never in client code, prompts or agent memory.

**Cloud options (identity):**

| | Azure | AWS | GCP |
|---|---|---|---|
| Workforce identity | Entra ID (OAuth2/OIDC, Conditional Access; agent-oriented identity is emerging) | IAM Identity Center, IAM roles | Cloud Identity, Workforce Identity Federation |
| Workload identity | Managed Identity, Workload Identity Federation | IAM roles, IRSA / Pod Identity | Service accounts, Workload Identity Federation |
| Customer identity | Entra External ID | Cognito | Identity Platform |
| Fine-grained authorization | Entra + custom policy engine | Amazon Verified Permissions (Cedar) | IAM Conditions + custom policy engine |
| Agent-specific identity / credential brokering | Emerging (Entra agent identity, Foundry) | Bedrock AgentCore Identity | Emerging (Agent Engine, Gemini Enterprise) |

**Honest assessment:** agent-to-tool delegated identity is the least standardized area in the whole architecture. Cloud-native services cover *authentication* well. Expect to build glue: a token-exchange service or gateway plugin, a per-tool scope map, and audit enrichment that records `user`, `agent`, `tool`, `decision`. **Delegation semantics usually need custom design.**

**Cross-cutting lens: Identity & Access**

| Concern | What it looks like here |
|---|---|
| **Security** | Least privilege by intersection; short-lived scoped tokens; no shared super-credentials; separate identities for dev, test and prod agents; protection against confused-deputy and privilege escalation via tool chaining. |
| **Observability** | Log identity decisions (who, which agent, which scope, allow/deny, policy version). Watch for anomalies such as an agent suddenly using scopes it never used before. |
| **Evaluation** | Test authorization as a first-class suite: cross-tenant attempts, expired tokens, stale permissions after role change, escalation via a chain of tool calls. |
| **FinOps** | Identity is the key to cost attribution: without a reliable user/team/tenant identity on every request, no chargeback model works. |
| **Audit** | Every agent action traceable to a human principal (or explicitly to an autonomous agent policy), with the authorization decision and policy version recorded. This is the foundation for any regulatory review. |
| **Governance** | Agent identity lifecycle: who registers an agent, who owns it, how permissions are reviewed, how it is decommissioned. Treat agents like privileged non-human identities with owners and access reviews. |
| **Data privacy** | Consent and purpose limitation travel with the delegated token; minimize the claims passed to downstream tools; tenant context is enforced, not assumed. |
| **Business outcomes** | Correct identity enables per-team and per-product value tracking, and safe self-service onboarding of new use cases. |

### 2b. API Gateway vs AI Gateway — related, not interchangeable

Teams often ask whether their existing API gateway can also be the AI gateway. Sometimes yes, often partly. They protect different things.

| Concern | API Gateway | AI / LLM Gateway |
|---|---|---|
| Unit of control | Requests, routes, clients | **Tokens**, models, prompts, providers |
| Rate limiting | Requests per second/minute | Tokens per minute per team/app/user; budget caps |
| Cost control | Indirect | Direct: per-token attribution, quotas, alerts |
| Routing | Path/host/version based | Model-aware: capability, cost, region, health, A/B |
| Resilience | Retries and circuit breakers on backends | Provider failover, model fallback, 429-aware backoff, load balancing across deployments |
| Caching | HTTP response caching | Exact and semantic caching (with correctness and authorization risks) |
| Payload awareness | Mostly opaque | Understands prompts and completions; can log, redact and apply policy |
| Streaming | Often an afterthought | Must be first class (token counting on streams) |
| Key management | Client keys | Central custody of provider keys; clients never see them |
| Observability | Latency, status | Tokens in/out, cost, model version, cache hit, guardrail verdicts |

**The AI gateway in one line:** the single choke point where model access is authenticated, metered, routed, protected and observed, so applications never talk to model providers directly.

**Why it matters:** without it, every team hardcodes a provider, holds its own key, implements its own retries (multiplying load during an outage), and reports cost differently. Swapping a model then becomes a migration project instead of a configuration change.

### Options for the gateway layer

| Approach | Examples | Strengths | Limits |
|---|---|---|---|
| **Cloud API gateway with AI policies** | Azure API Management (token limits, token metrics, semantic cache, backend pools); Google Apigee (LLM/token-aware policies, Model Armor integration); AWS API Gateway (generic; AI controls mostly via Bedrock features or custom code) | Fits existing enterprise API governance, auth, developer portal and networking | AI-specific depth varies by cloud; streaming and long-duration limits need verification; policy language can get awkward |
| **Model-platform built-ins** | Amazon Bedrock (guardrails, inference profiles, cross-region inference, quotas); Azure AI Foundry (content safety, deployments, quotas); Vertex AI (Model Armor, quotas) | Least operational effort; deep integration with that cloud's models | Lock-in; weak for multi-provider routing; governance split across consoles |
| **Managed AI gateway (SaaS)** | Cloudflare AI Gateway, Portkey, managed Kong | Fast adoption; multi-provider; analytics out of the box | Another vendor in the data path; data residency and prompt-content concerns |
| **Self-hosted / open source** | LiteLLM proxy, Kong, Envoy AI Gateway, Apache APISIX | Full control; portable; runs inside your network | You own scaling, HA, upgrades, patching and on-call |
| **Fully custom** | Thin service on Envoy/NGINX or an application framework | Exactly your semantics | Highest long-term cost; easy to underestimate |

*Product capabilities in this area change every quarter. Verify current features, streaming behavior, timeouts and limits against vendor documentation before committing.*

### When to go custom (and when not to)

Choose **managed / cloud-native** when one cloud and a small number of model providers cover your needs, governance already runs through that cloud's API management, and the team has no capacity to operate another critical-path service.

Choose **open source / self-hosted** when multi-cloud or multi-provider routing is a real requirement (not a hedge), data must stay inside a controlled network boundary, or you need routing or budget logic managed products cannot express.

Go **custom** only for the parts that are truly yours:

1. **Business-aware quota and chargeback**: budgets by cost centre, product or tenant tier, tied to your billing model.
2. **Delegated identity and token exchange** for agent-to-tool calls.
3. **Routing rules tied to business context**: for example, "regulated-data requests may only use models in region X".
4. **The channel-to-gateway envelope**: correlation ids, idempotency keys, tenant context.
5. **Approval workflows** integrated with internal systems.

A healthy common pattern is **a managed or open-source gateway as the data plane, plus a small custom control plane** (policy, budgets, identity mapping) that configures it. Avoid writing a full LLM proxy from scratch unless you have a specific reason: streaming, token accounting, provider quirks and failover are each harder than they look.

### The gateway is now critical infrastructure

Everything routes through it, so the trade-offs are sharp:

- **Latency:** every hop adds time. Budget for it explicitly, and keep heavy checks (deep guardrail scans, semantic-cache lookups) off the path when a risk-based route can skip them.
- **Single point of failure:** run multi-instance and multi-zone at minimum. Decide each failure mode up front: for key custody and authorization the answer is usually *fail closed*; for optional analytics it is *degrade and alert*.
- **Blast radius of a bad configuration:** one wrong routing rule can send regulated data to the wrong model. Treat gateway configuration as code: reviewed, versioned, canary-deployed.
- **Streaming token accounting:** cost and quota logic must handle partial streams, client disconnects and retries without double-counting.
- **Semantic cache versus authorization:** a cache lookup must sit *behind* identity and tenant scope, or be keyed by it. Otherwise a cache hit can serve one tenant's answer to another and the authorization check never runs.
- **Retry ownership:** if the client SDK, the gateway and the orchestrator each retry, one provider blip becomes a request storm. Decide who retries, and only for transient errors.

**Cross-cutting lens: API Gateway & AI Gateway**

| Concern | What it looks like here |
|---|---|
| **Security** | Central custody of provider keys; mTLS and private connectivity to providers; request validation and size limits; blocking direct egress to model providers from application networks so the gateway cannot be bypassed; signed and reviewed configuration changes. |
| **Observability** | The gateway is the best place to capture uniform telemetry: tokens in/out, latency (including time to first token), model and version, provider errors and throttling, retries, fallbacks, cache hits, guardrail verdicts, cost per request. Emit standard traces so downstream layers can join on the same correlation id. Capture prompt and completion *content* selectively under a retention policy, not by default. |
| **Evaluation** | Route a slice of traffic (shadow or canary) to a candidate model through the gateway and compare quality, latency and cost before switching. Gateway-level A/B is the natural rollout mechanism for model changes. Also test failover paths: does the fallback model behave acceptably, not merely respond? |
| **FinOps** | The main control point for spend: token quotas per team/app/user, hard and soft budgets, alerts on anomalies, model-tier routing (cheaper models for easy tasks), cache savings, and chargeback/showback reports. Watch for hidden cost: a semantic cache with a poor hit rate adds embedding cost and latency for little benefit. |
| **Audit** | Immutable log of who called which model, when, with what policy decisions. Record configuration changes (who changed a route or quota and when), since a routing change can itself be a compliance event. |
| **Governance** | An approved model catalogue: which models are allowed for which data classifications and regions. Model onboarding and deprecation process. Change management for routing and quota policies. |
| **Data privacy** | Enforce data-residency routing (regulated data only to in-region deployments). Decide what leaves the boundary to third-party providers, what is redacted in flight, and whether providers may retain or train on prompts (contractually and technically verified). Manage retention of any logged content. |
| **Business outcomes** | Cost per successful task, cost per business transaction and cost by product line, not just cost per million tokens. This lets the business decide whether a use case is worth its running cost. |

### 2c. Guardrails at the gateway — what belongs here and what does not

The gateway is a natural place for **coarse, cheap, uniform** controls: input size and rate limits, PII detection on prompts leaving the trust boundary, known-pattern injection screening, output secret/PII scanning, and content policy. Options include provider-native safety services (Azure Content Safety and Prompt Shields, Bedrock Guardrails, Google Model Armor) and open-source tooling (NeMo Guardrails, Llama Guard, LLM Guard, Presidio).

It is the *wrong* place for controls that need application context: document-level authorization, groundedness against retrieved evidence, and business-rule validation. Those belong closer to retrieval, tools and output handling. Central guardrails give consistency, not completeness. A gateway that "handles safety" can give downstream teams false confidence that they can skip their own controls.

| Option | Pros | Cons |
|---|---|---|
| Provider-native guardrails | Easy; low latency; maintained by the vendor | Tied to that platform; limited customization; behaviour changes with vendor updates |
| Open-source / self-hosted classifiers | Portable; tunable; data stays inside | You own evaluation, tuning and scaling; added latency and compute cost |
| Custom rules / classifiers | Fits domain-specific risks precisely | Coverage gaps; maintenance burden; keyword-only rules are easily bypassed |

**Cross-cutting lens: Guardrails**

| Concern | What it looks like here |
|---|---|
| **Security** | Direct and indirect injection screening, jailbreak patterns, secret and PII leakage prevention, unsafe output rendering. Assume paraphrase, encoding and multi-turn attacks will get past keyword rules. |
| **Observability** | Log every guardrail decision (rule/model, verdict, confidence, action taken), not only blocks. Track block rate, override rate and latency added per guard. |
| **Evaluation** | Guardrails need their own test suites: false positives (legitimate requests blocked) and false negatives (attacks passed), across languages and obfuscation. Re-run security regression whenever the guardrail, model or prompt changes. |
| **FinOps** | Each guard adds latency and often a model call. Use risk-based routing so only higher-risk requests get the expensive checks. |
| **Audit** | Record which policy version made each decision, so a blocked or allowed request can be explained later. |
| **Governance** | A named policy owner; a change process for thresholds and rules; documented fail-closed vs degrade decisions per guard. |
| **Data privacy** | PII detection and redaction *policy* should be purpose-aware: blindly masking every identifier can break legitimate use, while allowing everything defeats the purpose. The question is "is this disclosure allowed for this user, purpose and policy?" |
| **Business outcomes** | Balance risk and usability: track false-positive rates and their impact on task completion, not only the number of threats blocked. |

### 2d. Approvals — human-in-the-loop as an architecture component

Approvals are more than a UI button. They interrupt an agent mid-plan and resume it later, potentially hours or days later.

Design decisions:

- **What triggers approval?** Risk-based rules: action type, monetary threshold, data sensitivity, first-time actions, low confidence. Requiring approval for everything trains humans to click "yes" without reading.
- **What does the approver see?** The proposed action, its parameters, the evidence and reasoning, the requester and the consequences, not a raw model transcript.
- **Who may approve?** Separation of duties: the requester must not approve their own high-risk action.
- **What happens on timeout?** The default should be *do nothing and notify*, not auto-approve.
- **State durability.** The agent's state must survive restarts while it waits, which pushes toward durable workflow engines instead of in-memory loops.
- **Approver channel.** Approving from chat is convenient but weaker evidence than a step-up authenticated approval.

**Options:** cloud workflow services (AWS Step Functions, Azure Durable Functions / Logic Apps, Google Workflows), durable-execution engines (Temporal), agent-framework interrupts (for example LangGraph), and ITSM or chat-based approvals (ServiceNow, Teams, Slack). The approval *mechanism* is easy to buy. The approval *policy*, meaning what needs a human and who decides, is always custom and is a governance conversation before it is a technical one.

**Cross-cutting lens: Approvals**

| Concern | What it looks like here |
|---|---|
| **Security** | Approver identity is strongly authenticated; approval requests cannot be forged or replayed; the approved action is bound to the *exact* parameters shown (no swap-after-approval); protection against approval-fatigue attacks that flood reviewers. |
| **Observability** | Metrics for approval queue depth, time to decision, approve/reject/timeout rates, and how many agent runs are stalled waiting. |
| **Evaluation** | Test that risk thresholds route the right actions to humans; sample approved actions for outcome quality; measure how often approvers change or reject the agent's proposal, which is a direct signal of agent reliability. |
| **FinOps** | Human review time is a real cost. Tune thresholds so humans review what matters. Waiting agents should not hold expensive resources (idle compute, open model sessions). |
| **Audit** | The strongest audit artefact in the system: who approved, when, what they saw, which policy required it, and what then executed. Must be tamper-evident. |
| **Governance** | Delegation and out-of-office rules, escalation paths, periodic review of thresholds, and explicit criteria for when an action class can graduate from "human approves" to "human is notified". |
| **Data privacy** | Approvers should see only the data necessary to decide; the approval record itself may contain sensitive content and needs retention and access rules. |
| **Business outcomes** | Track the trade-off between risk avoided and speed lost: approval latency, straight-through rate, and incidents prevented. |

---

## Cross-cutting summary matrix (Layers 1–2)

| Concern | Layer 1: Channels | Identity & Access | API / AI Gateway | Guardrails | Approvals |
|---|---|---|---|---|---|
| **Security** | Edge protection, signed webhooks, channel-specific abuse | Least privilege, scoped tokens | Key custody, no bypass path | Injection, leakage, unsafe output | Strong approver auth, bind to exact action |
| **Observability** | Trace id origin, per-channel SLIs | Identity decision logs | Tokens, cost, model, fallbacks | Guard verdict logs | Queue and decision metrics |
| **Evaluation** | Per-channel eval sets, replay tests | Authz test suite | Shadow/canary model rollout | False positive/negative tests | Threshold and outcome sampling |
| **FinOps** | Channel cost profiles | Attribution key | Quotas, routing, chargeback | Risk-routed checks | Human-review cost |
| **Audit** | Channel and consent context | Human-to-agent action trace | Immutable call and config log | Policy version per decision | Tamper-evident approval record |
| **Governance** | Channel onboarding standard | Agent identity lifecycle | Model catalogue, change control | Policy ownership | Threshold and delegation policy |
| **Data privacy** | Recording, retention, residency | Purpose and consent in tokens | Residency routing, provider terms | Purpose-aware redaction | Minimal data shown to approvers |
| **Business outcomes** | Task completion per channel | Safe self-service onboarding | Cost per successful task | Risk vs usability | Risk avoided vs speed lost |

---

## What breaks when you scale the front door

| POC assumption | Production reality | Consequence |
|---|---|---|
| One shared API key | Many apps, tenants and environments | No attribution, no per-team limits; one leak compromises everything |
| Direct call to one model | Provider outages, 429s, model deprecations | Outage or emergency code change |
| Requests are cheap | One agent run means many model calls | Surprise bills; runaway loops |
| The caller is trusted | External users, events, partners | Abuse, injection, data exfiltration |
| Service account does everything | Users have different permissions | Privilege escalation through the agent |
| A human approves in the demo | Approvals at volume | Approval fatigue, or a bottleneck that stalls agents |
| Logs are for debugging | Logs are also a data store | Prompt and PII sprawl |

## Technical debt you may be accepting — and whether it is worth it

| Shortcut | Acceptable when | It hurts when |
|---|---|---|
| Direct-to-provider calls, no gateway | Single internal pilot, small user base, hard spending cap | A second team, second model or first external user arrives |
| Shared service identity for tools | Read-only, low-sensitivity tools | Any write action, or any per-user data |
| Provider-native guardrails only | Single provider, low-risk domain | Multi-model routing, regulated domain, audit-grade explanation needed |
| Keyword blocking as the security control | Never as the sole control | After the first paraphrased attack |
| Skipping approvals "for now" | Reversible, low-value actions | Anything financial, legal, external-facing or destructive |
| Building a custom LLM proxy | You have a unique requirement and an owner | It becomes a full-time product |
| Semantic cache before authorization | Never | The first cross-tenant hit |

---

## The decision framework: seven questions before you greenlight Layers 1–2

1. **Whose authority is every downstream action executed under?** If the answer is "the agent's service account" for anything user-specific, you have a confused-deputy problem.
2. **Can any client reach a model or tool without passing the gateway?** Direct provider access defeats every control described here. Block it at the network and credential level.
3. **Can you attribute every token to a team, user and feature, and cap it?** If not, a runaway loop is a budget incident waiting to happen.
4. **What happens when the primary model or provider is unavailable or throttled?** Is the fallback compatible in capability, region and data policy, or does it silently violate a compliance rule?
5. **Which controls run at the gateway, and which must run closer to the data?** Make sure central guardrails are not standing in for authorization in retrieval and tools.
6. **What may the agent do without a human, and who decides?** Approval thresholds need a named owner and a review cadence.
7. **Is the gateway operated like critical infrastructure?** HA, configuration review, canary deployment, capacity plan, on-call, and a defined failure mode for each dependency.

## Choosing an approach

| If your situation is... | Lean towards |
|---|---|
| Single cloud, enterprise API governance already in place | Cloud API management with AI policies + native model guardrails |
| Multi-provider or multi-cloud models, strong platform team | Open-source gateway (LiteLLM / Envoy / Kong) + custom control plane |
| Small team, speed matters, low data sensitivity | Managed AI gateway SaaS |
| Regulated data, strict residency | Self-hosted gateway inside your boundary; provider-native or self-hosted guardrails |
| Agents that act on user data across many systems | Invest early in delegated identity, a policy engine and durable approvals, whichever gateway you choose |

The consistent lesson: **buy the data plane where you can, build the control plane where your business rules live, and never let the front door be the only place security is enforced.**

## The end-to-end picture

The diagram below follows one request through Layers 1–2: from any channel, through edge protection and the channel adapter into a single request envelope, then through authentication, authorization, input guardrails, the AI gateway and model providers, back through output guardrails, and finally to either automatic release or a human approval before it is allowed to reach agent orchestration. The bottom band shows how the eight cross-cutting concerns all hang off the same correlation context created at the edge.

<iframe src="agentic-architecture-phase1-diagram.html" width="100%" height="1900" style="border:1px solid #ddd; border-radius:8px;" title="Agentic AI in Production - Phase 1 Channels, Identity and Gateways Architecture">
  <p><a href="agentic-architecture-phase1-diagram.html">View the Phase 1 architecture diagram</a> (your browser does not support iframes).</p>
</iframe>

The next phase moves one layer deeper into the architecture, to the point where a request stops being a call and becomes a plan: agent orchestration, routing, workflows and state.

---

<nav class="pager">
  <a href="../">↑ Agentic AI in Production</a>
</nav>

## Comments

<script src="https://giscus.app/client.js"
        data-repo="uday-579/vajranex-knowledge-solutions"
        data-repo-id="R_kgDOUKZfgA"
        data-category="Announcements"
        data-category-id="DIC_kwDOUKZfgM4DEu_R"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
