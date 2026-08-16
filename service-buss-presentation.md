# Service Bus — Full Talk Script

This replaces the earlier shorter script. It's built to be genuinely comprehensive — by the end, your team should be able to reason about queues, topics, filters, sessions, failure handling, real deployment patterns, and what it costs.

**Two ways to run it** (see the timing table in Appendix B for the full breakdown):
- 🔵 **CORE** — the ~28–30 min version. Hit every 🔵 section, trim or skip 🟣 asides.
- 🟣 **EXTENDED** — everything, ~50–55 min. Use this if you've got a longer slot, or send the full doc as pre-read/follow-up material and deliver CORE live.

Don't feel obligated to read this out loud verbatim — it's dense on purpose so you have the detail in your back pocket if someone asks. Say it in your own words.

---

# 0. Cold open 🔵 (1 min)

**Say:**
"Quick show of hands — who's built or maintained an integration where one service calls another directly?" *(pause)* "Keep it up if you've then had to bolt a third, fourth, fifth service onto that same picture."

---

# 1. What is a service bus? 🔵 (3 min)

**Say:**
"That pain has a name, and it's what today is about. So before anything else, let me tell you in plain words what a service bus actually is.

A service bus is infrastructure that sits between your services so they never talk to each other directly. Instead of Payments calling Orders, calling Inventory, calling Notifications — each one individually — every service sends its messages to one place: the bus. The bus takes care of getting each message to whoever actually needs it.

If you want something close to home: think of it like a clearing house in banking. Banks don't settle every transaction bilaterally with every other bank they deal with — that doesn't scale past about three banks. They all connect to one clearing house, which routes, sequences, and reconciles on their behalf. A service bus does exactly that job for your microservices."

---

# 2. The building blocks 🔵 (core definitions ~4 min / 🟣 full depth ~10 min)

Walk through these *before* opening the demo — the demo will then feel like confirmation, not new information.

### 2.1 Producers & consumers 🔵
**Say:** "A service that sends a message is a *producer*. A service that receives one is a *consumer*, or when we're talking about topics, a *subscriber*. Nothing stops a service being both."

### 2.2 Queues 🔵
**Say:** "A queue is point-to-point. One message, one consumer — even if you've got five instances of a service all listening on the same queue, each message only gets picked up once. That's called *competing consumers*, and it's how you scale out work horizontally without duplicating effort."

🟣 **Extended aside — receive modes:** "There are two ways a consumer can read a message. **PeekLock** — the default — locks the message so nobody else can grab it while you're working on it, and you have to explicitly mark it complete when you're done, or it becomes visible again for retry. **ReceiveAndDelete** removes it from the queue immediately on receipt — faster, but if your service crashes mid-processing, that message is just gone. For anything where losing a message matters — which, in finance, is basically everything — you want PeekLock."

### 2.3 Topics & subscriptions 🔵
**Say:** "A topic is the pub/sub version. One producer sends to the topic once. Every *subscription* on that topic is effectively its own independent queue — it gets a full copy. Add a fifth subscriber tomorrow, and Payments' code doesn't change at all."

### 2.4 Filters 🔵 (concept) / 🟣 (syntax)
**Say:** "Subscriptions don't have to take everything on the topic. You can attach a filter so a subscription only receives messages that match certain criteria — that's content-based routing, and it's the difference between 'everyone gets everything' and 'the right service gets the right slice.'"

🟣 **Extended aside — the three filter types:**
- **Boolean** — `TrueFilter` (get everything, the default on a new subscription) or `FalseFilter` (get nothing).
- **SQL filter** — a SQL-like expression evaluated against message properties, e.g. `user.amount > 10000`. Flexible — supports `>`, `<`, `LIKE`, `IN` — but more expensive to evaluate at high volume.
- **Correlation filter** — matches specific property values using a fast hash lookup instead of expression evaluation. Less flexible, but the recommended choice once you're at real throughput.

"One gotcha worth flagging: filters evaluate message *properties* — metadata you attach — not the message body itself. If you want to route on a value, it needs to be a property, not something buried in the payload."

### 2.5 Sessions 🔵 (concept) / 🟣 (mechanics)
**Say:** "Sessions solve ordering. Messages that share a Session ID — say, all instructions for one account — get delivered strictly in order to a single consumer holding that session. Nothing else about the bus guarantees order on its own; sessions are how you opt into it."

