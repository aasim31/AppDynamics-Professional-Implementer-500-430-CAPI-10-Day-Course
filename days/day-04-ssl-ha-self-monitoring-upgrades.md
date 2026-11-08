# Day 4 — SSL, High Availability, Self-Monitoring and Upgrades

[🏠 Home](../README.md) · [← Day 3](day-03-controller-install-and-enterprise-console.md) · [Day 5 →](day-05-events-service-and-eum-server.md) · [📄 PDF version](../pdf/Day04_SSL_HA_Self-Monitoring_Upgrades.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 2.2, 2.3, 2.4, 2.6 (Domain 2 = 25%)

**Goal:** secure the Controller with a CA certificate, design and run an HA pair, monitor the Controller itself, back it up, and upgrade the platform safely.

**Verification:** every fact checked against the current Splunk AppDynamics on-prem docs (October 2026).

## Block 1 — Controller SSL (2.3)

|**Item**|**Default**|
|---|---|
|HTTPS port|**8181**, self-signed certificate|
|Keystore|&lt;controller_home&gt;/appserver/jetty/etc/keystore.jks|
|Truststore|cacerts.jks (same folder) — CA root/intermediate certs, never private keys|
|Controller key alias|**s1as** — do not rename|
|Reporting Service alias|reporting-instance|
|Keystore password|**changeit** — well known, change it; keep keystore.jks and cacerts.jks on the same password|

### Replace the certificate (CSR flow)

```
cd <controller_home>/appserver/jetty/etc
cp keystore.jks keystore.jks.bak                        # 1. back up, stop the Controller
keytool -delete -alias s1as -keystore keystore.jks      # 2. remove self-signed key
keytool -genkeypair -alias s1as -keyalg RSA -keysize 2048 -validity 1825 \
        -keystore keystore.jks -ext SAN=dns:ctrl.bank.local   # 3. new key pair
keytool -certreq -alias s1as -keystore keystore.jks -file AppDynamics.csr   # 4. CSR
# 5. send CSR to the CA, receive signed certificate + chain
keytool -import -trustcacerts -alias s1as -file mycert.cer -keystore keystore.jks  # 6
# 7. restart the Controller
```

- **"Failed to establish chain from reply"** → import the root and intermediate CA certificates into **cacerts.jks**, then repeat step 6.

- **Existing key + certificate:** `openssl pkcs12 -export` → `keytool -importkeystore` → `-changealias` to **s1as**.

- **GUI:** Enterprise Console → Configurations → Controller Settings → Appserver Configurations → SSL Certificate Management → Edit Certificate.

### HTTPS only

1. Stop the app server.

2. In `start.ini`, set `-Dappdynamics.controller.port` and `-Dappdynamics.controller.services.port` to the SSL port.

3. Add `-Dappdynamics.controller.ssl.enabled=true` so internal agents use SSL.

4. Change the deep-link URL to https.

5. Verify at `https://<host>:8181/controller`.

### Agents over SSL (controller-info.xml)

```
<controller-port>8181</controller-port>   <!-- 443 via a proxy -->
<controller-ssl-enabled>true</controller-ssl-enabled>
<controller-keystore-password>...</controller-keystore-password>
<!-- only if the truststore is not the default cacerts.jks: -->
<controller-keystore-filename>../../conf/cacerts.jks</controller-keystore-filename>
```

Restart the JVM afterwards. A self-signed or private-CA certificate must be trusted by the agent's truststore.

**Exam traps:** s1as / changeit / 8181. If HTTPS is enabled, the Controller **hostname must not start with a digit** — upgrades fail (JDK limitation).

## Block 2 — Reverse proxy and load balancer

- Agents and browsers use a **virtual IP** → move the Controller or switch the HA primary **without reconfiguring agents**. Adds a security layer (often in the DMZ).

- **SSL termination at the proxy** offloads SSL work from the Controller and keeps certificates in one place — recommended for large deployments.

- Set `-Dappdynamics.controller.ui.deeplink.url` to the external URL ending in /controller. If SSL ends at the proxy, also set `-Dappdynamics.controller.services.hostName` and `-Dappdynamics.controller.services.port` (usually 443). Edit via the EC Controller Settings page so changes persist.

- Proxy must send X-Forwarded-Proto: https and add Secure to cookies (X-CSRF-TOKEN isn't marked secure when SSL ends at the proxy).

- **.NET agents need a CA-signed certificate on the proxy** — self-signed won't work.

- Health check `/controller/rest/serverstatus`: **HTTP 200** on the active Controller, **HTTP 503 on the standby**.

## Block 3 — High availability (2.4)

Primary + secondary Controller, **active-passive**, MySQL replicating primary → secondary. The load balancer / VIP sends traffic to **one** Controller only — never round-robin.

### Prerequisite

- **Identical** servers: OS, CPU, RAM, disk
- **Linux only** for EC-managed HA
- **Passwordless SSH** between hosts (RSA key pair + authorized_keys on each)
- Both hostnames in each /etc/hosts; files before dns in /etc/nsswitch.conf
- Two IPs per server (main + dedicated replication link) — recommended
- **Same database root password** on both
- C compiler (gcc) for the service installer

### Setup

1. EC: add credentials and both hosts.

2. **Custom Install** with primary + **secondary** host (existing standalone Controller → **Add Secondary Controller** wizard).

3. `platform-admin.sh submit-job --service controller --job activate-ha-modules` — removes the EC from the failover path.

4. As root on both servers, from `<controller_home>/controller-ha`: `set_mysql_password_file.sh`, then `install-init.sh` (installs the appdcontroller and appdcontroller-db services).

### Running HA

|**Task**|**How**|
|---|---|
|Manual failover / failback|EC Controller page → HA Failover, or platform-admin.sh submit-job --service controller --job ha-failover --platform-name &lt;name&gt;|
|Automatic failover|The **Watchdog**. **Blocked** if the secondary database is down or replication is unhealthy|
|Restart primary without failover|Disable auto-failover first (EC Appserver Configurations → clear Enable Auto Failover, or --job stop-controller-watchdog); re-enable with start-controller-watchdog|
|Secondary &gt; 3 days behind|--job incremental-replication (no downtime)|
|Secondary &gt; 7 days behind|**Revive**: --job remove, then --job add-secondary — **restarts the primary (downtime)**|

## Block 4 — Self-monitoring (2.2)

- Bundled agent: `<controller_home>/appserver/jetty/appagent/javaagent.jar` — the **only** agent supported for self-monitoring.

- View: `/controller?enableAccounts=true` → account **system**, user **root**.

- Thread pools: Machine Agent reads JMX MBean domain `com.appdynamics.serverthreadpools`. JMX is on by default in Jetty; for a remote Machine Agent run `java -jar start.jar --add-to-start=jmx-remote` and restart.

- Disk latency: a SIM (Server Visibility) agent watches the data partition and alerts **above 3 ms**.

- HA best practice: agents on **both** Controllers report to a **separate monitoring Controller**.

## Block 5 — Backup

|**Level**|**What**|**Suggested frequency**|
|---|---|---|
|1 — Light|Controller install directory, excluding data (configuration)|Nightly|
|2 — Metadata|mysqldump-based script + keystores .appd.scskeystore and platform-admin/.appd.scs|Nightly|
|3 — Complete|Everything: metrics, snapshots, events, EC data|About weekly|

- **Cold:** stop app server and database; copy install dir, MySQL datadir and JRE dir.

- **Hot:** Controller keeps running; Percona XtraBackup copies the datadir (can fail on very busy Controllers).

- **With HA:** stop the **secondary's** database, cold-backup there, restart it.

- **Exam fact:** the Enterprise Console **does not back up MySQL data** — back up yourself before upgrades.

## Block 6 — Upgrades (2.6)

**Order:** Enterprise Console → Events Service → EUM Server → **Controller last**.

### Before upgrading:

1. Read release notes for every version in between.

2. Check hardware and profile — profiles only grow.

3. Check db/logs/database.log for **InnoDB: Error** → if found, stop and contact support.

4. Record app server customizations (EC may not keep domain.xml, db.cnf changes).

5. New EC install? Create .passwordfile with AS_ADMIN_PASSWORD=&lt;root password&gt;.

6. HTTPS → hostname must not start with a digit.

7. **Back up** (Block 5).

- **Single Controller not yet managed by the EC:** Custom Install → **Discover & Upgrade** (discovery learns profile, tenancy, database settings), then upgrade.

- **HA pair (Upgrade Wizard):** upgrade EC → **fail over** to secondary → upgrade old primary (now standby) → **verify metrics in the last 5 minutes** (else "Failover & Rebuild Secondary" to roll back) → upgrade the other Controller. One Controller serves agents throughout.

- **Agents:** since 20.2, language agents are backward-compatible with any Controller.

## Paper lab (~45 min)

1. **SSL:** write the keytool sequence to replace the self-signed certificate with a CA-signed one for ctrl.bank.local, and what to do on "Failed to establish chain from reply".

2. **HA design:** two Linux servers ctrl01 and ctrl02 behind VIP appd.bank.local — list prerequisites, setup commands and the load balancer health check.

3. **Maintenance:** OS patching needs the **primary** restarted without a failover — steps and commands.

4. **Upgrade plan:** EC, 3-node Events Service, EUM Server and an HA Controller pair — order, backups, Controller steps, verification, rollback.

## Day 4 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Which alias must the Controller's SSL key use in keystore.jks?**

- A. controller
- B. s1as
- C. reporting-instance
- D. appd-ssl

<details><summary>Show answer</summary>

**Answer: B** — s1as on port 8181 — don't rename it. reporting-instance is the Reporting Service.

</details>

**Q2. Import gives 'Failed to establish chain from reply'. Fix?**

- A. Regenerate CSR with 4096-bit key
- B. Import root + intermediate CA certs into cacerts.jks, then import again
- C. Rename alias to controller
- D. Change keystore password to changeit

<details><summary>Show answer</summary>

**Answer: B** — keytool can't build the chain; add the CA certs to cacerts.jks and retry.

</details>

**Q3. Behind a load balancer, what does /controller/rest/serverstatus return on the standby Controller?**

- A. HTTP 200 with available=true
- B. HTTP 503
- C. HTTP 401
- D. A redirect to the primary

<details><summary>Show answer</summary>

**Answer: B** — Active = 200, standby = 503, so the LB routes to the active node only.

</details>

**Q4. Why run activate-ha-modules?**

- A. To start MySQL replication
- B. To remove the Enterprise Console from the failover path
- C. To license the secondary
- D. To enable SSL between Controllers

<details><summary>Show answer</summary>

**Answer: B** — Failover then works without depending on the EC.

</details>

**Q5. Secondary database down for 10 days — what's required?**

- A. Nothing
- B. Incremental replication only
- C. Revive: remove, then add-secondary (restarts primary)
- D. Reinstall the EC

<details><summary>Show answer</summary>

**Answer: C** — More than 7 days behind → revive. Incremental replication is for more than 3 days.

</details>

**Q6. When is automatic failover blocked?**

- A. Secondary DB down or replication unhealthy
- B. More than 1,000 agents
- C. Controller uses HTTPS
- D. Events Service is a cluster

<details><summary>Show answer</summary>

**Answer: A** — The Watchdog won't fail over to an unhealthy secondary.

</details>

**Q7. Which Java Agent is supported for Controller self-monitoring?**

- A. Any version newer than the Controller
- B. Only the bundled appserver/jetty/appagent/javaagent.jar
- C. One in /opt/appdynamics/javaagent
- D. The .NET Agent

<details><summary>Show answer</summary>

**Answer: B** — Only the internal bundled agent is supported.

</details>

**Q8. What does the EC back up automatically before a Controller upgrade?**

- A. Full MySQL datadir
- B. Metadata tables only
- C. Nothing — the EC does not back up MySQL data
- D. Keystores only

<details><summary>Show answer</summary>

**Answer: C** — You must back up yourself.

</details>

**Q9. HA Upgrade Wizard — what happens after upgrading the EC and verifying credentials?**

- A. Both Controllers upgraded together
- B. Fail over to secondary, then upgrade the old primary (now standby)
- C. Uninstall the secondary
- D. Upgrade the agents

<details><summary>Show answer</summary>

**Answer: B** — Fail over, upgrade standby, verify metrics in last 5 minutes, then upgrade the other.

</details>

**Q10. SSL terminated at a reverse proxy — which agents need a CA-signed certificate on the proxy?**

- A. Machine Agents
- B. .NET agents
- C. Database Agents
- D. None

<details><summary>Show answer</summary>

**Answer: B** — .NET agents need a CA-signed certificate; self-signed won't work.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|Controller SSL|8181 · keystore.jks · alias s1as · password changeit · truststore cacerts.jks|
|Chain error|Import root/intermediate into cacerts.jks, retry|
|HTTPS only|start.ini ports → SSL port · ssl.enabled=true · deeplink https|
|Hostname rule|Must not start with a digit when HTTPS is used|
|Proxy settings|deeplink.url · services.hostName/port (443) · .NET needs CA-signed cert|
|serverstatus|Active 200 · standby 503|
|HA basics|Active-passive · identical Linux servers · passwordless SSH · same DB root password|
|HA jobs|activate-ha-modules · ha-failover · stop/start-controller-watchdog|
|Replication lag|&gt; 3 days incremental-replication · &gt; 7 days revive (primary restarts)|
|Self-monitoring|Bundled appagent only · system account · JMX serverthreadpools|
|Backup|Levels 1/2/3 · cold vs hot (Percona) · HA → back up the secondary · EC doesn't back up MySQL|
|Upgrade order|EC → Events Service → EUM Server → Controller|
|HA upgrade|Fail over → upgrade standby → verify 5 min of metrics → upgrade other|

## Sources

- [Controller SSL and Certificates](https://help.splunk.com/ja-jp/appdynamics-on-premises/secure-the-platform/コントローラsslと証明書)

- [Enable SSL between the Java Agent and the Controller](https://help.splunk.com/en/appdynamics-on-premises/application-performance-monitoring/26.9.0/install-app-server-agents/java-agent/administer-the-java-agent/enable-ssl-for-the-java-agent/enable-ssl-between-the-java-agent-and-the-controller)

- [Use a Reverse Proxy](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.3.0/controller-deployment/administer-the-controller/use-a-reverse-proxy)

- [Set Up a High Availability Deployment](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.5.0/controller-deployment/controller-high-availability/set-up-a-high-availability-deployment)

- [Manage a High Availability Deployment](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.8.0/controller-deployment/controller-high-availability/manage-a-high-availability-deployment)

- [Self-Monitoring the Controller](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.2.0/controller-deployment/self-monitoring-the-controller)

- [Best Practices for Backups](https://help.splunk.com/en/appdynamics-on-premises/plan-your-deployment/physical-machine-controller-deployment-guide/controller-data-and-backups/controller-data-backup-and-restore/best-practices-for-backups)

- [Back up the Existing Controller](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.6.0/controller-deployment/upgrade-the-controller-using-the-enterprise-console/before-upgrading/back-up-the-existing-controller)

- [Before Upgrading](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.6.0/controller-deployment/upgrade-the-controller-using-the-enterprise-console/before-upgrading)

- [Upgrade the HA Controller Pair Using the Upgrade Wizard](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.7.0/controller-deployment/upgrade-the-controller-using-the-enterprise-console/upgrade-an-ha-pair/upgrade-the-ha-controller-pair-using-the-upgrade-wizard)

- [Upgrade Platform Components](https://help.splunk.com/en/appdynamics-on-premises/upgrade-platform-components)

- [Disk I/O Requirements](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.7.0/controller-deployment/controller-system-requirements/disk-io-requirements)

## Paper lab — model answers

How an implementer would answer each Day 4 paper-lab task, with the rule behind each step.

### 1. Replace the Controller certificate for ctrl.bank.local

```
cd <controller_home>/appserver/jetty/etc
cp keystore.jks keystore.jks.bak
platform-admin.sh stop-controller-appserver
keytool -delete -alias s1as -keystore keystore.jks
keytool -genkeypair -alias s1as -keyalg RSA -keysize 2048 -validity 1825 \
        -keystore keystore.jks -ext SAN=dns:ctrl.bank.local
keytool -certreq -alias s1as -keystore keystore.jks -file AppDynamics.csr
#   → send AppDynamics.csr to the bank CA, receive mycert.cer + root/intermediate
keytool -import -trustcacerts -alias s1as -file mycert.cer -keystore keystore.jks
platform-admin.sh start-controller-appserver
```

### If "Failed to establish chain from reply" appears:

```
keytool -import -trustcacerts -alias bankroot -file root.cer -keystore cacerts.jks
keytool -import -trustcacerts -alias bankinter -file intermediate.cer -keystore cacerts.jks
keytool -import -trustcacerts -alias s1as -file mycert.cer -keystore keystore.jks   # retry
```

- Keep the alias **s1as**; any alias works for CA certs in cacerts.jks.

- Replace the default password **changeit**, keeping keystore.jks and cacerts.jks on the same password.

- Verify at https://ctrl.bank.local:8181/controller. Because the bank CA is private, agents must trust it (import the root into the agent truststore).

- GUI alternative: EC → Controller Settings → Appserver Configurations → SSL Certificate Management.

### 2. HA design for ctrl01 / ctrl02 behind appd.bank.local

**Prerequisites:** identical Linux servers (OS, CPU, RAM, disk); passwordless SSH both ways for the Controller user; both hostnames in each /etc/hosts with files before dns in nsswitch.conf; same database root password; optional second NIC for replication; gcc installed.

### Setup:

```
./platform-admin.sh add-credential --credential-name hakey --type ssh \
    --user-name appduser --ssh-key-file /home/appduser/.ssh/appd.pem
./platform-admin.sh add-hosts --hosts ctrl01 ctrl02 --credential hakey
# EC GUI → Custom Install: primary host ctrl01, secondary host ctrl02, profile, passwords
./platform-admin.sh submit-job --service controller --job activate-ha-modules
# as root on ctrl01 AND ctrl02, in <controller_home>/controller-ha:
./set_mysql_password_file.sh ...
./install-init.sh ...          # installs appdcontroller + appdcontroller-db services
```

- **Load balancer:** VIP appd.bank.local → pool ctrl01, ctrl02. Health check GET /controller/rest/serverstatus — healthy = HTTP 200 (active); the standby returns 503. Active-passive only, **no round-robin**.

- Agents use controller-host appd.bank.local (never ctrl01/ctrl02), so failover needs no agent change.

- If SSL ends on the LB: set deeplink.url to https://appd.bank.local/controller and services.hostName/port to appd.bank.local / 443.

- Add self-monitoring: agents on both Controllers report to a separate monitoring Controller.

### 3. Restart the primary for OS patching without failover

1. Disable auto-failover: EC → Appserver Configurations → clear **Enable Auto Failover** → Save, or `platform-admin.sh submit-job --job stop-controller-watchdog --service controller`.

2. On the primary: `bin/stopController.sh` (or `service appdcontroller stop`).

3. Patch and reboot the OS.

4. Start: `bin/startController.sh` (or `service appdcontroller start`); confirm serverstatus returns 200 through the VIP.

5. Re-enable auto-failover: `platform-admin.sh submit-job --job`

- `start-controller-watchdog --service controller` (or re-tick the box).

Why: with auto-failover on, the Watchdog fails over if the app server is down more than 5 minutes. Alternative that avoids downtime: planned ha-failover to the secondary, patch the old primary, then fail back.

### 4. Full platform upgrade plan

|**Step**|**Action**|
|---|---|
|0. Prepare|Read release notes for all versions in between; check OS/hardware support; check database.log for InnoDB: Error; record app server customizations; HTTPS hostname must not start with a digit|
|1. Back up|Controller: cold backup on the **secondary** (stop its DB, copy install dir, datadir, JRE, restart). Level 2 metadata (mysqldump + .appd.scskeystore, platform-admin/.appd.scs). EC data with mysqldump. The EC itself does not back up MySQL|
|2. Enterprise Console|Upgrade the EC first|
|3. Events Service|Upgrade the 3-node cluster from the EC Events Service page|
|4. EUM Server|Upgrade the EUM Server (back up its data first)|
|5. Controller HA pair|Upgrade Wizard: verify credentials → **fail over** to secondary → upgrade old primary (standby) → **verify**: log in, metrics in the last 5 minutes → upgrade the other Controller|
|Rollback|If verification fails: **Failover & Rebuild Secondary** — traffic returns to the not-yet-upgraded Controller|
|6. Post-checks|serverstatus 200 via VIP; agents reporting; EUM beacons arriving; Analytics searches return data; re-enable anything disabled|
|7. Agents|Upgrade later on their own schedule — agents are backward-compatible with any Controller (since 20.2)|

---

[← Day 3](day-03-controller-install-and-enterprise-console.md) · [🏠 Home](../README.md) · [Day 5 →](day-05-events-service-and-eum-server.md)
