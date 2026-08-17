
# 1. What is a service bus?

A service bus is infrastructure that sits between your services so they never talk to each other directly. Instead of multiple services calling one another individually, each service sends its message to one place the buss. The Buss takes care of each message to whoever actually needs it.

If you want something close to home: think of it like a clearing house in banking. Banks don't settle every transaction bilaterally with every other bank they deal with — that doesn't scale past about three banks. They all connect to one clearing house, which routes, sequences, and reconciles on their behalf. A service bus does exactly that job for your microservices."

A couple of things worth being precise about, because the term gets thrown around loosely.
- A service bus is **not** a database, 
- messages are not meant to live there forever, they are meant to be picked up and gone. 
- It's **not** an event-sourcing system, 
- it's not the permanent, replayable source of truth for your data, even though it can feed one.
- and it's **not** just a fancier HTTP call 
The entire point is that sender and receiver are decoupled in _time_, not just in address. 

One confusion worth heading off directly: 
- This is not the config-heavy, 2000s-era 'Enterprise Service Bus' - the on-prem transformation-and-orchestration monolith some of you may have encountered. 
- Azure Service Bus is a much narrower, cloud-native messaging primitive. Same name, genuinely different thing. 
- And if anyone asks how it's different from a message queue, an event bus, or a streaming platform like Kafka: 
	- a plain queue has no topics or routing; an event bus is built around a high-throughput, replayable log rather than per-message lock-and-complete semantics; 
	- and an API gateway is synchronous - none of the durability or decoupling we're about to see applies to it at all.

---

# 2. The building blocks 

### 2.1 Producers & consumers 

- A service that sends a message is a _producer_. 
- A service that receives one is a _consumer_ - or when we're talking about topics, a _subscriber_
- Nothing stops a service being both."

### 2.2 Queues 

- A queue is point-to-point. One message, one consumer - even if you've got five instances of a service all listening on the same queue, each message only gets picked up once. That's called _competing consumers_, and it's how you scale out work horizontally without duplicating effort.

There are two ways a consumer can read a message. 
- **PeekLock**: locks the message so nobody else can grab it while you're working on it, and you have to explicitly mark it complete when you're done, or it becomes visible again for retry.
- **ReceiveAndDelete**: removes it from the queue immediately on receipt - it's faster, but if your service crashes mid-processing, that message is just gone. 
- For anything where losing a message matters - which, in our financial world, is basically everything  - you want PeekLock.

### 2.3 Topics & subscriptions 

- A topic is the pub/sub version. One producer sends to the topic once. 
- Every _subscription_ on that topic is effectively its own independent queue - It gets a full copy.
- And if you add a fifth subscriber tomorrow, and the services code doesn't change at all.

### 2.4 Filters 

- Subscriptions don't have to take everything on the topic. 
- You can attach a filter so a subscription only receives messages that match certain criteria, that's content-based routing, 
- Filters are the difference between 'everyone gets everything' and 'the right service gets the right slice.'"

**Extended aside — the three filter types:**

- **Boolean** 
	- TrueFilter (get everything, the default on a new subscription) 
	- FalseFilter (get nothing).
- **SQL filter** 
	- a SQL-like expression evaluated against message properties, 
		- e.g. `user.amount > 10000`. Flexible - supports `>`, `<`, `LIKE`, `IN` — but more expensive to evaluate at high volume.
- **Correlation filter** 
	- matches specific property values using a fast hash lookup instead of expression evaluation. Less flexible, but the recommended choice once you're at real throughput.

Filters evaluate message _properties_ - metadata you attach - not the message body itself. 
If you want to route on a value, it needs to be a property, not something buried in the payload.

### 2.5 Sessions 

If you need message to be in order, Sessions solve ordering. 
- Messages that share a Session ID - say, all instructions for one account get delivered strictly in order to a single consumer holding that session. Nothing else about the bus guarantees order on its own; sessions are how you opt into it.

### 2.6 Delivery guarantees, retries, and dead-lettering 