🟣 **Extended aside:** "Your consumer has to be session-aware to use this — a normal receiver just sees the messages, not the grouping. Sessions are also a natural way to carry a bit of state across a related sequence of messages, since one consumer owns the whole session while it's active."

### 2.6 Delivery guarantees, retries, and dead-lettering 🔵
**Say:** "Service Bus is *at-least-once* delivery — never zero, but in rare failure windows a message could be delivered more than once. So your consumers need to be idempotent — processing the same message twice should be safe. When a consumer can't process a message, the bus retries automatically — the default is 10 attempts — and after that it's moved to a dead-letter queue rather than lost. Every queue and subscription gets one automatically, no setup required."

🟣 **Extended aside:** "Each delivery attempt holds a lock on the message for a configurable window — typically tens of seconds by default, renewable if your processing takes longer. If your consumer doesn't complete or renew in time, the lock expires and the message becomes visible again, counting as another delivery attempt."

### 2.7 Namespaces 🟣 (mention if time allows)
**Say:** "One more term you'll see constantly: a *namespace* is the actual deployed instance that hosts your queues and topics — the isolation boundary you'd draw around one environment or one business domain."

**Transition into the demo:**
"That's the full vocabulary. Now let's stop talking about it and watch one actually work."

---

# 3. Live demo 🔵 (target ~18–22 min)

Screen: `service-bus-demo.html`, Tab 1, Point-to-point mode, 5 services.

## 3.1 Tab 01 — Why (3 min)

**Say:**
"Five services, wired point-to-point right now — payments, orders, inventory, notifications, ledger."

**Action:** Click **+ Add service** three times.

**Say:**
"Watch what happens adding three more — nothing exotic, a normal quarter of feature work. We started at 10 connections. Now — 28. That's N times N-minus-1 over 2. Every new service is a new integration project with *every existing service*."

**Action:** Click **With a bus**.

**Say:**
"Now the bus goes in the middle. Same eight services. Eight connections."

## 3.2 Tab 02 — How it moves (7–8 min)

**Screen:** switch to Tab 2.

**Beat 1 — No bus vs. bus:**
**Action:** Send a message in No-bus mode, then switch to With a bus and send again.
**Say:** "No bus — Payments calls Orders, then Inventory, then Notify, then Ledger, one after another, and has to manage failures for each individually. With a bus — one call out, the bus fans it out. Payments doesn't know or care who's listening."

**Beat 2 — Queue vs. Topic:**
**Action:** Click **Queue**, send 2–3 times; then **Topic – broadcast**, send once.
**Say:** "Queue — only one consumer picks it up, rotating between them. This is for distributing work across a pool. Topic, no filters — every subscriber gets its own full copy. This is fan-out, for when several unrelated services all care about the same event."

**Beat 3 — Filtered routing:**
**Action:** Click **Topic – filtered**. Point at the tags under each node. Send tagged Standard, then tagged High-value.
**Say:** "Each subscription carries a filter — Inventory only wants standard, Ledger only wants high-value, Orders and Notify get everything. Tagged standard, Ledger's filter doesn't match, it's skipped — logged, not silently dropped. Tag it high-value instead, Ledger picks it up, Inventory doesn't."

🟣 **Extended aside:** "What you just saw is the demo's simplified version of a correlation or SQL filter — in Azure this would literally be something like `user.tag = 'high-value'` on the Ledger subscription. Same mechanism, same idea."

**Say:** "Hold that thought — it's exactly how we'll route a real payment instruction to manual review shortly."

## 3.3 Tab 03 — When it fails (6–7 min)

**Screen:** switch to Tab 3.

**Action:** Mark Ledger unhealthy, send a message.
**Say:** "Inventory and Notify get theirs immediately. Ledger — that's a retry. One, two, three, each failing. Then it's moved to the dead-letter queue automatically."

**Key line:** "Notice the healthy consumers weren't held up at all. One broken service does not take the other three down with it."

**Action:** Open **Review failed messages**. Retry the Ledger entry while it's still unhealthy (fails, stays). Mark Ledger healthy, send again, retry that entry (succeeds, resolves).
**Say:** "This inspector is what an on-call engineer actually works from — which service, when, why, and a way to retry or discard. Nothing here is invented for the demo."

