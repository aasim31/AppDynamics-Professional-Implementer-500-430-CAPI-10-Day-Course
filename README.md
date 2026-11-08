# AppDynamics Professional Implementer (500-430 CAPI) — 10-Day Course

Free, structured study notes for the **Cisco AppDynamics Professional Implementer** exam (**500-430 CAPI v1.0**), organised as a 10-day classroom-style course. Each day covers a slice of the official exam blueprint with a lesson, a hands-on or "paper" lab, a 10-question practice quiz with answers, a cheat sheet, links to the official documentation, and model answers for the lab.

> Built while preparing for the exam myself. If it helps you, please ⭐ the repo and share it.

-----

## Exam at a glance

| Item | Detail |
|---|---|
| Exam | 500-430 CAPI — Cisco AppDynamics Professional Implementer v1.0 |
| Duration / price | 90 minutes · US$300 · English · Pearson VUE (online or test centre) |
| Question count / pass mark | Not published by Cisco for v1.0 (the older AppDynamics guide listed 50–55 questions and a 64% pass mark) |

| # | Domain | Weight | Course days |
|---|---|---|---|
| 1 | Pre-Deployment Planning | 25% | 2, 5, 6 |
| 2 | Platform Installation and Administration | 25% | 3, 4, 5 |
| 3 | Agent Configuration and Installation | 25% | 5, 6, 7 |
| 4 | Post-Deployment Verification | 5% | 3, 8 |
| 5 | APIs | 20% | 9 |

Official exam page: <https://cisco.com/site/us/en/learn/training-certifications/exams/capi.html> · Exam topics (PDF): <https://bxin.cisco.com/c/dam/en_us/training-events/exams/capi.pdf>

---

## Course content

Start with the **[10-day course outline](00-Course-Outline.md)**, then work through one day at a time. Every lesson reads directly on GitHub — click a topic to open it. A PDF copy of each day is there if you prefer to print or study offline.

| Day | Topic (read online) | Blueprint | Offline |
|---|---|---|---|
| 1 | **[APM concepts and AppDynamics architecture](days/day-01-apm-concepts-and-architecture.md)** | Foundation | [PDF](pdf/Day01_APM_Concepts_and_Architecture.pdf) |
| 2 | **[Pre-deployment planning and sizing](days/day-02-pre-deployment-planning-and-sizing.md)** | 1.1–1.6 | [PDF](pdf/Day02_Pre-Deployment_Planning_and_Sizing.pdf) |
| 3 | **[Controller installation and the Enterprise Console](days/day-03-controller-install-and-enterprise-console.md)** | 2.1, 4.3 | [PDF](pdf/Day03_Controller_Install_and_Enterprise_Console.pdf) |
| 4 | **[SSL, high availability, self-monitoring, upgrades](days/day-04-ssl-ha-self-monitoring-upgrades.md)** | 2.2, 2.3, 2.4, 2.6 | [PDF](pdf/Day04_SSL_HA_Self-Monitoring_Upgrades.pdf) |
| 5 | **[Events Service and EUM Server](days/day-05-events-service-and-eum-server.md)** | 1.4, 2.5, 2.7, 2.8, 3.7 | [PDF](pdf/Day05_Events_Service_and_EUM_Server.pdf) |
| 6 | **[Java and .NET agents, custom correlation](days/day-06-java-and-dotnet-agents-custom-correlation.md)** | 1.6, 3.1, 3.2, 3.9, 3.10 | [PDF](pdf/Day06_Java_and_NET_Agents_Custom_Correlation.pdf) |
| 7 | **[Machine, Database, Browser and Mobile agents](days/day-07-machine-database-browser-mobile-agents.md)** | 3.3–3.6, 3.8, 3.11 | [PDF](pdf/Day07_Machine_Database_Browser_Mobile_Agents.pdf) |
| 8 | **[Post-deployment verification, RBAC and revision](days/day-08-verification-rbac-and-revision.md)** | 4.1–4.3 | [PDF](pdf/Day08_Verification_RBAC_and_Revision.pdf) |
| 9 | **[APIs and extensions (+ Postman)](days/day-09-apis-and-extensions.md)** | 5.1–5.6 | [PDF](pdf/Day09_APIs_and_Extensions.pdf) |
| 10 | Mock exam and revision | All | Coming soon |

Every day follows the same layout: **lesson blocks → lab → quiz (click *Show answer* to check yourself) → cheat sheet → sources → lab model answers**.

---

## Postman collection (Day 9)

The [`postman`](postman/) folder contains every Day 9 API request ready to import:

1. In Postman choose **Import** and select both files in `postman/`.
2. Select the **AppDynamics Lab** environment and fill in `baseUrl`, `account`, `clientName` and `clientSecret`.
3. Run **1 Authentication → Get OAuth token** first; it saves the token automatically (tokens last 5 minutes by default).

The [Day 9 page](days/day-09-apis-and-extensions.md#postman-equivalents-of-every-curl-request) explains each request field by field.

---

## Suggested lab setup

- An **AppDynamics SaaS trial** for the agent, RBAC and API labs (Days 1, 6–9).
- One **Linux VM** with Java 17 + Tomcat (Spring PetClinic) and one **Windows VM** with IIS.
- On-prem installation topics (Days 2–5) are taught as **paper labs** with exact commands, config files and model answers, because on-prem installers need a licence. If you have an on-prem licence (partner NFR or trial from sales), repeat Days 3–5 on three Linux VMs.

---

## How the content was checked

Facts were checked against the official **Splunk AppDynamics documentation** (<https://help.splunk.com/en/appdynamics-on-premises>) in October–November 2026, and each day lists its sources. Where a fact could not be confirmed in the current docs, the notes say so (for example, the default slow/very slow/stall transaction thresholds). Product versions change — always confirm against the docs for your version.

---

## Disclaimer

- This is an independent study resource. It is **not affiliated with, endorsed by or produced by Cisco, Splunk or AppDynamics**. Product names are trademarks of their owners.
- The practice questions are **original** and written for learning. They are **not** real exam questions. Please don't use or share exam dumps — they break the Cisco candidate agreement and are often wrong.

---

## Contributing

Found an error or an outdated fact? Open an **issue** or a **pull request** with a link to the official doc page. Corrections that keep the notes accurate are very welcome.

## Repository structure

```
.
├── 00-Course-Outline.md       ← start here
├── README.md                  ← you are here
├── days/
│   ├── day-01-apm-concepts-and-architecture.md
│   ├── ...
│   └── day-09-apis-and-extensions.md
├── images/                    ← diagrams used by the pages
├── pdf/                       ← printable copies of every page
└── postman/
    ├── AppDynamics_Day9_Postman_Collection.json
    └── AppDynamics_Lab_Postman_Environment.json
```