There are three delivery semantics in messaging generally: 
- at-most-once - fast, but a message can just vanish on failure; 
- at-least-once - never zero, but a message could arrive twice in rare failure windows; 
- exactly-once - the one everyone wants, genuinely hard to guarantee end-to-end, and usually comes with a performance cost where it's offered at all. 

- Service Bus's default is at-least-once. That's why the idempotency point matters 
- Design consumers so processing the same message twice is safe, and you get the practical benefit of exactly-once without paying for it. 
- When a consumer can't process a message, the bus retries automatically - the default is 10 attempts - and after that it's moved to a dead-letter queue rather than lost. 
- Every queue and subscription gets one automatically, no setup required.


# 3. Live demo 

## 3.1 Tab 01 — Why 

Four separate calls, and Payments' own code has to know about every one of those four services individually, and handle it if any single one of them is down. Now the exact same message, but with a bus sitting in the middle. One call out  that's it. The bus takes it from there and gets it to whoever's listening. Payments doesn't know, and genuinely doesn't need to care, who's actually on the other end.

Queue vs Topic
So how does the bus actually hand this off? There are two ways, and they exist for genuinely different reasons. 

This is a Queue. Send it again -and again. Only one consumer ever picks up each message, and it rotates between them. The queue only ever sends this one message to one service at a time. That's what you reach for when you've got a pool of workers and you want the load spread across them — you don't want five services all processing the same order. 

Now flip it to a Topic, no filters yet. Same single message - but this time every subscriber gets its own full copy. With the bus, we're not stuck delegating to just one service — we can hand this one message to every service that cares about it, all at once. That's fan-out, and it's what you use when several unrelated services all need to react to the same event.

**Filtered routing:**

Here's where it actually earns its keep for something like ours. Each subscription can carry its own filter — you can see the small tags sitting under each node. Inventory only wants standard messages, Ledger only wants high-value ones, Orders and Notify get everything regardless.  Tag this one standard and send it - it goes to Orders, Inventory, Notify. Ledger's filter doesn't match, so it's skipped — and notice that's logged, not silently dropped.  Now tag the exact same kind of message high-value instead — Ledger picks it up this time, Inventory doesn't. One bus, one topic, but different consumers see a different slice of the traffic depending on what's actually inside the message.

Tab 3
Orders is sending here, and I've just marked Ledger unhealthy - picture its database just went down. Watch what happens now. Inventory and Notify get their copy immediately, no problem at all. Ledger - watch closely - that's a retry. One, two, three attempts, each one failing. And after the third, it gets moved into the dead-letter queue automatically - nobody wrote code for that, it's just how the bus behaves by default."

The important thing to notice: Inventory and Notify weren't held up for even a second while this was happening. One broken service does not take the healthy ones down with it - that isolation is the entire point of doing it this way.

This is the part that actually matters day to day - what do you _do_ with a message once it's failed? Here's an inspector: which service, when it failed, why it failed.  Ledger's still down, so retrying it now - as expected - just puts it straight back in the queue, unresolved.  Now that Ledger's actually healthy again, the same retry succeeds, and the message is resolved. Submit, fail, retry, resolve — that's the real lifecycle a message goes through, and every step of it is exactly what an on-call engineer does for real, not something invented for this demo."

Same bus, same ideas we just spent ten minutes on - but now let's actually use it. This is a payment instruction pipeline. A client submits an instruction, it lands on a topic, and three subscriptions each pick up whatever's relevant to them: Compliance, Ledger, Notifications. ]** Compliance already has a filter sitting on it - anything over ten thousand dollars gets flagged for manual review automatically. Nobody has to remember to check that by hand, it just happens. Let's send a standard instruction - twenty-five hundred dollars. Watch it move — Ledger gets it, Notifications gets it. Compliance… skipped. Below the threshold, auto-cleared, and notice it's logged as skipped, not silently dropped.  Now the same account, but twenty-five thousand dollars this time. Ledger and Notifications, exactly the same as before. But Compliance lights up amber — flagged for review — and the log shows you exactly which filter matched."

Notice I picked an account before I sent that one. Let's see why that actually mattered.

