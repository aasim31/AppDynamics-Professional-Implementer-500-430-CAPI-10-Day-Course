# Day 1 — APM Concepts and AppDynamics Architecture

[🏠 Home](../README.md) · [← Course outline](../course/00-course-outline.md) · [Day 2 →](day-02-pre-deployment-planning-and-sizing.md) · [📄 PDF version](../pdf/Day01_APM_Concepts_and_Architecture.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Session notes

**Goal:** speak AppDynamics fluently. Every later day builds on the terms in this session.

**Session plan (~2.5 hours):** six lesson blocks (~90 min) · SaaS lab (~45 min) · 10-question quiz with answer key.

## Block 1 — What APM does and how AppDynamics does it

APM (Application Performance Monitoring) answers three questions: **Is the app slow or failing? Where? Why?** AppDynamics answers them with **agents inside the running application**:

- **Java:** the agent is loaded with `-javaagent` and modifies the bytecode of known frameworks (servlets, JDBC, HTTP clients) as classes load. Your code does not change.

- **.NET:** the agent uses the CLR profiler interface to do the same.

- The agent times each request, follows it through your code, and sends metrics to the **Controller** every minute.

**Key idea:** agents _push_ data to the Controller. The Controller never connects to an agent, so firewalls only need to allow **agent → Controller** traffic, on 8090 (HTTP) or 8181 (HTTPS) for on-prem.

## Block 2 — The application model

Example: an online shop called **ShopApp**.

```
Business Application: ShopApp
├── Tier: Web       → Nodes: web01, web02   (2 Tomcat JVMs)
├── Tier: Orders    → Node:  orders01
└── Tier: Payments  → Node:  pay01
Backends (detected, not instrumented): MySQL "shopdb", HTTP "bank-gateway.com"
```

|**Term**|**Meaning**|**Exam trap**|
|---|---|---|
|Business application|The top-level container: everything that makes up one system||
|Tier|A group of nodes that do the same job (all web servers, for example)|A node belongs to exactly **one** tier|
|Node|One instrumented process: a single JVM, CLR app pool or Node.js process|Not a server. One server can run several nodes|
|Backend|Something your app calls that has no agent: a database, a remote service, a message queue|Found automatically from **exit calls**|

If a backend later gets its own agent, it stops being a backend and becomes a tier on the flow map.

## Block 3 — Business transactions (BTs)

A **BT** is one type of user request, such as "Checkout", "Login" or "Search", tracked end to end.

- **Entry point:** where the BT starts — servlet/HTTP, Spring bean, web service, JMS/message listener, POJO, ASP.NET, WCF.

- **Exit call:** a call out of a node — JDBC to a database, an HTTP call, a message sent to a queue.

- **Correlation:** when Web calls Orders over HTTP, the agent adds a header (`singularityheader`). The Orders agent reads it, so the request continues as the **same BT** across tiers.

**Default naming for Java servlets:** the first **two** URI segments. `/shop/checkout/confirm` becomes the BT `/shop/checkout`. To change how BTs are named or detected, write **custom match rules**.

**BT limits:** 50 per agent and 200 per application. Requests beyond the limit go into **"All Other Traffic"**. Common exam question: "New BTs are not appearing — why?" Usually the BT limit was reached.

## Block 4 — Snapshots, thresholds and diagnosis

- **Transaction snapshot:** a detailed record of one request, including a **call graph** (method-by-method timing), SQL, HTTP calls and errors.

- **Default thresholds (unverified):** commonly cited as slow = more than 3 standard deviations above baseline · very slow = more than 4 · stall = longer than 45 seconds. The current docs do not list these numbers — confirm them in your Controller under Configuration → Slow Transaction Thresholds.

- Snapshots are collected automatically for slow requests and errors, plus periodic samples.

- **Diagnostic session:** tells the agent to capture more snapshots for a while — started by hand or by a policy.

- **Flow map:** the live picture of tiers, backends, calls per minute and response time between them.

## Block 5 — Baselines, health rules, policies, actions

This chain is tested often. Learn it as one flow:

```
Metric → compared with a Baseline → Health Rule is violated → Policy matches → Action runs
```

|**Item**|**What it is**|**Example**|
|---|---|---|
|Baseline|The learned "normal" for a metric over a rolling time window; can follow daily or weekly patterns|Checkout is normally 400 ms on Monday mornings|
|Health rule|A condition on metrics: a fixed value or compared with the baseline|Avg response time &gt; 2 s, or &gt; 3 standard deviations above baseline|
|Policy|Links events (such as a health rule violation) to actions|"If Checkout goes critical, then …"|
|Action|What happens|Email, HTTP request (webhook), diagnostic session, remediation script, custom action|

## Block 6 — Platform components

![On-prem platform architecture: agents report to the Controller; browsers send beacons to the EUM Server](../images/day01-platform-architecture.png)

Classic on-prem architecture: all data flows into the Controller or the EUM Server. The Enterprise Console installs the Controller and Events Service, but not the EUM Server.

|**Component**|**Role**|**On-prem port**|
|---|---|---|
|Controller|UI, metric storage, baselines, alerting. App server + MySQL|8090 / 8181, MySQL 3388|
|App agents|Java, .NET, Node.js, PHP, Python… inside the app|—|
|Machine Agent|Server CPU, memory, disk, network; runs extensions|—|
|Database Agent|Monitors databases **remotely** through collectors|—|
|Browser / Mobile agents|Real-user data from browsers and phones, sent as **beacons**|—|
|EUM Server|Receives beacons; the Controller **polls** it for EUM metrics|7001 / 7002|
|Events Service|Stores Analytics, Database Visibility and EUM event data|9080 / 9081|
|Enterprise Console|Installs and manages the Controller and Events Service|9191|

### Deployment models

- **SaaS:** AppDynamics hosts the Controller.

- **On-prem:** you host everything.

- **Hybrid:** an on-prem Controller that uses the SaaS EUM Cloud and SaaS Events Service.

## Lab — SaaS trial (~45 min)

Log in to your SaaS Controller. If no application is reporting yet, use the demo or sample app it provides, or read through the steps now and run them after Day 6, when PetClinic is connected.

1. **Flow map:** Open an application and note every tier, how many nodes each has, and each backend.

2. **BTs:** Go to Business Transactions, sort by Average Response Time, and pick the slowest BT. What is its entry point type?

3. **Snapshot:** Open a slow snapshot, go to the call graph, and find the slowest method or SQL query.

4. **Alerting:** Open Alert & Respond → Health Rules. Open one default rule: what is its condition, which entities does it affect, and does it use a baseline?

5. **Notes:** Write down one thing that surprised you, to discuss at the start of Day 2.

## Day 1 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Which statement about communication between app agents and an on-prem Controller is correct?**

- A. The Controller polls each agent every minute for metrics
- B. Agents initiate connections to the Controller and push data
- C. Agents send data to the Events Service, which forwards it to the Controller
- D. Agents and the Controller open connections to each other in both directions

<details><summary>Show answer</summary>

**Answer: B** — Agents always start the connection (8090/8181 on-prem) and push metrics and snapshots. The Controller never connects to agents.

</details>

**Q2. A server runs three separate Tomcat JVMs, each with a Java agent. How many nodes does AppDynamics show?**

- A. 1, because there is one server
- B. 3, one per JVM
- C. 1 node with 3 tiers
- D. 0 until a Machine Agent is installed

<details><summary>Show answer</summary>

**Answer: B** — A node is one instrumented process, not a server. Three JVMs with agents = three nodes.

</details>

**Q3. Your Java app calls an Oracle database that has no AppDynamics agent. How does the database appear?**

- A. As a tier
- B. As a node
- C. As a backend detected from the exit call
- D. It does not appear until a Database Agent is installed

<details><summary>Show answer</summary>

**Answer: C** — Systems without an agent that your app calls are detected from exit calls and shown as backends.

</details>

**Q4. With default Java servlet naming, which BT name does /shop/checkout/confirm produce?**

- A. /shop
- B. /shop/checkout
- C. /shop/checkout/confirm
- D. checkout

<details><summary>Show answer</summary>

**Answer: B** — Servlet BTs are named from the first two URI segments by default.

</details>

**Q5. New BTs stopped appearing and requests show under 'All Other Traffic'. Most likely cause?**

- A. The Controller license expired
- B. The business transaction limit was reached
- C. The Events Service is down
- D. The Machine Agent is not installed

<details><summary>Show answer</summary>

**Answer: B** — BT limits (50 per agent, 200 per application) were reached, so new requests are grouped into All Other Traffic.

</details>

**Q6. What keeps a request as the same BT as it moves from the Web tier to the Orders tier over HTTP?**

- A. The Controller matches requests by timestamp
- B. The calling agent adds a correlation header that the receiving agent reads
- C. The Machine Agent links the two servers
- D. The Events Service joins the records later

<details><summary>Show answer</summary>

**Answer: B** — The upstream agent adds singularityheader; the downstream agent reads it. Unsupported protocols need custom correlation.

</details>

**Q7. What is the correct order of the alerting chain?**

- A. Action → Policy → Health rule → Metric
- B. Metric → Health rule violation → Policy → Action
- C. Policy → Metric → Action → Health rule
- D. Health rule → Action → Policy → Metric

<details><summary>Show answer</summary>

**Answer: B** — A metric is evaluated by a health rule; the violation event is matched by a policy, which triggers actions.

</details>

**Q8. Under default thresholds, a request is a stall when it takes longer than:**

- A. 3 standard deviations above baseline
- B. 4 standard deviations above baseline
- C. 45 seconds
- D. 10 seconds

<details><summary>Show answer</summary>

**Answer: C** — Commonly cited defaults: slow &gt; 3 SD, very slow &gt; 4 SD, stall &gt; 45 seconds. Unverified in current docs — confirm in your Controller (Day 2 lab task 1).

</details>

**Q9. In an on-prem deployment, where do browsers send Browser RUM beacons?**

- A. To the Controller on port 8090
- B. To the EUM Server
- C. To the Events Service on port 9080
- D. To the Java agent on the web server

<details><summary>Show answer</summary>

**Answer: B** — Beacons go to the EUM Server (7001/7002). The Controller polls the EUM Server for metrics.

</details>

**Q10. Which component does the Enterprise Console NOT install?**

- A. Controller
- B. Events Service
- C. EUM Server
- D. An embedded Events Service with Express Install

<details><summary>Show answer</summary>

**Answer: C** — The EUM Server has its own installer; the EC installs the Controller and Events Service only.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|Who starts the connection?|Agent → Controller (push)|
|Controller ports|8090 HTTP · 8181 HTTPS · 3388 MySQL|
|EUM Server ports|7001 HTTP · 7002 HTTPS|
|Events Service ports|9080 API · 9081 admin|
|Enterprise Console port|9191|
|Node =|One process (JVM / CLR app pool), not a server|
|Backend =|Called system without an agent, found from exit calls|
|Default servlet BT name|First two URI segments|
|BT limits|50 per agent · 200 per application → All Other Traffic|
|Thresholds (unverified)|Slow &gt; 3 SD · Very slow &gt; 4 SD · Stall &gt; 45 s — confirm in your Controller|
|Alerting chain|Metric → Health rule → Policy → Action|
|EC does not install|EUM Server|

**Homework before Day 2:** re-take the quiz until 10/10, memorise the port table, and bring your lab notes. Day 2 covers sizing and pre-deployment planning (25% of the exam).

---

[← Course outline](../course/00-course-outline.md) · [🏠 Home](../README.md) · [Day 2 →](day-02-pre-deployment-planning-and-sizing.md)
