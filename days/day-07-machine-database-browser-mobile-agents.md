# Day 7 — Machine, Database, Browser and Mobile Agents

[🏠 Home](../README.md) · [← Day 6](day-06-java-and-dotnet-agents-custom-correlation.md) · [Day 8 →](day-08-verification-rbac-and-revision.md) · [📄 PDF version](../pdf/Day07_Machine_Database_Browser_Mobile_Agents.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 3.3, 3.4, 3.5, 3.6, 3.8, 3.11

**Goal:** install and configure the Machine Agent, Database Agent, JavaScript (Browser RUM) agent and Mobile SDK, and upgrade the Database Agent. In the blueprint, "EUEM Agent" (3.8) is the old name for the JavaScript agent.

**Verification:** every fact checked against the current Splunk AppDynamics docs (November 2026).

## Block 1 — Standalone Machine Agent (3.3)

|**Method**|**Details**|
|---|---|
|ZIP bundle (Java included)|Unzip, configure, start with java -jar &lt;MA_HOME&gt;/machineagent.jar (or the start script)|
|RPM / DEB (Linux)|sudo rpm -ivh appdynamics-machine-agent-&lt;ver&gt;.x86_64.rpm → /opt/appdynamics/machine-agent, service **appdynamics-machine-agent** (systemctl start\|stop appdynamics-machine-agent; SysV: service ... start)|

- **Config:** &lt;machine_agent_home&gt;/conf/controller-info.xml (host, port, SSL, account, key). The RPM links it at /etc/appdynamics/machine-agent/controller-info.xml.

- **Server Visibility:** set **sim-enabled=true** (&lt;sim-enabled&gt;, -Dappdynamics.sim.enabled or APPDYNAMICS_SIM_ENABLED); needs a **Server Visibility license** (add-on); restart the agent.

- **HTTP listener:** off by default; -Dmetric.http.listener=true; default port **8293** (metric.http.listener.port); localhost only unless metric.http.listener.host=0.0.0.0; POST JSON to /api/v1/metrics at least every 5 minutes.

- **Extensions** go in the monitors/ folder (Day 9).

```
java -Dmetric.http.listener=true -jar machineagent.jar
curl -X POST http://localhost:8293/api/v1/metrics -H "Content-Type: application/json" \
  -d '[{"metricName":"Custom Metrics|Demo|QueueDepth","aggregatorType":"SUM","value":5}]'
```

## Block 2 — Database Agent (3.4)

- Standalone Java process (db-agent.jar) that monitors databases **remotely** — not installed on the database server.

- Configure &lt;db_agent_home&gt;/conf/controller-info.xml; start with the script (recommended):

```
cd <db_agent_home>
./start-dbagent -Xms<min> -Xmx<max> &          # Linux
start-dbagent.bat -Xms<min> -Xmx<max>          # Windows
```

- Stop: kill &lt;PID&gt; (background), Ctrl+C (console), or stop the **AppDynamics Database Agent** Windows service.

- Several Database Agents on one Controller → **-Ddbagent.name="..."** each; without it the name is **"Default Database Agent"**.

- Java 15+: add --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.security=ALL-UNNAMED. Windows raw java: -Djava.library.path=&lt;db_agent_home&gt;\auth\x64.

- **Collectors** are configured in the Controller UI (Databases → Configuration): type, host, port, monitoring user with the required permissions.

### • On-prem, Database Monitoring data is stored in the **Events Service** .

**Hardware (OS) monitoring** — tick Monitor Operating System:

|**DB host OS**|**Connection**|
|---|---|
|Linux / AIX / Solaris|**SSH** (port 22, password or PEM key)|
|Windows|**WMI** — user must open a WMI connection and read Performance Counters. "Use Local WMI" = the agent's own machine|

**Exam trap:** a Database Agent on **Linux can't monitor Windows hardware**. SQL Server hardware metrics can still come through SQL query collection.

## Block 3 — Upgrade the Database Agent (3.11)

1. **Upgrade the Controller first**, then the Database Agents.

2. Check the monitoring user still has the required permissions on each collector's database.

3. **Stop** the agent (agents 4.2+ can keep running while you install the new one).

4. **Back up** &lt;db_agent_home&gt; for rollback.

5. Install the new agent; **copy conf/controller-info.xml** from the backup.

6. **Start** the agent.

## Block 4 — Browser RUM: the JavaScript ("EUEM") agent (3.5, 3.8)

1. User Experience → Browser Apps → **+Add** (unique name).

2. Open the app → Configuration → Instrumentation → **Browser Monitoring on**.

3. Each browser app has an **EUM App Key** that identifies its data.

4. Inject the JavaScript agent into the **page header**.

|**Method**|**How**|**Supported by**|
|---|---|---|
|Manual|Paste the snippet into each page's &lt;head&gt;|**All** platforms and frameworks|
|Automatic|Server-side app agent injects at runtime|Java (Jasper JSP), .NET (ASP.NET/ASPX, ASP.NET Core on Windows)|
|Assisted — injection rules|Rules name the Java classes/methods that write the page|**Java only**|
|Assisted — attribute injection|JS_HEADER / JS_FOOTER snippets in templates, filled by the agent|App-agent based|
|Container-based|Nginx / Apache directives inject into the response|Web server|

```
<script charset="UTF-8">
window['adrum-config'] = {
  appKey: '<EUM_APP_KEY>',
```

```
  adrumExtUrlHttp:  'http://cdn.appdynamics.com',   // adrum-ext.js (CDN or self-hosted)
  adrumExtUrlHttps: 'https://cdn.appdynamics.com',
  beaconUrlHttp:  'http://eum01.bank.local:7001',  // ON-PREM: your EUM Server
  beaconUrlHttps: 'https://eum.bank.com',          // ON-PREM: reverse proxy to EUM
  useHTTPSAlways: true                              // optional
};
</script>
<script src="//cdn.appdynamics.com/adrum/adrum-latest.js"></script>
```

- **On-prem:** beaconUrlHttp/beaconUrlHttps must point to your EUM Server under a hostname **users' browsers** can reach. A wrong beacon URL is a common cause of no browser data.

- By default the agent uses the same protocol as the page; useHTTPSAlways forces HTTPS.

## Block 5 — Mobile agent SDK (3.6)

- Create a Mobile App in the Controller to get its **EUM App Key**.

- **Android:** AppDynamics Gradle plugin + library; start the agent in the main Activity's onCreate():

```
import com.appdynamics.eumagent.runtime.Instrumentation;
```

```
Instrumentation.start(AgentConfiguration.builder()
    .withAppKey("<EUM_APP_KEY>")
    .withContext(getApplicationContext())
    .withCollectorURL("https://eum.bank.com")     // ON-PREM: your EUM Server
    .withScreenshotURL("https://eum.bank.com/")   // ON-PREM: your EUM Server
    .build());
```

- **iOS:** add the SDK (CocoaPods / Swift Package Manager); initialise ADEumInstrumentation with a configuration holding the app key plus collectorURL and screenshotURL for on-prem.

- SaaS default Americas region can omit the collector/screenshot URLs — **on-prem must always set them**.

- Controller side: screenshots are located through eum.mobile.screenshot.host (Day 5).

## Lab — SaaS trial (~60 min)

1. **Machine Agent:** install the ZIP bundle on the PetClinic VM, sim-enabled=true, start it, confirm the server is linked to the web01 node.

2. **Custom metric:** restart with -Dmetric.http.listener=true; post Custom Metrics|Lab|Orders = 10 every minute for 5 minutes; find it in the Metric Browser.

3. **Database Agent:** install MySQL or PostgreSQL, start the Database Agent named "Lab DB Agent", add a collector, find the top 10 queries.

4. **Browser RUM:** create a browser app, paste the snippet (manual injection) into PetClinic's header, browse, compare page load vs server response times.

5. **Paper (on-prem):** browsers reach the EUM Server via https://eum.bank.com (reverse proxy → eum01:7001). Write the adrum-config beacon settings and the Android collector/screenshot URLs.

## Day 7 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. What is needed to enable Server Visibility?**

- A. Install the Database Agent
- B. sim-enabled=true + Server Visibility license + restart
- C. admin.jsp only
- D. Always on

<details><summary>Show answer</summary>

**Answer: B** — Server Visibility is an add-on module.

</details>

**Q2. Default port of the Machine Agent HTTP listener?**

- A. 8090
- B. 8293
- C. 9080
- D. 7001

<details><summary>Show answer</summary>

**Answer: B** — Off by default; localhost only unless host=0.0.0.0; POST /api/v1/metrics.

</details>

**Q3. Machine Agent installed from RPM — start it on systemd?**

- A. systemctl start appdynamics-machine-agent
- B. platform-admin.sh start-machine-agent
- C. service machineagent start
- D. ./start-dbagent

<details><summary>Show answer</summary>

**Answer: A** — RPM service name: appdynamics-machine-agent.

</details>

**Q4. Where should the Database Agent be installed?**

- A. Every database server
- B. Any host that can reach the databases (remote collectors)
- C. Inside the Controller
- D. On the EUM Server

<details><summary>Show answer</summary>

**Answer: B** — It monitors remotely; collectors are set in the Controller UI.

</details>

**Q5. Two Database Agents, neither sets -Ddbagent.name — result?**

- A. Both 'Default Database Agent' — name each one
- B. Controller names them
- C. Second fails
- D. They merge

<details><summary>Show answer</summary>

**Answer: A** — A name is required when several agents report to one Controller.

</details>

**Q6. Linux Database Agent — can it collect Windows SQL Server host hardware via WMI?**

- A. Yes, domain user
- B. No (SQL Server metrics still via SQL queries)
- C. Yes, via SSH
- D. Only with Server Visibility

<details><summary>Show answer</summary>

**Answer: B** — WMI needs a Windows Database Agent.

</details>

**Q7. Upgrading Controller and Database Agents — order?**

- A. Agents first
- B. Controller first, then Database Agents
- C. Same time
- D. Doesn't matter

<details><summary>Show answer</summary>

**Answer: B** — Back up, install new, copy controller-info.xml, start.

</details>

**Q8. Which injection method works on every platform?**

- A. Automatic
- B. Assisted (rules)
- C. Manual
- D. Container-based

<details><summary>Show answer</summary>

**Answer: C** — Automatic needs Java/.NET app agents; assisted rules are Java only.

</details>

**Q9. On-prem Browser RUM shows no data; beaconUrlHttps points to the SaaS cloud. Fix?**

- A. Change appKey
- B. Point beacon URLs to the on-prem EUM Server (browser-reachable)
- C. Enable Server Visibility
- D. Restart the Controller

<details><summary>Show answer</summary>

**Answer: B** — Beacons must reach your EUM Server.

</details>

**Q10. When must Android set withCollectorURL / withScreenshotURL?**

- A. Never
- B. Always with on-prem EUM (only SaaS Americas default can omit)
- C. Only if screenshots off
- D. Only iOS

<details><summary>Show answer</summary>

**Answer: B** — On-prem agents must send to your EUM Server.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|Machine Agent RPM|/opt/appdynamics/machine-agent · service appdynamics-machine-agent|
|Server Visibility|sim-enabled=true · add-on license · restart|
|HTTP listener|-Dmetric.http.listener=true · port 8293 · localhost by default · /api/v1/metrics every ≤ 5 min|
|Database Agent|Remote · start-dbagent -Xms -Xmx · -Ddbagent.name (else 'Default Database Agent')|
|DB hardware|SSH for Linux/AIX/Solaris · WMI for Windows · Linux agent can't do Windows hardware|
|DB data on-prem|Stored in the Events Service|
|DB Agent upgrade|Controller first · back up · install new · copy controller-info.xml · start|
|Browser RUM|Browser App + EUM App Key · enable in Instrumentation · inject in &lt;head&gt;|
|Injection|Manual = all · Automatic = Java JSP / .NET ASP.NET · Assisted rules = Java only|
|adrum-config|appKey · adrumExtUrlHttp/Https · beaconUrlHttp/Https (on-prem → EUM Server) · useHTTPSAlways|
|Mobile|EUM App Key · Android Instrumentation.start(...withCollectorURL, withScreenshotURL) · on-prem must set both|

## Sources

- [Linux Install Using the RPM Package (Machine Agent)](https://help.splunk.com/en/appdynamics-on-premises/infrastructure-visibility/25.7.0/machine-agent/install-the-machine-agent/linux-install-using-the-rpm-package)

- [Enable Server Visibility](https://help.splunk.com/en/appdynamics-saas/infrastructure-visibility/26.4.0/server-visibility/enable-server-visibility)

- [Machine Agent HTTP Listener](https://help.splunk.com/en/appdynamics-saas/infrastructure-visibility/26.5.0/machine-agent/extensions-and-custom-metrics/machine-agent-http-listener)

- [Launch the Database Agent using System Properties](https://help.splunk.com/en/appdynamics-on-premises/database-visibility/26.3.0/administer-the-database-agent/install-the-database-agent/launch-the-database-agent-using-system-properties)

- [Start and Stop the Database Agent](https://help.splunk.com/en/appdynamics-on-premises/database-visibility/26.7.0/administer-the-database-agent/start-and-stop-the-database-agent)

- [Complete the Hardware Monitoring Section](https://help.splunk.com/en/appdynamics-on-premises/database-visibility/26.3.0/add-database-collectors/configure-the-database-agent-to-monitor-server-hardware/complete-the-hardware-monitoring-section)

- [Upgrade the Database Agent](https://help.splunk.com/en/appdynamics-on-premises/database-visibility/26.3.0/administer-the-database-agent/upgrade-the-database-agent)

- [Set Up and Access Browser RUM](https://help.splunk.com/en/appdynamics-saas/end-user-monitoring/25.7.0/end-user-monitoring/browser-monitoring/browser-real-user-monitoring/set-up-and-access-browser-rum)

- [Overview of Injection Types](https://help.splunk.com/appdynamics-saas/end-user-monitoring/25.4.0/end-user-monitoring/browser-monitoring/browser-real-user-monitoring/inject-the-javascript-agent/overview-of-injection-types)

- [Configure the JavaScript Agent to Use HTTPS](https://help.splunk.com/en/appdynamics-on-premises/end-user-monitoring/25.7.0/end-user-monitoring/browser-monitoring/browser-real-user-monitoring/configure-the-javascript-agent/configure-the-javascript-agent-to-use-https)

- [Android: Modify the Source](https://help.splunk.com/appdynamics-on-premises/end-user-monitoring/25.6.0/end-user-monitoring/mobile-real-user-monitoring/instrument-android-applications/instrument-the-android-application/modify-the-source)

## Lab — model answers

What a correct result looks like for each Day 7 lab task, with the rule behind it.

### 1. Machine Agent with Server Visibility

```
unzip machineagent-bundle-64bit-linux-<ver>.zip -d /opt/appdynamics/machine-agent
# conf/controller-info.xml: controller-host, controller-port 443,
#   controller-ssl-enabled true, account-name, account-access-key, and:
<sim-enabled>true</sim-enabled>
```

```
nohup java -jar /opt/appdynamics/machine-agent/machineagent.jar &
```

- The host appears in the Controller's server list with full Server Visibility metrics (if the trial includes the Server Visibility license).

- Because the Machine Agent runs on the same host as the web01 Java Agent, the Controller associates the server with that node.

- No data? Check the Machine Agent's controller-info.xml (same host/port/SSL/key as the Java Agent) and that the license includes Server Visibility.

### 2. Custom metric through the HTTP listener

```
java -Dmetric.http.listener=true -jar /opt/appdynamics/machine-agent/machineagent.jar &
for i in 1 2 3 4 5; do
```

```
  curl -s -X POST http://localhost:8293/api/v1/metrics \
```

```
    -H "Content-Type: application/json" \
```

```
    -d '[{"metricName":"Custom Metrics|Lab|Orders","aggregatorType":"SUM","value":10}]'
  sleep 60
done
```

- In the Metric Browser, the metric appears under **Custom Metrics → Lab → Orders** with value 10 each minute.

- The listener only accepts localhost by default; posting from another host needs -Dmetric.http.listener.host=0.0.0.0.

- If posting stops for more than 5 minutes, the metric stops reporting.

### 3. Database Agent and collector

```
cd /opt/appdynamics/db-agent       # conf/controller-info.xml already set
java -Ddbagent.name="Lab DB Agent" -jar db-agent.jar &
#   (Java 15+: add --add-opens java.base/java.lang=ALL-UNNAMED
#              --add-opens java.base/java.security=ALL-UNNAMED)
```

- Controller → Databases → Configuration → Add collector: type MySQL (or PostgreSQL), host, port 3306 (5432), a monitoring user with the required permissions, agent **Lab DB Agent**.

- Open the database dashboard → **Queries** to see the top queries by time — PetClinic's owner/pet lookups appear after you browse the site.

- Optional: Monitor Operating System over SSH for host CPU/memory.

### 4. Browser RUM with manual injection

1. User Experience → Browser Apps → +Add → name "PetClinic-Web"; Configuration → Instrumentation → Browser Monitoring on.

2. Copy the generated snippet (it already contains your appKey and the SaaS beacon URLs).

3. Paste it into the &lt;head&gt; of PetClinic's shared page template (in Spring PetClinic: templates/fragments/layout.html), rebuild/restart.

4. Browse several pages. The browser app shows page load times; compare with the BT response times — the difference is network time plus browser rendering, which server-side APM can't see.

### 5. On-prem beacon and mobile settings for https://eum.bank.com

```
window['adrum-config'] = {
  appKey: '<EUM_APP_KEY>',
  beaconUrlHttp:  'http://eum.bank.com',    // only if the proxy also serves HTTP
  beaconUrlHttps: 'https://eum.bank.com',
  useHTTPSAlways: true                       // bank pages are HTTPS
};
.withCollectorURL("https://eum.bank.com")
.withScreenshotURL("https://eum.bank.com/")
```

- Browsers and phones on the internet use the **public** name eum.bank.com; the reverse proxy (CA-signed certificate) forwards to eum01:7001.

- Controller (admin.jsp): eum.beacon.https.host = https://eum.bank.com (what browsers use), eum.cloud.host = http://eum01.bank.local:7001 (internal polling), eum.mobile.screenshot.host to the same public EUM name.

- adrumExtUrlHttp/Https can stay on the AppDynamics CDN, or point to a self-hosted copy of adrum-ext.js if browsers can't reach the internet CDN.

---

[← Day 6](day-06-java-and-dotnet-agents-custom-correlation.md) · [🏠 Home](../README.md) · [Day 8 →](day-08-verification-rbac-and-revision.md)