Tab 5

This is Sessions, made concrete. Two accounts, each with three pending instructions sitting in order — a deposit, then two withdrawals. 
Deposit first — balance goes to a thousand. Withdrawal — five hundred. 
Withdrawal again — two hundred. Balance stays positive the entire way through, because the order was guaranteed, not just hoped for. And notice — ACC-2210 hasn't moved at all. It's not waiting on ACC-1044, and ACC-1044 isn't waiting on it. 
Each account is its own session, processed completely independently, but strictly in order within itself."

If these had arrived out of order — a withdrawal landing before the deposit it actually depends on this account would have gone negative for however long that gap lasted. Sessions don't just make that unlikely. They make it structurally impossible.

Tab 7

Compliance's screening service just went down. Let's send a high-value instruction anyway and see what happens. Watch — Ledger and Notifications, delivered immediately, completely unaffected. Compliance — retry, retry, retry, three attempts, each one failing, then dead-lettered. Same mechanism you already saw in Tab 3, same reason it matters here: one broken dependency does not stall the rest of the pipeline, even when real money's involved.  Let's retry it while Compliance is still down — as expected, it just goes straight back into the dead-letter queue, unresolved. Now fix the underlying issue, mark it healthy, retry — resolved. Submit, route, fail, recover — that's the exact lifecycle a real payment instruction goes through on this platform, and every step of it is something we just clicked through ourselves, not a diagram."

**Transition:** "Two of the real-world patterns I want to mention, you've already seen live. Let's cover the two you haven't."
# 4. Real-world implementation patterns 

**1. High-volume financial batch transaction processing.** Microsoft's own Azure Architecture Center publishes a reference architecture for exactly this — using AKS alongside Service Bus for high-volume, parallelizable transaction processing such as payroll, orders, and payments. Topics and queues can be replicated across regions so processing continues even if one region fails, and multiple compute clusters can read the same backlog in parallel. Microsoft explicitly calls this out as a fit for finance, and as a modern path for migrating mainframe-style batch workloads (the kind traditionally run on IBM MQ) onto Azure.

**2. Distributed transactions / outbox pattern for payments.** Where a single business action needs to update multiple services consistently — debit one ledger, credit another, notify a third — the bus becomes the reliable event backbone: built-in retries, duplicate detection, and (on Standard/Premium) transactional sends keep those steps consistent even when individual downstream calls fail. This is effectively what Tab 7 showed in miniature — the DLQ and retry logic are the same primitives that make this pattern safe at scale.

---

# 5. What this costs 
### The three tiers

|Tier|Supports|Billing model|Fits|
|---|---|---|---|
|**Basic**|Queues only — no topics, no subscriptions, no sessions|Pay per operation, very low unit cost|Simple single-queue workloads, dev/test|
|**Standard**|Everything in Basic, plus topics, subscriptions, sessions, transactions, duplicate detection (max message size 256 KB)|Small monthly base fee + per-operation charge|Most production pub/sub workloads — this is where most teams land|
|**Premium**|Everything in Standard, plus dedicated CPU/memory isolation, messages up to 100 MB, VNet/Private Link, Geo-Replication, JMS 2.0|Billed per _messaging unit_, by the hour — a fixed cost whether idle or busy|High or predictable-throughput, latency-sensitive, or network-isolation/compliance-driven workloads|

**Extended aside — other cost drivers:**

- Concurrent connections beyond the included allowance cost extra per connection per day.
- Premium's per-messaging-unit price is a _fixed monthly commitment_ — roughly in the high hundreds of dollars per unit per month — so it's a deliberate scaling decision, not something you switch on to "try it out."
- **You cannot upgrade a namespace in place.** Moving from Basic to Standard, or Standard to Premium, means creating a new namespace and migrating entities across — plan the tier before you build, not after.

### Simple decision tree

Do you need topics and subscriptions? 
- No → Basic. 
- Yes → do you need VNet isolation, Private Link, or geo-disaster recovery, or guaranteed dedicated throughput? 
	- No → Standard. 
	- Yes → Premium."