🟣 **Extended aside:** "The demo dead-letters after 3 attempts for pacing — Azure's real default is 10. Same mechanism, just sped up for a live audience."

## 3.4 Tab 04 — Meet Azure Service Bus (3 min)

**Screen:** switch to Tab 4. Walk the four cards, then read the namespace note aloud.

**Say:** "Queue, Topic-with-filters, Retry-then-DLQ, Sessions — everything we just clicked through maps directly onto Azure Service Bus. None of it was invented for this demo."

**Transition:** "Let's ground this further — where do teams actually use this, and what does it cost to run?"

---

# 4. Real-world implementation patterns 🔵 (core: pick 2 / 🟣 full: all 4) (~5 min)

**1. Order orchestration (general pattern).** An order-processing flow — checkout, inventory, shipping, notifications — publishes to a topic; each concern subscribes independently, so adding a new downstream service is config, not code change in the producer.

**2. Fraud/compliance monitoring via filtered subscriptions.** This is the direct, financial-services version of what you just saw in the demo — transaction events flow into a topic, and a compliance subscription filters for the specific risk criteria (amount thresholds, flagged accounts, geographies) that need manual review, while the rest continue straight through.

**3. High-volume financial batch transaction processing.** Microsoft's own Azure Architecture Center publishes a reference architecture for exactly this — using AKS alongside Service Bus for high-volume, parallelizable transaction processing such as payroll, orders, and payments. Topics and queues can be replicated across regions so processing continues even if one region fails, and multiple compute clusters can read the same backlog in parallel. Microsoft explicitly calls this out as a fit for finance, and as a modern path for migrating mainframe-style batch workloads (the kind traditionally run on IBM MQ) onto Azure.

**4. Distributed transactions / outbox pattern for payments.** Where a single business action needs to update multiple services consistently — debit one ledger, credit another, notify a third — the bus becomes the reliable event backbone: built-in retries, duplicate detection, and (on Standard/Premium) transactional sends keep those steps consistent even when individual downstream calls fail.

*Further reading: search "Azure Architecture Center high-volume batch transaction processing" on Microsoft Learn for the full reference diagram — worth having open as a backup slide if someone asks "has anyone actually done this at scale."*

---

# 5. What this costs 🔵 (core: table + decision tree / 🟣 full: everything) (~4–5 min)

**Say up front:** "Pull these numbers up fresh in the Azure Pricing Calculator before you quote them to anyone — they move, and I'm giving you the shape of the cost, not gospel figures."

### The three tiers

| Tier | Supports | Billing model | Fits |
|---|---|---|---|
| **Basic** | Queues only — no topics, no subscriptions, no sessions | Pay per operation, very low unit cost | Simple single-queue workloads, dev/test |
| **Standard** | Everything in Basic, plus topics, subscriptions, sessions, transactions, duplicate detection (max message size 256 KB) | Small monthly base fee + per-operation charge | Most production pub/sub workloads — this is where most teams land |
| **Premium** | Everything in Standard, plus dedicated CPU/memory isolation, messages up to 100 MB, VNet/Private Link, Geo-Replication, JMS 2.0 | Billed per *messaging unit*, by the hour — a fixed cost whether idle or busy | High or predictable-throughput, latency-sensitive, or network-isolation/compliance-driven workloads |

🟣 **Extended aside — other cost drivers:**
- Concurrent connections beyond the included allowance cost extra per connection per day.
- Premium's per-messaging-unit price is a *fixed monthly commitment* — roughly in the high hundreds of dollars per unit per month — so it's a deliberate scaling decision, not something you switch on to "try it out."
- **You cannot upgrade a namespace in place.** Moving from Basic to Standard, or Standard to Premium, means creating a new namespace and migrating entities across — plan the tier before you build, not after.

### Simple decision tree
"Do you need topics and subscriptions? No → Basic. Yes → do you need VNet isolation, Private Link, or geo-disaster recovery, or guaranteed dedicated throughput? No → Standard. Yes → Premium."

