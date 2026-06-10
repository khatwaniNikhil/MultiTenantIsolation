# Context - The Tenant Who Broke Everyone Else

Multi-tenant isolation, workload scheduling, and resource governance

This assessment is self-contained. Treat every ambiguity as part of the problem you are being assessed on.
 
- Sections A — Incident and failure analysis: Identify design failures; diagnose the architecture
Q1 – Q2: 10 min

- Section B — System design: Design isolation, scheduling, and detection mechanisms
Q3 – Q5: 22 min

- Section C — Constraints and trade-offs: Reason about scope, cost, and prioritisation
Q6 – Q7: 8 min
 
## Rules
- Open book — reference material, internet, and AI tooling all permitted.
- Do not request clarification. Ambiguity is intentional.
- Every question must be answered. Partial answers are accepted; blank answers are not.
- Write for a technically senior peer. No communication or stakeholder questions appear in this assessment.
- Precision over length. A concise, specific answer outscores a verbose one.
 
## Platform-agnostic
Describe mechanisms, contracts, and architectural properties — not specific products or languages. Naming a tool to make a point concrete is fine; your reasoning must not depend on it.

Scenario
Read in full before answering
All information needed to answer every question is on this page.

## Platform context
Lattice is a B2B SaaS platform with 600 tenants on a shared-infrastructure model. All tenants share the same compute pool, job queue, and database cluster. There are no per-tenant resource limits, no workload priority classes, and no tenant-attributed resource monitoring.

## Architecture
- Single job queue. One priority level. FIFO dispatch.
- Worker pool: 40 nodes, shared across all tenants.
- Database: shared cluster, no per-tenant connection limits or query quotas.
- Autoscaler: triggers at 80% average worker CPU. Adds nodes in batches of - Spin-up time: ~4 min.
- Monitoring: fleet-level CPU, queue depth, and error rate. No tenant-attributed metrics.
- Alert: global CPU >90% for >2 min. No queue-saturation or per-tenant alert.
 
## The incident
Saturday 02:14 AM: The platform became unresponsive for all 600 tenants for 23 minutes. Root cause: Tenant A ran a scheduled data-export job that consumed the entire worker pool. The job was legitimate and permitted by the ToS.
 
## Tenant A profile
- 22% of platform ARR.
- Job: 4.2 M record export, runs weekly.
- Normally scheduled: 02:14 PM. This run: 02:14 AM (timezone misconfiguration on their side).
- Job spawned 38 parallel worker tasks — saturating 95% of the fleet.
- Interactive requests from all other tenants queued behind the export tasks for the full 23 minutes.

## Incident data
What did not exist at incident time
-   Per-tenant queue visibility or task attribution.
-    Workload classification (interactive vs batch).
-    Per-tenant CPU or worker consumption metrics.
-    Queue saturation alert (only global CPU was monitored).
-    Capacity reservation for interactive traffic.
 
## Constraints you must design within
1. Existing tenants must not change how they submit work — no client-side changes permitted.
2. Tenant A's weekly export is a legitimate and contractually permitted workload.
3. Any new isolation mechanism must be implementable without migrating to a per-tenant database or compute cluster.
4. The autoscaler behaviour is fixed — spin-up takes ~4 minutes.
5. You may add new queue infrastructure but cannot replace the existing one in a single release. 

<br>

# Question 1
The post-incident report frames this as 'Tenant A scheduled a job at the wrong time. Explain precisely why that framing is incorrect. Then enumerate every design property of the current architecture that was required to be present for a single tenant's legitimate workload to cause a platform-wide outage. For each property, state whether it is a missing control, an absent signal, or a flawed assumption baked into the original design.
Structured list. Each property must be classified as: missing control / absent signal / flawed assumption.
[Max 280 words]


# Answer 1
Tenant A scheduled a job at the wrong time' framing is incorrect because the actual issue is no per-tenant resource limits leading to starvation for other tenants by one tenant.

## Design properties:
### Missing Control
1. No control around worker allocation quotas per tenant for job execution missing. 

### Absent signal
1. Interactive versus Batch workload segregation is missing.
2. Separate worker pool for interactive and batch workload is missing.  

### Flawed assumption
1. Long running/batch jobs are assumed to be executed only in off peak hours.
2. Autoscaling will lead to extra worker pool getting utilised for other pending jobs.

<br>