**Extended aside — where it sits next to other Azure services:** 
 - Event Grid is for lightweight, reactive event notifications, priced per event. 
 - Event Hubs is for high-throughput streaming and telemetry ingestion — think millions of events a second. 
 - Service Bus is for enterprise messaging that needs ordering, transactions, and complex routing. 
 - These services are commonly used together across architectures, they're not mutually exclusive."

---

# 6. When to use it - and when not to 

- Don't reach for a bus to connect two services that will only ever be two services — that's just added latency and infrastructure for no benefit.
- It's an extra piece of infrastructure to run, monitor, and pay for.
- Async delivery means eventual consistency - the caller doesn't get an instant answer back.
- It earns its place exactly where you saw it today: many producers/consumers, a real need for isolation between them, and a need for auditable retry and failure handling.

---

# 7. Key takeaways 🔵 (~1–2 min)

Three things to leave with. 
- One — a bus decouples N services from an N-squared integration problem down to N. 
- Two — queues, topics, filters, and sessions are four different tools for four different delivery shapes, not four ways of doing the same thing. 
- Three — failure isn't a bug in this model, it's designed for: retries and dead-lettering happen automatically, and isolating one broken consumer from the healthy ones is the actual point.

---

# Appendix A — Anticipated Q&A (for the separate room, not counted in talk time)

**"How is this different from just calling a REST API directly?"** A direct call is synchronous and tightly coupled — if the callee is down, the call fails immediately. A bus decouples that: the message is durably stored and delivered when the consumer is ready, and the sender never needs to know who or how many are listening.

**"What happens if the bus itself goes down?"** Azure Service Bus is a managed, highly available service with an SLA (higher on Premium with geo-redundancy). For most teams the bus is materially more reliable than any individual service they'd otherwise be calling directly.

**"Why not just use Kafka?"** Different center of gravity — Kafka is built around high-throughput, replayable event logs; Service Bus is built around enterprise messaging semantics like per-message locking, sessions, and transactional sends. Since we're an Azure shop, Service Bus also means one less piece of infrastructure to self-host and operate.

**"How do we test this locally / in CI?"** The Service Bus emulator can run locally in a container for integration tests; unit tests typically mock the client SDK directly.

**"What's the latency overhead versus a direct call?"** Single-digit to low double-digit milliseconds typically, on Standard/Premium — but you're trading a few milliseconds for durability and decoupling, which is the point, not a compromise.

**"How do we secure access to the bus?"** Microsoft Entra ID (Azure AD) role-based access is the recommended approach over shared access keys — scoped per namespace, queue, or topic.

**"What monitoring do we get out of the box?"** Built-in metrics in Azure Monitor — active message counts, dead-letter counts, incoming/outgoing rates — plus alerting you can wire up on DLQ growth specifically, which is usually the most actionable signal.


# Appendix A — Quick-reference glossary (not for live delivery)

Terms your team will run into that aren't fully explained in the main talk — hand this out or link it afterward rather than reading it aloud.

