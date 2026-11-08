# Day 8 — Post-Deployment Verification, RBAC and Revision

[🏠 Home](../README.md) · [← Day 7](day-07-machine-database-browser-mobile-agents.md) · [Day 9 →](day-09-apis-and-extensions.md) · [📄 PDF version](../pdf/Day08_Verification_RBAC_and_Revision.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 4.1, 4.2, 4.3 (Domain 4 = 5%) + revision of Domains 1–2

**Goal:** control access with roles and permissions, prove a deployment is healthy, use the Administration Console, and revise the most-tested platform facts.

**Verification:** every fact checked against the current Splunk AppDynamics on-prem docs (November 2026).

## Block 1 — Role-based access control (4.1)

```
Users ──┐
        ├──▶ Roles ──▶ Permissions (account-level + application-level)
Groups ─┘
```

Roles can be given to users or groups. Groups are easier to manage and are how LDAP/SAML mapping works.

|**Predefined role**|**What it can do**|
|---|---|
|Account Owner|Manages **users, groups, roles and the authentication provider**+ most account permissions — the **only** role that can do user/role administration|
|Administrator|Day-to-day objects (applications, dashboards, war rooms, scheduled reports).**Can't** manage users, groups or roles|
|Analytics Administrator|Analytics access: API keys, extracted fields, saved searches, which roles see which data|
|Applications and Dashboards Viewer|Views all applications and dashboards, can't edit (**former "Read-Only User"**)|
|Dashboards Viewer|Views custom dashboards only|
|DB Monitoring User|Views Database Monitoring; can't change collectors|
|DB Monitoring Administrator|Views Database Monitoring; adds, edits, deletes collectors|
|Server Monitoring User|Views Server Monitoring; can't configure it|
|Server Monitoring Administrator|Views and configures Server Monitoring, incl. Service Availability Monitoring|

- **Predefined roles can't be edited** — create a **custom role** instead.

- **Application permissions** apply at **Default → Application → Tier**; the lower level wins. Defaults include **Can Create Applications** and View / Edit / Delete.

- Specific permissions: Configure Health Rules, Configure Business Transactions, Configure Agent Properties, **Agent Advanced Operations** (reset agent, thread dumps, debug logs), Configure Monitoring Level.

- **Several roles:** if one role grants a permission and another doesn't, the **grant wins**.

- **LDAP:** any LDAPv3 directory (verified: Active Directory, OpenLDAP 2.4+); **map LDAP groups to roles**; if LDAP is down or a user isn't found, the Controller **falls back to local login** — keep a local admin account.

- **SAML** single sign-on is the other external option.

- The Administration Console **root** user is separate from account users.

## Block 2 — Platform health checks (4.2)

|**Component**|**Check**|**Healthy result**|
|---|---|---|
|Controller|http://&lt;ctrl&gt;:8090/controller/rest/serverstatus bin/platform-admin.sh check-controller-health Log in as the initial admin|&lt;available&gt;true&lt;/available&gt; (200; HA standby 503) No issues UI loads, applications visible|
|Enterprise Console|http://&lt;ec&gt;:9191 → Platform page|All services running|
|Events Service|bin/platform-admin.sh show-events-service-health https://&lt;es&gt;:9080/_ping EC → Events Service page|No issues HTTP 200 All hosts green|
|EUM Server|http://&lt;eum&gt;:7001/eumaggregator/ping (or /eumcollector/ping)|ping|
|App agents|Tiers & Nodes → App Agents|Nodes reporting recent metrics|
|Machine / DB agents|Servers list; Databases → collector status|Reporting, no collector errors|
|Controller itself|system account (?enableAccounts=true); disk latency alerts|Normal memory trend; write latency ≤ 3 ms|
|Licenses|License usage page|Enough units for the agents deployed|

|**Component**|**Log**|
|---|---|
|Controller|&lt;controller_home&gt;/logs/server.log · db/logs/database.log|
|Enterprise Console|platform-admin-server.log|
|Events Service|Collect with retrieve-events-service-logs|
|Java Agent|&lt;agent_home&gt;/ver&lt;version&gt;/logs/&lt;node&gt;/agent.log|
|.NET Agent|%ProgramData%\AppDynamics\DotNetAgent\Logs|

## Block 3 — The Administration Console (4.3)

- URL: http(s)://&lt;ctrl&gt;:&lt;port&gt;/controller/admin.jsp — log in as **root**. Another user logged in → **Access Denied**: log out or use a private window.

- **Controller Settings:** eum.cloud.host · eum.beacon.host · eum.beacon.https.host · eum.mobile.screenshot.host · appdynamics.on.premise.event.service.url/.key · metrics.min.retention.period (4 h) · metrics.ten.min.retention.period (48 h) · metrics.retention.period (365 days).

- **Account Settings:** accounts, licenses, access keys; multi-tenant → each account's EUM License Key and EUM Account Name.

|**Task**|**How**|
|---|---|
|Change root user password|Administration Console → user profile → My Settings → My Preferences → Change Password|
|Reset lost root password|bin/controller.sh login-db → update user set encrypted_password = sha1('&lt;new&gt;') where name='root'; → restart app server (hash upgrades to PBKDF2 at next login)|
|Change MySQL root password|**EC → Configurations → Controller Settings → Database Configurations** (immediate), or manually start-db insecure → ALTER USER|

## Block 4 — Revision: most-tested platform facts (Days 2–5)

|**#**|**Fact**|**Day**|
|---|---|---|
|1|Primary sizing driver = metric ingestion rate; second = number of agents|2|
|2|Demo 10K/5 · Small 50K/50 · Medium 1M/2,000 · Large 5M/15,000 (Linux only)|2|
|3|Disk write latency ≤ 3 ms; no NFS|2|
|4|Tenancy: single → multi OK; multi → single never|2|
|5|Events Service: 1 node or ≥ 3; heap ½ RAM, 7–31 GB; SSD|2|
|6|EUM Demo = same host as Controller; Production = own host + own MySQL|2, 5|
|7|EC installer platform-setup-64bit-linux.sh -q -varfile; EC 9191, EC DB 3377, Controller DB 3388|3|
|8|Start database first, then app server; stop in reverse|3|
|9|Retention defaults 4 h / 48 h / 365 days|3|
|10|SSL: keystore.jks, alias s1as, changeit, 8181; chain error → cacerts.jks|4|
|11|HA: active-passive, identical Linux, activate-ha-modules; &gt; 3 days incremental replication; &gt; 7 days revive|4|
|12|Disable auto-failover before restarting the primary (Watchdog fails over after 5 min)|3, 4|
|13|Upgrade order EC → Events Service → EUM → Controller; EC doesn't back up MySQL|4|
|14|Controller→ES key ad.accountmanager.key.controller; EUM→ES key appdynamics.es.eum.key|5|
|15|ES upgrade failed → useCheckpoint=true; EUM upgrade → keep EUM MySQL running|5|

## Lab (~45 min)

1. **Custom role (SaaS):** "PetClinic Viewer" with View on PetClinic only, assigned to group "QA"; add a test user; confirm only PetClinic is visible and read-only.

2. **Combined roles:** give the test user a second role with Configure Health Rules on PetClinic. Can they now edit health rules? Why?

3. **Health rule + policy:** average response time &gt; 2 s on PetClinic, with a policy that sends an email.

4. **Paper:** a 10-step post-go-live verification checklist for the bank platform (HA Controller, 3-node Events Service, EUM Server, agents).

## Day 8 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Only predefined role that can add/edit users, groups, roles and the authentication provider?**

- A. Administrator
- B. Account Owner
- C. Analytics Administrator
- D. Server Monitoring Administrator

<details><summary>Show answer</summary>

**Answer: B** — Administrator handles day-to-day objects, not user administration.

</details>

**Q2. Team needs Administrator rights minus deleting applications — what do you do?**

- A. Edit the Administrator role
- B. Create a custom role
- C. Give Account Owner
- D. Change admin.jsp

<details><summary>Show answer</summary>

**Answer: B** — Predefined roles can't be edited.

</details>

**Q3. App-level View on PetClinic, tier-level Edit on Payments — which applies to Payments?**

- A. Application setting
- B. Tier setting (lower overrides higher)
- C. Default
- D. Neither

<details><summary>Show answer</summary>

**Answer: B** — Default → application → tier; lower wins.

</details>

**Q4. LDAP server is down — what happens to logins?**

- A. Nobody logs in
- B. Fallback to local authentication; keep a local admin
- C. SAML takes over
- D. Only root via UI

<details><summary>Show answer</summary>

**Answer: B** — The docs recommend keeping a local administrator account.

</details>

**Q5. Which HTTP check confirms the Events Service is healthy?**

- A. https://&lt;es&gt;:9080/_ping → 200
- B. /eumaggregator/ping
- C. /controller/rest/serverstatus
- D. :9191

<details><summary>Show answer</summary>

**Answer: A** — Also show-events-service-health or the EC page.

</details>

**Q6. Change the Controller MySQL root password in the EC?**

- A. admin.jsp → Account Settings
- B. EC → Configurations → Controller Settings → Database Configurations
- C. EC → Events Service page
- D. Reinstall only

<details><summary>Show answer</summary>

**Answer: B** — Takes effect immediately.

</details>

**Q7. Two measurements to estimate Controller requirements?**

- A. Users + dashboards
- B. Metrics/min + agents reporting
- C. Health rules + tiers
- D. Disk + applications

<details><summary>Show answer</summary>

**Answer: B** — Prep-guide sample answer.

</details>

**Q8. Platform upgrade order?**

- A. Controller → ES → EUM → EC
- B. EC → Events Service → EUM Server → Controller
- C. EUM → Controller → EC → ES
- D. EC → Controller → EUM → ES

<details><summary>Show answer</summary>

**Answer: B** — Controller last.

</details>

**Q9. Secondary 4 days behind — which job, no downtime?**

- A. ha-failover
- B. incremental-replication
- C. remove + add-secondary
- D. activate-ha-modules

<details><summary>Show answer</summary>

**Answer: B** — &gt; 7 days needs a revive (primary restarts).

</details>

**Q10. Which value goes into EUM analytics.accountAccessKey?**

- A. ad.accountmanager.key.controller
- B. appdynamics.es.eum.key
- C. Root password
- D. EUM license key

<details><summary>Show answer</summary>

**Answer: B** — Controller → ES uses the other key.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|RBAC model|Users / groups → roles → account + application permissions|
|Account Owner|Only role for users, groups, roles, authentication provider|
|Read-Only User|Now called Applications and Dashboards Viewer|
|Predefined roles|Can't be edited → custom roles|
|Permission levels|Default → Application → Tier (lower wins)|
|Multiple roles|A grant in any role wins|
|LDAP|Group → role mapping · local fallback · keep a local admin|
|Health URLs|Controller /controller/rest/serverstatus · ES :9080/_ping · EUM /eumaggregator/ping|
|Health CLI|check-controller-health · show-events-service-health|
|Admin Console|/controller/admin.jsp · root · Controller Settings / Account Settings|
|Root password|My Preferences → Change Password · reset via login-db SQL|
|MySQL root password|EC → Controller Settings → Database Configurations|

## Sources

- [Predefined Roles (on-prem)](https://help.splunk.com/en/appdynamics-on-premises/accounts/splunk-appdynamics-on-premises-user-management/roles-and-permissions/predefined-roles)

- [Application Permissions for On-Premises](https://help-preview.splunk.com/en/appdynamics-on-premises/accounts/splunk-appdynamics-on-premises-user-management/roles-and-permissions/application-permissions-for-on-premises)

- [LDAP Support](https://help.splunk.com/en/appdynamics-on-premises/accounts/splunk-appdynamics-on-premises-user-management/manage-controller-users-and-groups/external-authentication-for-on-premises-deployments/ldap-for-on-premises-deployments/ldap-support)

- [Update the Root User Passwords](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.8.0/controller-deployment/administer-the-controller/update-the-root-user-passwords)

- [Start or Stop the Controller (health check CLI)](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.9.0/controller-deployment/administer-the-controller/start-or-stop-the-controller)

- [Administer the Events Service](https://help.splunk.com/en/appdynamics-on-premises/events-service-deployment/events-service-deployment/administer-the-events-service)

- [Start and Stop the EUM Server and Database](https://help.splunk.com/en/appdynamics-on-premises/eum-server-deployment/eum-server-deployment/start-and-stop-the-eum-server-and-database)

- [Access the Administration Console](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.2.0/controller-deployment/administer-the-controller/access-the-administration-console)

## Lab — model answers

What a correct result looks like for each Day 8 lab task, with the rule behind it.

### 1. Custom role "PetClinic Viewer"

1. Settings (gear) → Administration → **Roles** → + → name "PetClinic Viewer".

2. Applications tab: leave default permissions with no View/Edit; for **PetClinic** set Custom → **View** only (Edit and Delete off). Save.

3. **Groups** → + "QA" → assign the PetClinic Viewer role.

4. **Users** → + test user → add to group QA (with LDAP you would map the directory group instead).

5. Log in as the test user: only PetClinic is listed; configuration pages are read-only. Only an **Account Owner** (or a role with user administration rights) can do steps 1–4.

### 2. Combined roles

**Yes** — the test user can now create and edit PetClinic health rules. When a user holds several roles, a permission granted in any one role takes precedence over its absence in another. They still can't edit other PetClinic configuration that neither role grants.

### 3. Health rule and email policy

1. Alert & Respond → **Health Rules** → PetClinic → + → type Business Transaction performance, affects all BTs.

2. Critical condition: **Average Response Time &gt; 2000 ms** (static value). Optionally a warning condition, e.g. &gt; 1500 ms. Save.

3. Alert & Respond → **Actions** → + → Send an Email → recipient address.

4. Alert & Respond → **Policies** → + → trigger "Health Rule Violation Started — Critical" for that rule → attach the email action. Save.

5. Generate slow traffic to test; the violation appears and the email is sent (on-prem, the Controller's SMTP settings must be configured).

Chain to remember: Metric → Health rule violation → Policy → Action.

### 4. 10-step post-go-live checklist — bank platform

|**#**|**Check**|**Pass criteria**|
|---|---|---|
|1|Controller via VIP: /controller/rest/serverstatus|200 + available=true on the active node; standby returns 503|
|2|check-controller-health; log in as admin|No issues; applications visible|
|3|HA pair|Replication healthy; auto-failover (Watchdog) enabled; planned failover tested in a maintenance window|
|4|Enterprise Console Platform page|All services running|
|5|Events Service|show-events-service-health clean; :9080/_ping → 200 on each node and the VIP; Analytics searches return data|
|6|EUM Server|/eumaggregator/ping → ping; licenses provisioned; browser beacons and mobile data arriving|
|7|Agents|All expected nodes reporting; agent.log clean; Machine Agents with Server Visibility; DB collectors without errors|
|8|Licenses|Usage within purchased units|
|9|Security|CA-signed certificate on s1as, changeit replaced, root/MySQL passwords changed, RBAC roles + LDAP group mapping, local admin kept|
|10|Operations|Self-monitoring to a separate monitoring Controller; disk latency alert at 3 ms; backups scheduled (levels 1–3); NTP in sync; retention reviewed|

---

[← Day 7](day-07-machine-database-browser-mobile-agents.md) · [🏠 Home](../README.md) · [Day 9 →](day-09-apis-and-extensions.md)