🟣 **Extended aside — where it sits next to other Azure services:** "Event Grid is for lightweight, reactive event notifications, priced per event. Event Hubs is for high-throughput streaming and telemetry ingestion — think millions of events a second. Service Bus is for enterprise messaging that needs ordering, transactions, and complex routing. Plenty of real architectures use more than one of these together — they're not mutually exclusive."

**TCO note:** "The per-operation cost is rarely the real cost. Budget for monitoring dead-letter queue growth, alerting, and capacity planning — that's an ongoing operational commitment, not a one-time setup."

---

# 6. When to use it — and when not to 🔵 (~2–3 min)

- Don't reach for a bus to connect two services that will only ever be two services — that's just added latency and infrastructure for no benefit.
- It's an extra piece of infrastructure to run, monitor, and pay for.
- Async delivery means eventual consistency — the caller doesn't get an instant answer back.
- It earns its place exactly where you saw it today: many producers/consumers, a real need for isolation between them, and a need for auditable retry and failure handling.

---

# 7. Key takeaways 🔵 (~1 min)

**Say:**
"Three things to leave with. One — a bus decouples N services from an N-squared integration problem down to N. Two — queues, topics, filters, and sessions are four different tools for four different delivery shapes, not four ways of doing the same thing. Three — failure isn't a bug in this model, it's designed for: retries and dead-lettering happen automatically, and isolating one broken consumer from the healthy ones is the actual point."

---

# Appendix A — Anticipated Q&A (for the separate room, not counted in talk time)

**"How is this different from just calling a REST API directly?"**
A direct call is synchronous and tightly coupled — if the callee is down, the call fails immediately. A bus decouples that: the message is durably stored and delivered when the consumer is ready, and the sender never needs to know who or how many are listening.

**"What happens if the bus itself goes down?"**
Azure Service Bus is a managed, highly available service with an SLA (higher on Premium with geo-redundancy). For most teams the bus is materially more reliable than any individual service they'd otherwise be calling directly.

**"Why not just use Kafka?"**
Different center of gravity — Kafka is built around high-throughput, replayable event logs; Service Bus is built around enterprise messaging semantics like per-message locking, sessions, and transactional sends. Since we're an Azure shop, Service Bus also means one less piece of infrastructure to self-host and operate.

**"How do we test this locally / in CI?"**
The Service Bus emulator can run locally in a container for integration tests; unit tests typically mock the client SDK directly.

**"What's the latency overhead versus a direct call?"**
Single-digit to low double-digit milliseconds typically, on Standard/Premium — but you're trading a few milliseconds for durability and decoupling, which is the point, not a compromise.

**"How do we secure access to the bus?"**
Microsoft Entra ID (Azure AD) role-based access is the recommended approach over shared access keys — scoped per namespace, queue, or topic.

**"What monitoring do we get out of the box?"**
Built-in metrics in Azure Monitor — active message counts, dead-letter counts, incoming/outgoing rates — plus alerting you can wire up on DLQ growth specifically, which is usually the most actionable signal.

---

# Appendix B — Timing & two delivery paths

| Section | Tight (🔵 CORE only) | Full (🔵+🟣) |
|---|---|---|
| 0. Cold open | 1 min | 1 min |
| 1. What is a service bus | 3 min | 3 min |
| 2. Building blocks | 4 min (definitions only) | 10 min (with asides) |
| 3. Live demo | 15 min (use the cuts below) | 22 min (all asides included) |
| 4. Real-world patterns | 2 min (pick 2 of 4) | 5 min (all 4) |
| 5. Costs | 2 min (table + decision tree only) | 5 min (with asides) |
| 6. When to use / not | 2 min | 3 min |
| 7. Takeaways | 1 min | 1 min |
| **Total** | **~30 min** | **~50 min** |

*Appendix A is reference material for the Q&A room and isn't part of either total.*

### Cutting the demo down to 15 min (for the Tight path)
1. Skip the Queue-mode beat in Tab 2 — Broadcast alone makes the fan-out point.
2. In Tab 3, show one failed retry and describe the success case verbally rather than replaying the full mark-healthy-and-retry sequence.
3. Drop to 3 cards instead of 4 in the Azure tab.
4. Skip all 🟣 asides — they're for questions, not for the main narration.

### If the demo breaks live
Refresh the page — all state is local to the browser tab, so a reload resets everything in under a second. Keep that in your back pocket rather than apologizing for it.