# Question 2
The CPU alert fired 17 minutes into a 23-minute outage — 6 minutes before recovery, with no actionable recovery window.
## 2a
Explain why a fleet-level CPU threshold is structurally unsuitable as the primary alert for this failure class.
## 2b 
Specify the exact signal and threshold that would have detected this event within 3 minutes of onset — including the data source, the computation, and the condition that fires the alert.
## 2c 
If the detection had fired at 02:17, what automated action (if any) should the system have taken before a human was paged?
[Max 280 words.
Part (b) must be a complete alert specification — signal, source, threshold, condition — not a description of one.]

# Answer 2

## 2a 
Fleet-level CPU threshold does not convey any information around one tenant causing starvation to other tenants. Also it does not convey any info around tenant resource consumption is as per allowed quota or not.

## 2b
 Time to drain(TDD) alert. Formula:

    TDD(minute) = Queue depth(messages)/throughput(messages per minute)

 - Data source = Queue
 - Computation = Queue depth
 - Condition that fires the alert - Rate of queue depth decrease per minute not meeting configured threshold.

## 2c
If the detection had fired at 02:17, automated action from the system could be to **Auto kill/poison pill based graceful shutdown of offending tenant 1 task**, push current task to some retry queue for scheduled execution in off peak hours

<br>

# Question 3
Design a workload isolation mechanism for the job queue that satisfies all five constraints on the scenario page. Your design must:
## 3a 
Prevent any single tenant's tasks from consuming more than a defined share of the worker pool at any point in time — state the maximum share and the enforcement mechanism,
## 3b 
ensure interactive requests from all tenants receive a guaranteed minimum service level even when the queue is saturated with batch work — specify what 'guaranteed minimum' means numerically, and
## 3c
describe exactly what happens to Tenant A's export job under this design — does it run slower, get queued, get rejected, or something else?
[Max 400 words. All three parts required. Part (a) must name the enforcement mechanism, not just the limit. Describe Tenant A's experience specifically.

# Answer 3
## 3a
### Per tenant maximum share formula: 

    Min(SingleTenantCapacityCeiling, 100/N) 
    
    where N is unique tenants with jobs waiting in the queue 

This will ensure max share never goes above configured "SingleTenantCapacityCeiling" and dynamically adjusts basis tenants whose jobs are waiting in the queue.

### Enforcement mechanism:
**Use proxy (like envoy) for isolation between queue and worker pool**. Workers do not poll the raw queue anymore; they poll the Isolation Proxy. Current tenant utilised share is tracked within proxy using **redis hash**. In case tenant utilisation reaches to maximum share as per above formula, push the message
in low priority queue.

## 3b
From raw existing queue we will consumer, segregate and push interactive and batch messages to separate queues. At proxy level, We will set **SLA around "Time to drain"(minutes) for interactive queue and best effort basis for batch queue**. Proxy will also keep min. guaranteed worker pool availability share for interactive tasks.   

## 3c
Considering above design, based on other tenant job events waiting in the queue, tenant A's job will be accordingly allocated lesser capacity that in case of outage. In nutshell, it **will run slower** and **tenant A will receive a delayed response**.

<br>

# Question 4
The autoscaler added 4 nodes at 02:20 but they were immediately consumed by Tenant A's tasks, providing no relief to other tenants.

## 4a
 Identify the design assumption that caused the autoscaler to be ineffective in this scenario.
## 4b
Design a capacity reservation model that guarantees a minimum number of workers are always available for interactive traffic — even during a fleet-wide batch surge — without disabling or replacing the autoscaler. Specify: the reservation size, how it is enforced at the scheduler level, and how reservation headroom is reclaimed when interactive load is low.Max 320 words. Specify the reservation size as a formula or rule, not just a concept.]


# Answer 4
## 4a
**Design assumption** - Extra capacity made available via auto scaling will get utilised for pending jobs was ineffective in this scenario.

## 4b 
1. **Core Idea**: Dual-Lane Routing. Sort incoming jobs inside the Envoy Proxy into two logical lanes: **Interactive Lane(high priority)** and **Batch Lane(Low-priority)**

2. **Numerical Guarantee**: **Rule**: 20% of active workers are reserved for the Interactive Lane when jobs are waiting. **Floor**: A hard minimum of 5 workers remains locked for interactive tasks during low-traffic hours.

3. **Scheduler Enforcement**: When a worker asks Envoy for a new job, Envoy runs a real-time check:

