# Day 6 — Java and .NET Agents, Custom Correlation

[🏠 Home](../README.md) · [← Day 5](day-05-events-service-and-eum-server.md) · [Day 7 →](day-07-machine-database-browser-mobile-agents.md) · [📄 PDF version](../pdf/Day06_Java_and_NET_Agents_Custom_Correlation.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 1.6, 3.1, 3.2, 3.9, 3.10

**Goal:** install, configure, verify and upgrade the Java and .NET agents, and know when and how to use custom correlation.

**Verification:** every fact checked against the current Splunk AppDynamics docs (November 2026).

## Configuration precedence (highest first)

1. **Environment variables**

2. **System properties** (-D... on the JVM command line)

3. Versioned agent.properties — &lt;agent_home&gt;/&lt;ver&gt;/conf/agent.properties

4. Global agent.properties — &lt;agent_home&gt;/conf/agent.properties

5. Versioned controller-info.xml — &lt;agent_home&gt;/&lt;ver&gt;/conf/controller-info.xml

6. Global controller-info.xml — &lt;agent_home&gt;/conf/controller-info.xml

The agent applies the **first non-empty value** it finds. Environment variables beat system properties, which beat the files.

## Block 1 — Naming rules

- Application → Tier → Node. The agent tells the Controller which ones it belongs to.

- A node name must be **unique within the business application and the physical host**.

- Containers / short-lived JVMs: `-Dappdynamics.agent.reuse.nodeName=true` (optionally `...reuse.nodeName.prefix`) so new instances reuse node names.

## Block 2 — Install the Java Agent (3.1)

1. **Unzip** to a folder the app user can write to (e.g. /opt/appdynamics/javaagent). Each release has its own ver&lt;version&gt;/ folder. Lock down: `chmod -R 700 <agent_home>/<ver>/conf`.

2. **Configure** &lt;agent_home&gt;/&lt;ver&gt;/conf/controller-info.xml (below).

3. **Load** the agent with -javaagent in the JVM start-up.

4. **Restart the JVM** — the agent only loads at JVM start.

```
<controller-host>ctrl.bank.local</controller-host>
<controller-port>8181</controller-port>        <!-- on-prem 8090/8181; SaaS 443 (or 80) -->
<controller-ssl-enabled>true</controller-ssl-enabled>
<application-name>PetClinic</application-name>
<tier-name>Web</tier-name>
<node-name>web01</node-name>
<account-name>customer1</account-name>         <!-- required for SaaS / multi-tenant -->
<account-access-key>xxxx</account-access-key>  <!-- required for ALL Controllers -->
```

### **Tomcat (Linux)** — $CATALINA_BASE/bin/setenv.sh:

```
export CATALINA_OPTS="$CATALINA_OPTS -javaagent:/opt/appdynamics/javaagent/javaagent.jar"
```

- Windows: setenv.bat with `set CATALINA_OPTS=%CATALINA_OPTS% -javaagent:C:\appagent\javaagent.jar`.

- Tomcat as a Windows service: tomcat&lt;ver&gt;w.exe → Java tab → add the -javaagent line.

## Block 3 — Three ways to set the same value

|**Setting**|**controller-info.xml**|**System property**|**Environment variable**|
|---|---|---|---|
|Controller host|controller-host|-Dappdynamics.controller.hostName|APPDYNAMICS_CONTROLLER_HOST_NAME|
|Port|controller-port|-Dappdynamics.controller.port|APPDYNAMICS_CONTROLLER_PORT|
|SSL|controller-ssl-enabled|-Dappdynamics.controller.ssl.enabled|APPDYNAMICS_CONTROLLER_SSL_ENABLED|
|Account|account-name|-Dappdynamics.agent.accountName|APPDYNAMICS_AGENT_ACCOUNT_NAME|
|Access key|account-access-key|-Dappdynamics.agent.accountAccessKey|APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY|
|Application|application-name|-Dappdynamics.agent.applicationName|APPDYNAMICS_AGENT_APPLICATION_NAME|
|Tier|tier-name|-Dappdynamics.agent.tierName|APPDYNAMICS_AGENT_TIER_NAME|
|Node|node-name|-Dappdynamics.agent.nodeName|APPDYNAMICS_AGENT_NODE_NAME|

Use the XML for stable settings, system properties when several JVMs **share one agent folder** (each needs its own node name), and environment variables for **containers**.

## Block 4 — Verify and troubleshoot

- Success: the node appears under **Tiers & Nodes** and starts reporting business transactions.

- Logs: `<agent_home>/ver<version>/logs/<node_name>/agent.log` — always read this first.

|**Symptom**|**Usual cause**|
|---|---|
|Node never appears|Wrong host/port, firewall, wrong access key or account name|
|SSL errors in agent.log|controller-ssl-enabled vs port mismatch, or Controller certificate not trusted by the agent truststore (cacerts.jks)|
|Node under wrong application/tier|A higher-priority source (env var or -D) overrides controller-info.xml|
|Agent not loaded at all|-javaagent missing or wrong path, or JVM not restarted|

## Block 5 — Upgrade the Java Agent (3.9)

1. **Upgrade the Controller first**; upgrade agents on the tiers where BTs originate last.

2. Stop the app server; **back up** the agent folder.

3. Unzip the new agent to a new folder. **Rename** the old folder, then give the new folder the **old name** — the -javaagent path stays the same.

4. Copy controller-info.xml (and agent.properties if used). For app-agent-config.xml and other files, **copy your changes** into the new file — don't replace it.

5. SSL? Copy cacerts.jks too and make sure the agent user can read it.

6. **Restart the app server.** No Controller action needed.

7. After confirming, archive the old folder.

## Block 6 — Install the .NET Agent on Windows (3.2)

- Installer **dotNetAgentSetup64.msi**, from an **elevated** prompt (Administrators group alone isn't enough). Install once per machine.

- **AppDynamics Agent Configuration utility:** Controller connection + **IIS** apps to instrument.

- • Config: `%ProgramData%\AppDynamics\DotNetAgent\Config\config.xml` (edit as administrator). Logs: `%ProgramData%\AppDynamics\DotNetAgent\Logs`.

- **AppDynamics.Agent.Coordinator** service manages instrumentation. After a change: restart the coordinator, then the apps (iisreset for IIS).

- Windows services and standalone apps are **not** set up by the utility — add them to config.xml by hand:

```
<app-agents>
  <standalone-applications>
    <standalone-application executable="OrderProcessor.exe">
      <tier name="OrderProcessor"/>
    </standalone-application>
  </standalone-applications>
</app-agents>
```

Use the command-line attribute to tell apart two instances of the same executable; configure a **POCO entry point** if no framework is auto-detected.

### Silent install:

```
msiexec /i dotNetAgentSetup64.msi /q /norestart /lv %TEMP%\AgentInstaller.log ^
        AD_SetupFile=C:\appd\setup.xml
net start AppDynamics.Agent.Coordinator
iisreset
```

Setup file root element: &lt;winston&gt;. Optional properties: INSTALLDIR, DOTNETAGENTFOLDER.

## Block 7 — Upgrade the .NET Agent (3.10)

1. **From 3.9 or later:** no uninstall needed. **Before 3.9:** uninstall first.

2. Stop the instrumented apps (IIS w3wp processes, Windows services, standalone apps).

3. Elevated prompt: `msiexec /i dotNetAgentSetup.msi /l log.txt /q AD_CONTROLLER_ACCOUNT_ACCESS_KEY=<key>` (+ INSTALLDIR= for a custom location). The MSI keeps existing configuration.

4. Restart: **Restart IIS** in the Agent Configuration utility (or iisreset), plus services/apps.

## Block 8 — Custom correlation (1.6)

**When:** a call goes over an **unsupported protocol or framework**, so the downstream tier appears as a separate BT.

- **Producer (caller):** custom **exit point** injects the correlation header (default key **singularityheader**) into the payload (header map, message properties).

- **Consumer (receiver): POJO entry point** reads the header and continues the **same BT**.

- Configured in **custom-activity-correlation.xml** next to app-agent-config.xml (template in the agent conf folder). Java Agent **3.8+**. Change the key with &lt;header-key&gt; inside &lt;payload-operation&gt;.

- **Not supported:** cross-process correlation by an ID stored in a file/database; **Sun RMI** (no place to carry the header).

## Lab — SaaS trial (~60 min)

1. **Java Agent:** Tomcat + Spring PetClinic on Linux; application PetClinic, tier Web, node web01, SaaS host on 443 with SSL. Confirm the node and at least 3 BTs.

2. **Precedence test:** keep node-name=web01 in the XML, add -Dappdynamics.agent.nodeName=web-sysprop and export APPDYNAMICS_AGENT_NODE_NAME=web-env. Restart. Which name appears, and why?

3. **.NET Agent:** install the MSI on an IIS sample site, configure with the utility, iisreset, confirm a second tier.

4. **Break it:** set a wrong access key, restart, find the error in agent.log, fix it.

5. **Paper design:** an order service sends messages to a billing service over a proprietary TCP library and they show as separate BTs. Describe the custom correlation on each side.

## Day 6 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Node name set in controller-info.xml, as -Dappdynamics.agent.nodeName and as APPDYNAMICS_AGENT_NODE_NAME (all different). Which wins?**

- A. controller-info.xml
- B. The system property
- C. The environment variable
- D. Random

<details><summary>Show answer</summary>

**Answer: C** — Env vars &gt; system properties &gt; agent.properties &gt; controller-info.xml; first non-empty value wins.

</details>

**Q2. Which controller-info.xml element is required for every Controller?**

- A. account-name
- B. account-access-key
- C. controller-ssl-enabled
- D. node-name

<details><summary>Show answer</summary>

**Answer: B** — Access key is always required; account-name is required for SaaS / multi-tenant.

</details>

**Q3. How do you load the Java Agent into Tomcat on Linux?**

- A. Copy the jar to Tomcat lib
- B. Add -javaagent to CATALINA_OPTS in setenv.sh, restart
- C. Deploy as a WAR
- D. Register in the Controller UI

<details><summary>Show answer</summary>

**Answer: B** — setenv.sh CATALINA_OPTS, then restart the JVM.

</details>

**Q4. Default Java Agent log location?**

- A. &lt;controller_home&gt;/logs
- B. &lt;agent_home&gt;/ver&lt;version&gt;/logs/&lt;node_name&gt;/
- C. /var/log/appdynamics
- D. Tomcat logs only

<details><summary>Show answer</summary>

**Answer: B** — Start with agent.log.

</details>

**Q5. Why rename the new agent folder to the old name during upgrade?**

- A. Controller rejects new folders
- B. So the -javaagent path stays the same
- C. To keep the license
- D. Required for SSL

<details><summary>Show answer</summary>

**Answer: B** — Startup scripts don't change; still copy config, merge changes, copy cacerts.jks, restart.

</details>

**Q6. .NET config.xml default location?**

- A. C:\Program Files\AppDynamics
- B. %ProgramData%\AppDynamics\DotNetAgent\Config\config.xml
- C. web.config
- D. %TEMP%

<details><summary>Show answer</summary>

**Answer: B** — Edit as admin; restart coordinator then apps.

</details>

**Q7. After editing config.xml for a Windows service, restart what, in order?**

- A. Only the Controller
- B. AppDynamics.Agent.Coordinator, then the Windows service
- C. The whole server
- D. Nothing

<details><summary>Show answer</summary>

**Answer: B** — Coordinator first, then the instrumented app.

</details>

**Q8. How do you instrument a Windows service with the .NET Agent?**

- A. The utility detects it
- B. standalone-application (executable + tier) under standalone-applications in config.xml
- C. Separate MSI
- D. Not supported

<details><summary>Show answer</summary>

**Answer: B** — The utility only handles IIS; services and standalone apps are manual.

</details>

**Q9. Upgrading a .NET Agent from 3.7 — what differs from 4.x?**

- A. Nothing
- B. Before 3.9 you must uninstall first
- C. Upgrade Events Service first
- D. Reinstall IIS

<details><summary>Show answer</summary>

**Answer: B** — 3.9+ upgrades in place; older versions must be uninstalled first.

</details>

**Q10. Java custom correlation — what does each side do?**

- A. Consumer injects, producer reads
- B. Producer exit point injects singularityheader; consumer POJO entry point reads it
- C. Controller matches by timestamp
- D. Shared database ID

<details><summary>Show answer</summary>

**Answer: B** — custom-activity-correlation.xml, Java Agent 3.8+; ID-in-database correlation and Sun RMI unsupported.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|Precedence|Env vars &gt; system props &gt; agent.properties (ver, global) &gt; controller-info.xml (ver, global)|
|Required always|account-access-key (account-name for SaaS / multi-tenant)|
|Ports|On-prem 8090 / 8181 · SaaS 443 (or 80)|
|Load Java agent|-javaagent:&lt;agent_home&gt;/javaagent.jar · Tomcat setenv.sh CATALINA_OPTS · restart JVM|
|Node names|Unique per application + host · reuse.nodeName for containers|
|Java logs|&lt;agent_home&gt;/ver&lt;version&gt;/logs/&lt;node&gt;/agent.log|
|Java upgrade|Controller first · rename folders (same path) · copy controller-info.xml, merge config, cacerts.jks · restart|
|.NET install|dotNetAgentSetup64.msi (elevated) · Agent Configuration utility (IIS only) · iisreset|
|.NET files|%ProgramData%\AppDynamics\DotNetAgent\Config\config.xml · ...\Logs|
|.NET coordinator|AppDynamics.Agent.Coordinator — restart it, then the apps|
|Windows services|standalone-applications / standalone-application executable + tier in config.xml|
|.NET upgrade|≥ 3.9 in place · &lt; 3.9 uninstall first · stop w3wp · restart IIS|
|Custom correlation|Unsupported protocol/framework · exit point injects, POJO entry point reads · custom-activity-correlation.xml|

## Sources

- [Configuration Options and Precedence (on-prem 26.9.0)](https://help.splunk.com/en/appdynamics-on-premises/application-performance-monitoring/26.9.0/install-app-server-agents/java-agent/administer-the-java-agent/configuration-options-and-precedence)

- [Configure the Java Agent](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/25.7.0/install-app-server-agents/java-agent/install-the-java-agent/configure-the-java-agent)

- [Use Environment Variables for Java Agent Settings](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.4.0/install-app-server-agents/java-agent/install-the-java-agent/use-environment-variables-for-java-agent-settings)

- [Agent Identification Properties](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.9.0/install-app-server-agents/java-agent/administer-the-java-agent/java-agent-configuration-properties/agent-identification-properties)

- [Apache Tomcat Startup Settings](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.5.0/install-app-server-agents/java-agent/install-the-java-agent/agent-installation-by-java-framework/apache-tomcat-startup-settings)

- [Locate the Java Agent Log Files](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/25.7.0/install-app-server-agents/java-agent/administer-the-java-agent/troubleshooting-java-agent-issues/locate-the-java-agent-log-files)

- [Upgrade the Java Agent](https://help.splunk.com/en/appdynamics-on-premises/application-performance-monitoring/26.9.0/install-app-server-agents/java-agent/administer-the-java-agent/upgrade-the-java-agent)

- [Unattended Installation for .NET](https://help.splunk.com/en/appdynamics-on-premises/application-performance-monitoring/26.9.0/install-app-server-agents/.net-agent/install-the-.net-agent-for-windows/unattended-installation-for-.net)

- [Configure .NET Agent Properties](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.4.0/install-app-server-agents/.net-agent/administer-the-.net-agent/configure-agent-properties)

- [Configure the .NET Agent for Windows Services and Standalone Applications](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.9.0/install-app-server-agents/.net-agent/install-the-.net-agent-for-windows/configure-the-.net-agent-for-windows-services-and-standalone-applications)

- [Upgrade the .NET Agent from &gt;= 3.9](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/26.6.0/install-app-server-agents/.net-agent/install-the-.net-agent-for-windows/upgrade-the-.net-agent-for-windows/upgrade-the-.net-agent-from--3.9)

- [Custom Correlation for Java Applications (Knowledge Base)](https://community.splunk.com/t5/AppDynamics-Knowledge-Base/Custom-Correlation-for-Java-Applications/ta-p/735223)

## Lab — model answers

What a correct result looks like for each Day 6 lab task, with the rule behind it.

### 1. Java Agent on Tomcat + PetClinic (SaaS)

```
<controller-host>YOUR-ACCOUNT.saas.appdynamics.com</controller-host>
<controller-port>443</controller-port>
<controller-ssl-enabled>true</controller-ssl-enabled>
<application-name>PetClinic</application-name>
<tier-name>Web</tier-name>
<node-name>web01</node-name>
<account-name>YOUR-ACCOUNT</account-name>
<account-access-key>YOUR-ACCESS-KEY</account-access-key>
```

- setenv.sh: export CATALINA_OPTS="$CATALINA_OPTS -javaagent:/opt/appdynamics/javaagent/javaagent.jar" → restart Tomcat.

- Click around PetClinic to generate traffic; within a few minutes the node web01 appears under Tiers & Nodes and BTs (e.g. owners, vets pages) appear.

- The account name and access key come from your SaaS Controller's license/account settings or welcome email.

### 2. Precedence test

|**Sources set**|**Node name shown**|**Why**|
|---|---|---|
|XML + -D + env var|**web-env**|Environment variables have the highest priority|
|XML + -D (env var removed)|**web-sysprop**|System properties beat the files|
|XML only|**web01**|controller-info.xml is used only when nothing higher is set|

Each test creates a new node in the Controller, since node names differ. Exam takeaway: when a node shows up with an unexpected name, look for an environment variable or -D property first.

### 3. .NET Agent on IIS

1. Run dotNetAgentSetup64.msi from an elevated prompt (or double-click as administrator).

2. AppDynamics Agent Configuration utility: Controller host, port 443, SSL on, account name, access key, application name.

3. Select the IIS site(s) to instrument and assign a tier (e.g. Payments-IIS).

4. Finish, then Restart IIS in the utility (or iisreset). Confirm the AppDynamics.Agent.Coordinator service is running.

5. Browse the site to generate traffic → a second tier appears on the flow map and under Tiers & Nodes.

### 4. Wrong access key

- The node never appears. agent.log in &lt;agent_home&gt;/ver&lt;version&gt;/logs/web01/ shows the agent failing to register / authenticate with the Controller.

- Fix the key where it is **actually** being read from — if an environment variable or -D sets it, editing controller-info.xml does nothing (precedence).

- Restart the JVM and confirm the node registers.

### 5. Custom correlation design: order service → billing service over a proprietary TCP library

|**Side**|**Configuration**|
|---|---|
|Producer (order service)|Define a **custom exit point** on the library's send method (e.g. TcpClient.send(Message)). Add a payload operation that writes the correlation header (**singularityheader**) into the message's header map / properties.|
|Consumer (billing service)|Define a **POJO entry point** on the receive/handle method (e.g. BillingHandler.onMessage(Message)) and configure it to read the same header key from the payload.|
|Where|**custom-activity-correlation.xml** next to app-agent-config.xml on both agents (Java Agent 3.8+). Optional validation methods on each side. Restart both JVMs.|
|Verify|One BT now flows Order → Billing on the flow map instead of two separate BTs.|
|Limit|If the message has nowhere to carry the header (like Sun RMI), or the link is only an ID in a database/file, custom correlation can't join them.|

---

[← Day 5](day-05-events-service-and-eum-server.md) · [🏠 Home](../README.md) · [Day 7 →](day-07-machine-database-browser-mobile-agents.md)