|Term|Plain meaning|Why it matters|
|---|---|---|
|Correlation ID|An identifier you attach to tie related messages together across a flow|Essential for tracing one instruction's path across producer, bus, and every subscriber|
|Message envelope|The metadata wrapper around a message body (headers, properties, timestamps)|Filters evaluate the envelope, not the body — this is where SessionId and your filterable properties live|
|Visibility timeout|Other systems' name for what Service Bus calls Lock Duration|Useful if anyone on the team has AWS SQS background — same idea, different vendor term|
|Idempotency key|A value your consumer uses to recognize "I've already processed this"|What makes at-least-once delivery safe in practice|
|Poison message|A message that fails processing every single time, no matter how many retries|The formal name for what Tabs 3 and 7 demonstrate handling|
|Schema registry / schema versioning|A central place that governs what a valid message looks like, and how it's allowed to change|Prevents a producer's format change from silently breaking every consumer|
|Backpressure|Mechanisms that slow producers down when consumers can't keep up|The bus provides durable buffering; true backpressure back to the producer is still something you design for|
|Broker quota|Throttling limits the platform enforces to protect itself|Shows up as throttling errors under load — budget for it, don't fight it|
|Circuit breaker|A pattern that stops calling a dependency once it's clearly failing, instead of retrying into it|Complements MaxDeliveryCount — protects the _caller_ as well as isolating the failure|
|Message batching|Grouping multiple sends/receives into one network round trip|Improves throughput at the cost of slightly higher tail latency on individual messages|
|Compensation|The "undo" step in a multi-service workflow (saga pattern) when a later step fails|Relevant once you're chaining several bus-driven steps into one business transaction|
|WORM storage|Write-once-read-many storage — immutable, for audit trails|Relevant for compliance retention requirements, separate from the bus's own message retention|
|Replay|Reprocessing messages from an archive or dead-letter queue|Only safe if your consumers are idempotent — see above|
|Access control list / RBAC|Fine-grained permissions on who can send to or receive from a given entity|In Azure, this is Microsoft Entra ID roles scoped per namespace, queue, or topic|
|Sidecar pattern|Running a local proxy alongside a service to handle its messaging concerns|An architectural option, not something Service Bus requires|

---

# Appendix B — Common pitfalls (not for live delivery)

Curated from real-world "what breaks" lists, trimmed to what's most relevant to this environment. Good material for a follow-up doc or onboarding note.

| Symptom                         | Root cause                                          | Fix                                                                                                                                  |
| ------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Out-of-order processing         | No session key used                                 | Use a session/partition key — this is exactly what Tab 6 demonstrates                                                                |
| Sudden dead-letter spike        | Schema change or malformed messages upstream        | Validate schema before send, patch the consumer, then safely reprocess the DLQ                                                       |
| Duplicate processing            | At-least-once delivery without an idempotency check | Implement an idempotency key and de-dupe on it                                                                                       |
| Queue depth constantly climbing | Consumer capacity too low, or consumers crashed     | Autoscale consumers; alert on sustained (not instantaneous) growth                                                                   |
| Throttling errors               | Exceeding broker quotas under load                  | Rate-limit producers, implement exponential backoff                                                                                  |
| Messages silently going missing | Filter misconfiguration, or DLQ not monitored       | Log skipped/unmatched messages explicitly; alert on DLQ growth specifically                                                          |
| Secrets or PII in logs          | Logging raw message headers or bodies for debugging | Sanitize logs, redact sensitive fields before they ever hit a log sink — a compliance-relevant one for this environment specifically |




### 2.8 What actually breaks without this 

Before we get into the demo, it's worth naming what actually goes wrong in production — because every feature you're about to see is a direct answer to one of these. This isn't hypothetical; it's the standard on-call list for anyone running a bus."

- **Backlog explosion** — a consumer goes down, queue depth climbs fast, and if it climbs long enough you start hitting retention limits and losing messages. _Answer: competing consumers, autoscaling._
- **Poison messages** — a malformed or unprocessable message gets retried forever and, on a session-ordered queue, can block everything behind it. _Answer: MaxDeliveryCount + dead-lettering — exactly what Tabs 3 and 7 are about to show._
- **Hot spots** — traffic concentrates on one partition or one session, and that one path gets slow while everything else is fine. _Answer: pick a well-distributed session key — account ID works because you have many of them, not because it's the obvious choice._
- **Misconfigured routing filters** — a filter typo silently sends messages to the wrong subscription, or nowhere at all. _Answer: filters fail silently by nature — that's exactly why the demo logs a "skipped" line instead of hiding it._
- **Credential rotation failures** — producers or consumers lose the ability to authenticate mid-rotation and processing quietly stops. _Answer: not a bus feature — an operational discipline point, automate rotation and monitor for auth errors specifically._

🟣 **Extended aside — how you'd actually notice these:** "In production you'd track this with four numbers: delivery success rate, queue depth, oldest message age, and dead-letter rate. Alert on sustained trend, not instantaneous spikes — a one-second blip in queue depth is noise, a five-minute climb is the backlog explosion starting."

---