**Condition**: If interactive jobs are waiting AND under 20% of workers are processing them, Envoy forces the worker to take an interactive job.
Result: Batch jobs are blocked from that worker until the 20% floor is met.

**Efficient Headroom Reclaim**

***Borrowing***: If the interactive queue is empty, batch jobs can borrow 100% of the fleet.

**Preemption**: Batch tasks must run with checkpoints at regular intervals. 

**The Handover**: The moment an interactive job arrives, Envoy intercepts the next finishing worker (max 3-second wait) and hands it the real-time job.

# Question 5
The current database cluster has no per-tenant limits. A 4.2 M-record export query, even if its worker tasks are rate-limited by your Q3 design, will generate substantial database load. Design a per-tenant database resource control model that:
## 5a
 bounds the query throughput a single tenant can sustain at the database layer, independently of how their tasks are scheduled at the worker layer,
## 5b
tiers the limits — different bounds for different contract tiers — and specifies where in the request path enforcement occurs, and
## 5c
handles the case where Tenant A's export query is a single long-running transaction rather than many short queries.
[Max 350 words. Part (c) must be addressed — a model that only handles short-query workloads is incomplete.]

## Answer 5
## 5a
**Define db load into cost units(CU)**. 

1 indexed row read = 1CU

1 non indexed row read=10CU.

**Set tenant level cost units budget**. Track the current tenant level cost unit utilisation in redis. If the tenant has utilised the allocated budget, the worker pauses, sleeps for 100ms, and then retries.

## 5b
Tenants are assigned a dynamic Token Bucket in Redis based on their subscription tier:

| Contract Tier | Max DB Throughput Ceiling (CUs/sec) | Max Burst Allowance (CUs) |
| --- | --- | --- |
Enterprise		|50,000 (Full speed execution)			|100,000
Growth			|15,000 (Paced throughput)			    |30,000
Free/Self-Serve	|2,500 (Strictly throttled)			    |5,000

## 5c
For long running queries use **cursor based pagination** and before making any specific page sql query from application tier, **check tenant level cost units available or preempt** the transaction and retry later.

<br>

# Question 6
You have two sprints (4 weeks) of engineering capacity before a product feature freeze. Your designs in Q3, Q4, and Q5 collectively represent more than 4 weeks of work.
## 6a
Rank your three designs by implementation priority — the order in which you would ship them — and state the criterion you used to rank them.
## 6b
Describe what the blast-radius protection looks like after each sprint, so your engineering manager can communicate the incremental risk reduction to the CTO. After sprint 1: what is still possible that was not before? After sprint 2: what is additionally protected?
[Max 300 words. Ranking criterion must be stated explicitly. The sprint-by-sprint risk profile must be described, not implied.]


# Answer 6
## 6a
**Implementation priority** is as follows:
- Sprint 1. Db side per tenant quotas via cost units (via redis)
- Spring 2,3 Tenant specific worker pool quotas via proxy
- Sprint 4. Separate job queues and worker quotas for interactive and batch workloads.

**Reason/Rationale**: Priortised in terms of reducing the risk for given outage and reduce the overall blast radius.

## 6b
- Post sprint 1: We are protected from db side resource overuse by single tenant outages
- Post sprint 2,3 It will significantly reduce the risk of current discussed outage
- Post spring 4: Interactive jobs throughput will be maintained under the given SLA.

<br>

# Question 7
Your Q3 isolation design caps Tenant A's worker share. As a direct consequence, their weekly 4.2 M-record export will take longer under load. Estimate the worst-case latency increase as a function of the cap you chose — show the reasoning, not just the number. Then propose a scheduling design that allows Tenant A's export to complete in approximately the same wall-clock time as today while still honouring the worker cap during business hours. Your proposal must not require any changes to Tenant A's client code.
[Max 300 words.
The latency estimate must show the reasoning. The scheduling design must state when the export runs and how the cap interacts with that timing.]

# Answer 7
Refer 3a 
 
    Per tenant maximum share: Min(SingleTenantCapacityCeiling, 100/N)
Worst case latency will increase by the **percentage of (PerTenantMaximumShare of total capacity)**

Scheduling design to keep tenant A's export the same wall clock time
1) **Core Business Hours Window**: Majority of quotas allocated to interactive
   Off-Peak Business hrs: Majority of quotas allocated to batch

2) Also tenant level worker and db cost unit **quotas are relaxed in off-peak business hours**.
