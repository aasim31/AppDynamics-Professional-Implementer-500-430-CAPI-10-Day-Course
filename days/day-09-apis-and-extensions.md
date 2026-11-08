# Day 9 — APIs and Extensions

[🏠 Home](../README.md) · [← Day 8](day-08-verification-rbac-and-revision.md) · Day 10 (mock exam) — coming soon · [📄 PDF version](../pdf/Day09_APIs_and_Extensions.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 5.1–5.6 (Domain 5 = 20%)

**Goal:** authenticate to the Controller APIs, retrieve applications, tiers, nodes, BTs, metrics and health rule violations, create custom events, and build and install alerting and monitoring extensions.

**Verification:** every fact checked against the current Splunk AppDynamics docs (November 2026). Version note: the newest docs (25.10+) describe metric-data-v2; older Controllers and exam material use the classic metric-data — both are shown.

## Block 1 — API basics and authentication

- Base path: https://&lt;controller&gt;/controller/rest/... · XML is the **default**, add **output=JSON** · URL-encode spaces and pipes in metric paths.

- **Basic auth:** username = &lt;user&gt;@&lt;account&gt; — on-prem single-tenant account **customer1**; SaaS = your account name.

```
curl --user admin@customer1:<password> "https://<ctrl>/controller/rest/applications?output=JSON"
```

### API client + OAuth token (recommended for scripts)

1. **Administration → API Clients → + Create**: name, Generate Secret, token expiry, roles. Needs **Account Owner** (or Administer users, groups, roles).

2. Request a token (POST, form-encoded) — reply has access_token (JWT) and expires_in. **Default expiry: 5 minutes.**

3. Send Authorization: Bearer &lt;token&gt; on each call. Invalid/expired token → **HTTP 401**.

```
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  "https://<ctrl>/controller/api/oauth/access_token" \
  -d 'grant_type=client_credentials&client_id=<clientName>@<account>&client_secret=<secret>'
```

```
curl -H "Authorization: Bearer <access_token>" \
  "https://<ctrl>/controller/rest/applications?output=JSON"
```

## Block 2 — Application Model API (5.5)

|**Retrieve**|**URI (GET)**|
|---|---|
|Business applications|/controller/rest/applications|
|Business transactions|/controller/rest/applications/&lt;app&gt;/business-transactions|
|Tiers|/controller/rest/applications/&lt;app&gt;/tiers|
|Nodes in the application|/controller/rest/applications/&lt;app&gt;/nodes|
|Nodes in one tier|/controller/rest/applications/&lt;app&gt;/tiers/&lt;tier&gt;/nodes|
|Backends|/controller/rest/applications/&lt;app&gt;/backends|

&lt;app&gt; can be the application **name or ID**.

## Block 3 — Metric and Snapshot API (5.6)

- **Browse the tree:** GET /controller/rest/applications/&lt;app&gt;/metrics?metric-path=&lt;path&gt; → items are **folder** or **leaf**.

```
GET /controller/rest/applications/<app>/metric-data      <- classic (older Controllers / exam)
GET /controller/rest/applications/<app>/metric-data-v2   <- newest docs (25.10+)
    ?metric-path=Overall Application Performance|Average Response Time (ms)
    &time-range-type=BEFORE_NOW&duration-in-mins=60&rollup=false&output=JSON
```

|**Parameter**|**Rule**|
|---|---|
|time-range-type|BEFORE_NOW (+ duration-in-mins), BEFORE_TIME, AFTER_TIME, BETWEEN_TIMES (+ start-time / end-time in **milliseconds**)|
|rollup|**Default true** = one combined value for the range; false = every data point|
|Wildcards|* in a path segment, e.g. Application Infrastructure Performance\|*\|Agent\|App\|Availability = every tier|

- **Copy REST URL:** right-click a metric in the Metric Browser — exact path plus the selected time range.

- Granularity: 1-minute for ranges up to 24 h within the last 8 days; longer ranges hourly. On-prem retention defaults 4 h / 48 h / 365 days.

- **Snapshots:** GET .../request-snapshots — same time parameters, filters business-transaction-ids, user-experience, error-occurred, maximum-results (default **600**).

## Block 4 — Alert and Respond API (5.2, 5.4)

```
GET /controller/rest/applications/<app>/problems/healthrule-violations
    ?time-range-type=BEFORE_NOW&duration-in-mins=60&output=JSON
```

```
GET /controller/rest/applications/<app>/events
    ?time-range-type=BEFORE_NOW&duration-in-mins=30
    &event-types=APPLICATION_ERROR,DIAGNOSTIC_SESSION&severities=INFO,WARN,ERROR
```

Events: at most **600** per call. **Create a custom event** (needs the Create Events permission):

```
curl -X POST --user admin@customer1:<pwd> \
 "https://<ctrl>/controller/rest/applications/PetClinic/events?severity=INFO\
&summary=Release%202.1%20deployed&eventtype=CUSTOM&customeventtype=deployment\
&propertynames=version&propertynames=team&propertyvalues=2.1&propertyvalues=web"
```

|**Parameter**|**Rule**|
|---|---|
|summary, eventtype=CUSTOM, severity|**Required**; severity INFO, WARN or ERROR|
|customeventtype|Optional label (deployment, nagios ...)|
|node, tier, bt|Optional; **tier required** if node or bt is given|
|propertynames / propertyvalues|Paired **by position** — N names need N values; values up to 5,000 characters|

## Block 5 — Configuration Import and Export API

|**What**|**Export**|**Import**|**Format**|
|---|---|---|---|
|Actions|GET /controller/actions/&lt;app_id&gt;|POST same path|JSON|
|Health rules|GET /controller/healthrules/&lt;app_id&gt;?name=&lt;rule&gt;|POST /controller/healthrules/&lt;app_id&gt;?overwrite=true\|false|XML|
|Transaction detection|GET /controller/transactiondetection/&lt;app_id&gt;/...|POST with -F file=@rules.xml|XML|

overwrite defaults to **false**. These paths are under /controller/, **not** /controller/rest/. The docs recommend the Health Rule API over XML import for health rules.

## Block 6 — Alerting extensions: custom actions (5.1, 5.3)

**On-prem only** (SaaS: contact support).

1. Script in &lt;controller_home&gt;/custom/actions/&lt;type&gt;/ (.sh executable on Linux, .bat on Windows).

2. Describe it in &lt;controller_home&gt;/custom/actions/custom.xml.

3. **Restart the Controller.**

4. Controller UI: create a **Custom Action** and attach it to a **policy**.

```
<custom-actions>
  <action>
    <type>servicenow</type>            <!-- = subfolder name -->
    <executable>create-incident.sh</executable>
  </action>
</custom-actions>
```

- **Downloaded alerting extension (5.3):** unzip into custom/actions/&lt;type&gt;/, add the custom.xml entry, fill in its config, restart the Controller, create the action, attach to a policy.

- **Script input:** positional parameters. Health rule violation: $1 APP_NAME, $2 APP_ID, then PVN_ALERT_TIME, PRIORITY, SEVERITY, ACTION_NAME, HEALTH_RULE_NAME, HEALTH_RULE_ID ... ending with DEEP_LINK_URL. Other events use EN_TIME.

- **HTTP Request Actions:** template-based webhooks (ServiceNow, Slack) — no script, no Controller restart.

## Block 7 — Monitoring extensions with scripts (Machine Agent)

1. Create &lt;machine_agent_home&gt;/monitors/&lt;MyMonitor&gt;/ (agent user needs read/write/execute).

2. Script prints one line per metric to STDOUT (format below).

3. monitor.xml: &lt;type&gt;managed&lt;/type&gt;; execution-style **periodic** (every minute by default) or **continuous** (script loops); &lt;executable-task&gt; names the script per OS.

4. **Restart the Machine Agent**; the metric appears after about a minute.

5. Windows: use PowerShell or VBScript, not .bat (batch quotes metric names, so they're ignored).

```
name=Custom Metrics|Queue|Depth,value=42,aggregator=OBSERVATION,time-rollup=CURRENT,
     cluster-rollup=COLLECTIVE          (one line in real output)
```

|**Part**|**Options**|
|---|---|
|value|**Long integer** (whole number)|
|aggregator (per minute)|**AVERAGE** (default), SUM, OBSERVATION (last value)|
|time-rollup|AVERAGE, SUM, CURRENT|
|cluster-rollup|INDIVIDUAL (average across nodes), COLLECTIVE (add across nodes)|
|Path prefix|Custom Metrics\|... = all tiers · Server\|Component:&lt;tier&gt;\|... = one tier (Machine Agent associated with it)|

## Lab — SaaS + curl (~60 min)

1. **API client:** create one, request a token, call /controller/rest/applications?output=JSON.

2. **Application model:** PetClinic tiers, nodes, BTs and backends as JSON.

3. **Metrics:** browse /metrics; Copy REST URL for Average Response Time; run with rollup=true and rollup=false; wildcard Availability for every tier.

4. **Custom event:** post a deployment event with version=2.1 and team=web; find it in PetClinic's Events.

5. **Violations:** last 60 minutes of health rule violations from the Day 8 rule.

6. **Paper:** (a) custom.xml + script outline for an on-prem "servicenow" alerting extension; (b) a Linux monitoring script reporting Custom Metrics|Lab|TCP Connections.

## Day 9 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Basic-auth username format for a single-tenant on-prem Controller?**

- A. customer1/admin
- B. admin@customer1
- C. admin:customer1
- D. root@system

<details><summary>Show answer</summary>

**Answer: B** — &lt;user&gt;@&lt;account&gt;; on-prem account customer1.

</details>

**Q2. Default lifetime of an API-client OAuth token?**

- A. 5 minutes
- B. 1 hour
- C. 24 hours
- D. Never expires

<details><summary>Show answer</summary>

**Answer: A** — expires_in 300; configurable per client.

</details>

**Q3. URI listing all nodes in tier Web of PetClinic?**

- A. /controller/rest/nodes?tier=Web
- B. /controller/rest/applications/PetClinic/tiers/Web/nodes
- C. /controller/rest/applications/PetClinic/nodes/Web
- D. /controller/api/tiers/Web

<details><summary>Show answer</summary>

**Answer: B** — App by name or ID.

</details>

**Q4. metric-data returns one value for 60 minutes — why?**

- A. No data
- B. rollup defaults to true; use rollup=false
- C. output=JSON missing
- D. BEFORE_NOW returns latest only

<details><summary>Show answer</summary>

**Answer: B** — rollup=true combines the range.

</details>

**Q5. Easiest way to get a correct metric path?**

- A. Guess from dashboard
- B. Metric Browser → Copy REST URL
- C. agent.log
- D. Health rule XML

<details><summary>Show answer</summary>

**Answer: B** — Includes path and time range.

</details>

**Q6. URI for health rule violations in the last hour?**

- A. .../problems/healthrule-violations?time-range-type=BEFORE_NOW&duration-in-mins=60
- B. /controller/healthrules/&lt;app_id&gt;
- C. .../events?eventtype=CUSTOM
- D. /controller/api/violations

<details><summary>Show answer</summary>

**Answer: A** — /controller/healthrules is the config export.

</details>

**Q7. Custom event with node=web01 — which parameter becomes required?**

- A. bt
- B. tier
- C. comment
- D. customeventtype

<details><summary>Show answer</summary>

**Answer: B** — tier is required with node or bt.

</details>

**Q8. New custom action script + custom.xml installed — next?**

- A. Restart Machine Agent
- B. Restart Controller, create Custom Action, attach to policy
- C. Nothing
- D. Upload via REST

<details><summary>Show answer</summary>

**Answer: B** — On-prem only.

</details>

**Q9. Valid Machine Agent script output line?**

- A. Custom Metrics|Queue|Depth=42.5
- B. name=Custom Metrics|Queue|Depth,value=42,aggregator=OBSERVATION
- C. metric: Queue Depth 42
- D. JSON object

<details><summary>Show answer</summary>

**Answer: B** — Values are long integers.

</details>

**Q10. Health rule import, existing same-name rule — default?**

- A. Replaced
- B. Not overwritten (overwrite=false)
- C. Both kept
- D. Import fails

<details><summary>Show answer</summary>

**Answer: B** — Set overwrite=true to replace.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|Auth|Basic user@account (on-prem customer1) · OAuth API client → Bearer token (5 min default)|
|Token URL|POST /controller/api/oauth/access_token · grant_type=client_credentials · client_id=name@account|
|Format|XML default · output=JSON|
|App model|/applications · /&lt;app&gt;/tiers · /nodes · /tiers/&lt;tier&gt;/nodes · /business-transactions · /backends|
|Metrics|/&lt;app&gt;/metrics (browse) · /metric-data (classic) or /metric-data-v2 · rollup default true · * wildcard|
|Time ranges|BEFORE_NOW · BEFORE_TIME · AFTER_TIME · BETWEEN_TIMES (ms)|
|Violations|/&lt;app&gt;/problems/healthrule-violations|
|Custom event|POST /&lt;app&gt;/events · summary + eventtype=CUSTOM + severity · tier if node/bt · names/values by position|
|Limits|600 events per call · 600 snapshots default|
|Config export|/controller/actions (JSON) · /controller/healthrules (XML, overwrite=false default)|
|Custom action|custom/actions/&lt;type&gt;/script + custom.xml · restart Controller · action → policy · on-prem only|
|Script args|$1 APP_NAME · $2 APP_ID · ... · DEEP_LINK_URL last|
|Script extension|monitors/&lt;name&gt;/ + monitor.xml · name=...,value=&lt;long&gt; · restart Machine Agent|

## Sources

- [API Clients](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.9.0/extend-splunk-appdynamics/splunk-appdynamics-apis/api-clients)

- [Using the Controller APIs](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.3.0/extend-splunk-appdynamics/splunk-appdynamics-apis/using-the-controller-apis)

- [Application Model API](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.9.0/extend-splunk-appdynamics/splunk-appdynamics-apis/application-model-api)

- [Metric and Snapshot API](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.3.0/extend-splunk-appdynamics/splunk-appdynamics-apis/metric-and-snapshot-api)

- [Events API (Alert and Respond)](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.3.0/extend-splunk-appdynamics/splunk-appdynamics-apis/alert-and-respond-api/events-api)

- [Configuration Import and Export API](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.9.0/extend-splunk-appdynamics/splunk-appdynamics-apis/configuration-import-and-export-api)

- [Create a Custom Action (on-prem)](https://help.splunk.com/en/appdynamics-on-premises/get-started/26.9.0/alert-and-respond/actions/build-a-custom-action/create-a-custom-action-for-splunk-appdynamics-on-premises)

- [Contents of the custom.xml File](https://help.splunk.com/appdynamics-on-premises/get-started/25.7.0/alert-and-respond/actions/build-a-custom-action/contents-of-the-custom.xml-file)

- [Information Passed to the Custom Action Script](https://help.splunk.com/en/appdynamics-on-premises/get-started/26.9.0/alert-and-respond/actions/build-a-custom-action/information-passed-to-the-custom-action-script)

- [Build a Monitoring Extension Using Scripts](https://help.splunk.com/en/appdynamics-on-premises/infrastructure-visibility/26.3.0/machine-agent/extensions-and-custom-metrics/build-a-monitoring-extension-using-scripts)

## Lab — model answers

What a correct result looks like for each Day 9 lab task. Replace &lt;ctrl&gt;, &lt;account&gt; and secrets with your own; use HTTPS on SaaS.

### 1. API client and token

```
TOKEN=$(curl -s -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  "https://<ctrl>/controller/api/oauth/access_token" \
  -d 'grant_type=client_credentials&client_id=labclient@<account>&client_secret=<secret>' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["access_token"])')
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://<ctrl>/controller/rest/applications?output=JSON"
```

Expected: a JSON array of applications (name, id, description). After 5 minutes the same token returns HTTP 401 — request a new one.

### 2. Application model

```
B="https://<ctrl>/controller/rest/applications/PetClinic"
curl -s -H "Authorization: Bearer $TOKEN" "$B/tiers?output=JSON"
curl -s -H "Authorization: Bearer $TOKEN" "$B/nodes?output=JSON"
curl -s -H "Authorization: Bearer $TOKEN" "$B/tiers/Web/nodes?output=JSON"
curl -s -H "Authorization: Bearer $TOKEN" "$B/business-transactions?output=JSON"
curl -s -H "Authorization: Bearer $TOKEN" "$B/backends?output=JSON"
```

Expected: tier Web (and the .NET tier from Day 6), node web01, the PetClinic BTs, and backends such as the MySQL database.

### 3. Metrics

```
curl -s -H "Authorization: Bearer $TOKEN" "$B/metrics?output=JSON"     # top-level folders
curl -s -G -H "Authorization: Bearer $TOKEN" "$B/metric-data" \
  --data-urlencode "metric-path=Overall Application Performance|Average Response Time (ms)" \
  -d time-range-type=BEFORE_NOW -d duration-in-mins=60 -d rollup=false -d output=JSON
# wildcard: Availability for every tier
```

```
  --data-urlencode "metric-path=Application Infrastructure Performance|*|Agent|App|Availability"
```

- rollup=true → one entry in metricValues (the combined value for the hour). rollup=false → about 60 one-minute values.

- curl -G with --data-urlencode handles the spaces and pipes. On a newer Controller, metric-data-v2 accepts the same parameters.

- Copy REST URL from the Metric Browser gives a ready-made URL you can paste into curl.

### 4. Custom deployment event

```
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
 "$B/events?severity=INFO&summary=Release%202.1%20deployed&eventtype=CUSTOM\
&customeventtype=deployment&propertynames=version&propertynames=team\
&propertyvalues=2.1&propertyvalues=web"
```

Expected: the reply confirms creation; PetClinic → Events (filter Custom) shows "Release 2.1 deployed" with properties version=2.1, team=web. The API client's role needs the Create Events permission.

### 5. Health rule violations

```
curl -s -H "Authorization: Bearer $TOKEN" \
 "$B/problems/healthrule-violations?time-range-type=BEFORE_NOW&duration-in-mins=60&output=JSON"
```

Expected: entries for the Day 8 rule (name, severity, start time, affected BT) if it fired; an empty array if not — generate slow requests and retry.

### 6a. On-prem "servicenow" alerting extension

```
<controller_home>/custom/actions/
├── custom.xml
└── servicenow/
    └── create-incident.sh        (chmod +x)
<custom-actions>
  <action>
    <type>servicenow</type>
    <executable>create-incident.sh</executable>
  </action>
</custom-actions>
#!/bin/bash
APP_NAME="$1"; APP_ID="$2"; ALERT_TIME="$3"; PRIORITY="$4"; SEVERITY="$5"
ACTION_NAME="$6"; RULE_NAME="$7"; RULE_ID="$8"
DEEP_LINK="${@: -1}"                   # DEEP_LINK_URL is the last argument
curl -s -u "$SN_USER:$SN_PASS" -H "Content-Type: application/json" \
  -X POST "https://bank.service-now.com/api/now/table/incident" \
  -d "{\"short_description\":\"AppD $SEVERITY: $RULE_NAME on $APP_NAME\",
       \"description\":\"$DEEP_LINK\"}"
```

Then **restart the Controller**, create a Custom Action of type servicenow, and attach it to a policy on health rule violations. On SaaS, use an HTTP Request Action template instead.

### 6b. Monitoring script: Custom Metrics|Lab|TCP Connections

```
#!/bin/bash                    # <MA_HOME>/monitors/TcpMonitor/tcp.sh
COUNT=$(ss -tan state established | tail -n +2 | wc -l)
echo "name=Custom Metrics|Lab|TCP Connections,value=${COUNT},aggregator=OBSERVATION,\
time-rollup=CURRENT,cluster-rollup=INDIVIDUAL"
```

```
<monitor>
  <name>TcpMonitor</name>
  <type>managed</type>
  <monitor-run-task>
    <execution-style>periodic</execution-style>
    <name>Run</name>
    <type>executable</type>
    <task-arguments></task-arguments>
    <executable-task>
      <type>file</type>
      <file os-type="linux">tcp.sh</file>
    </executable-task>
  </monitor-run-task>
</monitor>
```

The echo must print the whole line without a line break (the backslash above only wraps it for the page). chmod +x tcp.sh, restart the Machine Agent, and after about a minute the metric appears under Custom Metrics → Lab → TCP Connections.

## Postman equivalents of every curl request

The same Day 9 requests (plus the Day 7 Machine Agent listener) set up in Postman. Import the two JSON files sent with these notes ( **AppDynamics_Day9_Postman_Collection.json** and **AppDynamics_Lab_Postman_Environment.json**), or build them by hand from the tables below.

### One-time setup

1. Postman → **Import** → select both JSON files.

2. Top-right environment selector → **AppDynamics Lab**; fill in the values below (Current value column).

3. The collection's Authorization tab is **Bearer Token = {{token}}**; every request inherits it unless shown otherwise.

4. Run **Get OAuth token** first — its Tests script saves the token into {{token}}. Re-run it after 5 minutes (HTTP 401 = expired).

|**Variable**|**Example value**|**Purpose**|
|---|---|---|
|baseUrl|https://&lt;your-account&gt;.saas.appdynamics.com|Controller URL, no trailing slash (on-prem e.g. http://ctrl.bank.local:8090)|
|account|&lt;your-account&gt;|On-prem single-tenant: customer1|
|user|&lt;username&gt;|For Basic auth|
|password|&lt;password&gt;|For Basic auth (secret)|
|clientName|labclient|API client name|
|clientSecret token|&lt;secret&gt;|API client secret (secret) Filled automatically by 'Get OAuth token'|
|app|PetClinic|Application name or ID|
|appId|&lt;numeric id&gt;|Numeric ID for config export/import|
|tier|Web|Tier name|
|maListener|http://localhost:8293|Machine Agent HTTP listener|

Tests script on **Get OAuth token**:

```
const j = pm.response.json();
pm.environment.set("token", j.access_token);
```

### Authentication

### Get OAuth token

|**Setting**|**Value**|
|---|---|
|Method + URL|**POST**{{baseUrl}}/controller/api/oauth/access_token|
|Authorization|No Auth|
|Body|x-www-form-urlencoded:<br>grant_type = client_credentials<br>client_id = {{clientName}}@{{account}}<br>client_secret = {{clientSecret}}|
|Note|Tests tab saves the token: pm.environment.set("token", pm.response.json().access_token). Token lasts 5 min by default.|

### List applications (Basic auth)

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications|
|Params tab|output = JSON|
|Authorization|Basic Auth — {{user}}@{{account}} / {{password}}|
|Note|Authorization tab → Basic Auth: Username {{user}}@{{account}}, Password {{password}}.|

### Application Model

### List applications

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Tiers

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/tiers|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Nodes in application

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/nodes|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Nodes in tier

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/tiers/{{tier}}/nodes|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Business transactions

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/business-transactions|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Backends

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/backends|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Metrics and Snapshots

### Browse metric tree

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/metrics|
|Params tab|output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|Add metric-path (e.g. Overall Application Performance) to go one level deeper.|

### Metric data - all points (rollup=false)

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/metric-data|
|Params tab|metric-path = Overall Application Performance\|Average Response Time (ms)<br>time-range-type = BEFORE_NOW<br>duration-in-mins = 60<br>rollup = false<br>output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|Postman encodes spaces and pipes in the Params tab. Newer Controllers (25.10+): metric-data-v2, same params.|

### Metric data - one value (rollup=true)

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/metric-data|
|Params tab|metric-path = Overall Application Performance\|Average Response Time (ms)<br>time-range-type = BEFORE_NOW<br>duration-in-mins = 60<br>rollup = true<br>output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|rollup=true is the default.|

### Metric data - wildcard, every tier

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/metric-data|
|Params tab|metric-path = Application Infrastructure Performance\|*\|Agent\|App\|Availability<br>time-range-type = BEFORE_NOW<br>duration-in-mins = 60<br>output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Request snapshots

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/request-snapshots|
|Params tab|time-range-type = BEFORE_NOW<br>duration-in-mins = 60<br>output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|maximum-results defaults to 600.|

### Alert and Respond

### Health rule violations

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/problems/healthrule-violations|
|Params tab|time-range-type = BEFORE_NOW<br>duration-in-mins = 60 output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|

### Get events

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/rest/applications/{{app}}/events|
|Params tab|time-range-type = BEFORE_NOW<br>duration-in-mins = 30<br>event-types = APPLICATION_ERROR,DIAGNOSTIC_SESSION,CUSTOM<br>severities = INFO,WARN,ERROR<br>output = JSON|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|Up to 600 events per call.|

### Create custom event

|**Setting**|**Value**|
|---|---|
|Method + URL|**POST**{{baseUrl}}/controller/rest/applications/{{app}}/events|
|Params tab|severity = INFO<br>summary = Release 2.1 deployed<br>eventtype = CUSTOM<br>customeventtype = deployment<br>propertynames = version<br>propertynames = team<br>propertyvalues = 2.1<br>propertyvalues = web|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|Repeat propertynames/propertyvalues as separate rows, in matching order. Needs the Create Events permission.|

### Config Import Export

### Export actions

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/actions/{{appId}}|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|Numeric application ID; JSON.|

### Export one health rule

|**Setting**|**Value**|
|---|---|
|Method + URL|**GET**{{baseUrl}}/controller/healthrules/{{appId}}|
|Params tab|name = PetClinic Response Time|
|Authorization|Inherit from collection (Bearer {{token}})|
|Note|XML.|

### Import health rules

|**Setting**|**Value**|
|---|---|
|Method + URL|**POST**{{baseUrl}}/controller/healthrules/{{appId}}|
|Params tab|overwrite = false|
|Authorization|Inherit from collection (Bearer {{token}})|
|Body|form-data: key **file** (type File) → the exported XML|
|Note|Body → form-data, key 'file' of type File. overwrite defaults to false.|

### Machine Agent listener (Day 7)

### Post custom metric

|**Setting**|**Value**|
|---|---|
|Method + URL|**POST**{{maListener}}/api/v1/metrics|
|Authorization|No Auth|
|Body|raw → JSON: [{"metricName":"Custom Metrics\|Lab\|Orders","aggregatorType":"SUM","value":10}]|
|Note|Full URL http://localhost:8293/api/v1/metrics; Header Content-Type: application/json. Run on the Machine Agent host (localhost only by default).|

---

[← Day 8](day-08-verification-rbac-and-revision.md) · [🏠 Home](../README.md) · Day 10 (mock exam) — coming soon
