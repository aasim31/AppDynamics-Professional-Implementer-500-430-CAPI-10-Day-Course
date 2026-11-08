# Day 3 — Controller Installation and the Enterprise Console

[🏠 Home](../README.md) · [← Day 2](day-02-pre-deployment-planning-and-sizing.md) · [Day 4 →](day-04-ssl-ha-self-monitoring-upgrades.md) · [📄 PDF version](../pdf/Day03_Controller_Install_and_Enterprise_Console.pdf)

Cisco AppDynamics Professional Implementer (500-430 CAPI) · 10-day course · Blueprint 2.1 and 4.3

**Goal:** install the Enterprise Console and a Controller three ways (GUI, CLI, silent), verify and operate it, and use the Administration Console.

**Verification:** every fact checked against the current Splunk AppDynamics on-prem docs (October 2026).

## Block 1 — The install roadmap

```
1. Prepare host  →  2. Install Enterprise Console  →  3. Create platform + add hosts
→  4. Install Controller (Express / Custom / CLI)  →  5. Verify  →  6. Secure (Day 4)
```

### Three accounts are set during install:

|**Account**|**Purpose**|
|---|---|
|Embedded MySQL root|The Controller's database|
|Controller root user|Built-in global administrator — used for the Administration Console|
|Controller administrator|Normal admin login to the Controller UI (default username **admin**)|

Usernames and passwords must **not** contain **&** or **!**.

## Block 2 — Prepare the Linux host

- Libraries: `libaio`, `numactl` (NUMA hosts), `tzdata` (Ubuntu), ncurses 5 compatibility, `net-tools`.

- User limits: `nofile` 65535 and `nproc` 8192, hard and soft.

- Install user: read/write/execute on the install directory, plus write access to `/etc/.java/.systemprefs`.

- Antivirus: exclude `<controller_home>/db/data`.

## Block 3 — Install the Enterprise Console

Installer: `platform-setup-64bit-linux.sh` (Windows: `platform-setup-64bit-windows.exe`). Run with GUI/console prompts, or silently:

```
./platform-setup-64bit-linux.sh -q -varfile ~/response.varfile
```

**response.varfile (Linux)** — values from the docs:

```
serverHostName=ec01.bank.local
sys.languageId=en
disableEULA=true
sys.installationDir=/opt/appdynamics/platform
platformAdmin.port=9191
platformAdmin.databasePort=3377
platformAdmin.adminPassword=<pwd>
platformAdmin.databasePassword=<pwd>
platformAdmin.databaseRootPassword=<pwd>
platformAdmin.useHttps$Boolean=false
```

- Always use an **absolute** installation path. On AWS, serverHostName = the **public DNS** name.

- The EC has its **own** MySQL on port **3377**; the Controller's database uses **3388**.

- Open the EC at `http://<ec-host>:9191`. Log: `<platform_admin_home>/logs/platform-admin-server.log`.

## Block 4 — Install the Controller

||**Express Install**|**Custom Install**|
|---|---|---|
|Installs|Single-node Controller + **embedded** Events Service on one host|Whatever you choose|
|Version|Latest only|You pick|
|HA pair|No|Yes — primary + optional secondary host|
|Tenancy mode|Default (single)|Choosable|
|Events Service|Embedded|Separate: 1 node or 3 or more|
|Use for|Labs, quick start|Anything real|

- SSH keys must be **PEM format** (convert newer OpenSSH / Ed25519 keys).

- The EC **can't manage remote Windows hosts** — a Windows Controller needs the EC on the same machine.

- The **Controller install is the longest job**. Default Java is **Azul Zulu** (Azul Prime optional).

- Platform path must contain no existing AppDynamics components. From the CLI, create the platform **first**.

### CLI install (run in &lt;EC&gt;/platform-admin):

```
./platform-admin.sh create-platform --name bank --installation-dir /opt/appdynamics/product
./platform-admin.sh add-credential --credential-name ctrlkey --type ssh \
    --user-name appduser --ssh-key-file /home/appduser/.ssh/id_rsa
./platform-admin.sh add-hosts --hosts ctrl01 --credential ctrlkey
./platform-admin.sh submit-job --service controller --job install --args \
    controllerPrimaryHost=ctrl01 controllerAdminUsername=admin \
    controllerAdminPassword=<pwd> controllerRootUserPassword=<rootpwd> \
    mysqlRootPassword=<dbrootpwd> controllerProfile=demo
./platform-admin.sh list-job-parameters --job install --service controller
```

**"Install the Controller manually" on the blueprint:** today the Controller is always installed through the EC; "manually" means Custom Install or the CLI rather than Express. Prep-guide sample answer — the two Linux installation modes are **silent mode and the GUI installer**.

## Block 5 — Verify and operate

### Verify (as documented):

1. Open `http://<host>:8090/controller/rest/serverstatus` → `<available>true</available>`.

2. Log in as the **initial Controller administrator** (admin).

3. Optional: `bin/platform-admin.sh check-controller-health`.

**Start / stop (standalone Linux)** — database first, then app server; stop in reverse:

```
platform-admin.sh start-controller-db --platform-name bank
platform-admin.sh start-controller-appserver --platform-name bank
platform-admin.sh stop-controller-appserver
platform-admin.sh stop-controller-db
```

- **Windows:** services _AppDynamics Controller Application Server_ and _AppDynamics Database_ . Starting the app server starts the database; stopping the app server does **not** stop the database.

- **HA pair:** `bin/startController.sh` / `bin/stopController.sh` or `service appdcontroller` / `appdcontroller-db`. **Disable auto-failover before restarting the primary** — otherwise the **Watchdog** fails over if the app server is down more than **5 minutes**.

- The EC waits up to **45 minutes** for a start/stop job.

|**Log**|**Location**|
|---|---|
|Controller app server|&lt;controller_home&gt;/logs/server.log|
|Controller database|&lt;controller_home&gt;/db/logs/database.log (check for InnoDB: Error before upgrades)|
|Enterprise Console|&lt;platform_admin_home&gt;/logs/platform-admin-server.log|

## Block 6 — Default ports (official table)

|**Port**|**Component**|
|---|---|
|8090|Controller app server HTTP|
|8181|Controller app server HTTPS|
|4848|Controller app server admin|
|3388|Controller database (MySQL)|
|3377|Enterprise Console database|
|9191|Enterprise Console|
|8020 / 8021|Reporting Service HTTP / HTTPS|
|8030|Fleet Management Service|
|9080 / 9081|Events Service REST API / admin|
|7001 / 7002|EUM Server HTTP / HTTPS|

## Block 7 — The Administration Console (4.3)

- URL: `http://<host>:8090/controller/admin.jsp` — log in as **root** with the root user password.

- Already logged in as another user → **Access Denied**. Log out first or use a private window.

- **Controller Settings:** platform-wide properties (EUM and Events Service URLs/keys, metric retention).

- **Account Settings:** accounts, licenses, access keys.

- Single-tenant default account: **customer1**. REST login: `<user>@customer1:<password>`.

- Self-monitoring: `/controller?enableAccounts=true` → account **system**, user **root**.

|**Property**|**Data**|**Default (on-prem)**|
|---|---|---|
|metrics.min.retention.period|1-minute|**4 hours**|
|metrics.ten.min.retention.period|10-minute|**48 hours**|
|metrics.retention.period|1-hour|**365 days**|

Each value must be larger than the one above it. Longer retention grows the database quickly. (SaaS keeps 1-minute data up to 8 days — don't mix them up.)

## Paper lab (~40 min)

1. **Response file:** install the EC silently at /opt/appdynamics/platform on ec01.bank.local, GUI over HTTP on port 9191.

2. **CLI commands:** install a **small** profile Controller on remote host ctrl01 with the key /home/appduser/.ssh/appd.pem.

3. **Troubleshoot:** after install, serverstatus doesn't respond — list the first three checks and the log files you'd open.

4. **Admin task:** keep 1-minute data for 8 hours — which property, where, and what risk?

## Day 3 Quiz — with answer key

Try each question first, then open **Show answer** to check.

**Q1. Which command installs the Enterprise Console silently on Linux?**

- A. ./platform-setup-64bit-linux.sh -q -varfile ~/response.varfile
- B. ./platform-admin.sh install --silent
- C. ./controller.sh install -q
- D. ./platform-setup-64bit-linux.sh --express

<details><summary>Show answer</summary>

**Answer: A** — -q plus a response file passed with -varfile; use an absolute install path.

</details>

**Q2. What does Express Install deploy?**

- A. An HA pair and a 3-node Events Service cluster
- B. A single-node Controller and an embedded Events Service on one host
- C. Controller, Events Service and EUM Server
- D. A Controller of any version you choose

- **Answer: B** — Latest version only; Custom Install gives choices. The EC never installs the EUM Server.

**Q3. Standalone Linux Controller — correct start order?**

- A. App server, then database
- B. Database, then app server
- C. Both with start-controller-all
- D. Reporting Service, then database

<details><summary>Show answer</summary>

**Answer: B** — start-controller-db, then start-controller-appserver; stop in reverse.

</details>

**Q4. Default port of the Enterprise Console's own database?**

- A. 3388
- B. 3377
- C. 4848
- D. 9191

<details><summary>Show answer</summary>

**Answer: B** — EC database 3377; Controller database 3388; 4848 app server admin; 9191 EC UI.

</details>

**Q5. Logged in as admin, a user opens admin.jsp in the same browser and gets Access Denied. Fix?**

- A. Restart the Controller
- B. Log out (or private window), then log in as root
- C. Give admin the Account Owner role
- D. Enable HTTPS on 8181

<details><summary>Show answer</summary>

**Answer: B** — The Administration Console needs the root user; log out first.

</details>

**Q6. Default retention for 1-minute metric data on an on-prem Controller?**

- A. 4 hours
- B. 48 hours
- C. 8 days
- D. 365 days

<details><summary>Show answer</summary>

**Answer: A** — metrics.min.retention.period = 4 hours on-prem (8 days is SaaS).

</details>

**Q7. Which two documented steps verify a new Controller installation?**

- A. Open /controller/rest/serverstatus and log in as the initial administrator
- B. mysql -u root and ping 9191
- C. Check license page and restart EC
- D. Install a Java agent and wait

<details><summary>Show answer</summary>

**Answer: A** — Browse to serverstatus, then log in with the initial administrator.

</details>

**Q8. Before restarting the primary Controller of an HA pair, you must:**

- A. Stop the secondary's database
- B. Disable auto-failover, then re-enable it afterwards
- C. Uninstall the HA modules
- D. Switch to multi-tenant

<details><summary>Show answer</summary>

**Answer: B** — Otherwise the Watchdog fails over if the app server is down more than 5 minutes.

</details>

**Q9. Basic-auth REST username format for a single-tenant on-prem Controller?**

- A. admin:password
- B. admin@customer1:password
- C. root@system:password
- D. customer1\admin:password

<details><summary>Show answer</summary>

**Answer: B** — Default account is customer1: &lt;user&gt;@customer1:&lt;password&gt;.

</details>

**Q10. EC on Linux, Controller wanted on a remote Windows server — result?**

- A. Works with WinRM
- B. Works with a PEM key
- C. Not supported: the EC must run on the same Windows machine
- D. Works for Large only

<details><summary>Show answer</summary>

**Answer: C** — The EC doesn't manage remote Windows hosts.

</details>

## Quick-review cheat sheet

|**Remember**|**Value**|
|---|---|
|EC installer|platform-setup-64bit-linux.sh · silent: -q -varfile response.varfile|
|EC ports|UI 9191 · own database 3377|
|Accounts at install|MySQL root · Controller root user · Controller admin (default 'admin'); no & or !|
|Express vs Custom|Express = 1 host, embedded ES, latest only · Custom = versions, HA, separate ES|
|EC limits|PEM keys · no remote Windows hosts · Controller install is the longest job|
|Verify|/controller/rest/serverstatus + log in as initial admin · check-controller-health|
|Start / stop order|DB then app server · stop app server then DB|
|HA restart|Disable auto-failover first · Watchdog fails over after 5 min|
|Logs|logs/server.log · db/logs/database.log · platform-admin-server.log|
|Admin Console|/controller/admin.jsp · root user · private window if Access Denied|
|Retention defaults (on-prem)|1-min 4 h · 10-min 48 h · 1-hour 365 days|
|Default account|customer1 → user@customer1:password|

## Sources

- [Silent Installation (Enterprise Console)](https://help.splunk.com/en/appdynamics-on-premises/enterprise-console/26.9.0/install-the-enterprise-console/silent-installation)

- [Custom Install](https://help.splunk.com/en/appdynamics-on-premises/enterprise-console/26.9.0/custom-install)

- [Install the Controller Using the Command Line](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.7.0/controller-deployment/install-the-controller-using-the-cli/install-the-controller-using-the-command-line)

- [Admin Accounts Created at Installation](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.1.0/controller-deployment/admin-accounts-created-at-installation)

- [Start or Stop the Controller](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.9.0/controller-deployment/administer-the-controller/start-or-stop-the-controller)

- [Platform Component Ports](https://help.splunk.com/en/appdynamics-on-premises/plan-your-deployment/port-settings/platform-component-ports)

- [Platform Log Files](https://help.splunk.com/en/appdynamics-on-premises/enterprise-console/26.7.0/administer-the-enterprise-console/platform-log-files)

- [Modify Metric Data Retention Periods](https://help.splunk.com/en/appdynamics-on-premises/plan-your-deployment/physical-machine-controller-deployment-guide/controller-data-and-backups/database-size-and-data-retention/modify-metric-data-retention-periods)

- [Using the Controller APIs](https://help.splunk.com/en/appdynamics-on-premises/extend-appdynamics/26.3.0/extend-splunk-appdynamics/splunk-appdynamics-apis/using-the-controller-apis)

- [Access the Administration Console](https://help.splunk.com/en/appdynamics-on-premises/controller-deployment/26.2.0/controller-deployment/administer-the-controller/access-the-administration-console)

## Paper lab — model answers

How an implementer would answer each Day 3 paper-lab task, with the rule behind each step.

### 1. Silent Enterprise Console install on ec01.bank.local

**response.varfile** (absolute paths; HTTP GUI on 9191):

```
serverHostName=ec01.bank.local
sys.languageId=en
disableEULA=true
sys.installationDir=/opt/appdynamics/platform
platformAdmin.port=9191
platformAdmin.databasePort=3377
platformAdmin.adminPassword=<pwd>
platformAdmin.databasePassword=<pwd>
platformAdmin.databaseRootPassword=<pwd>
platformAdmin.useHttps$Boolean=false
```

### Command:

```
./platform-setup-64bit-linux.sh -q -varfile /home/appduser/response.varfile
```

- `platformAdmin.useHttps$Boolean=false` gives the GUI over HTTP; `platformAdmin.port=9191` sets its port.

- 3377 is the EC's own database port — not the Controller's 3388.

- Check: open `http://ec01.bank.local:9191`; if it fails, read `<platform_admin_home>/logs/platform-admin-server.log`.

### 2. CLI install of a small Controller on remote host ctrl01

Run in &lt;EC&gt;/platform-admin. From the CLI the platform must be created **first**:

```
./platform-admin.sh create-platform --name bank --installation-dir /opt/appdynamics/product
./platform-admin.sh add-credential --credential-name ctrlkey --type ssh \
    --user-name appduser --ssh-key-file /home/appduser/.ssh/appd.pem
./platform-admin.sh add-hosts --hosts ctrl01 --credential ctrlkey
./platform-admin.sh submit-job --platform-name bank --service controller \
    --job install --args \
    controllerPrimaryHost=ctrl01 controllerAdminUsername=admin \
    controllerAdminPassword=<pwd> controllerRootUserPassword=<rootpwd> \
    mysqlRootPassword=<dbrootpwd> controllerProfile=small
```

- `appd.pem` is the **private** key and must be in **PEM format** (convert newer OpenSSH / Ed25519 keys).

- ctrl01 must be **Linux** — the EC does not manage remote Windows hosts.

- Not sure of a parameter? `./platform-admin.sh list-job-parameters --job install --service controller` lists them all.

- Expect the Controller install to be the longest job; watch it on the EC Jobs page.

### 3. serverstatus doesn't respond after install

### First three checks, in order:

1. **Did the install job succeed?** Check the EC Jobs page or `platform-admin-server.log`. A failed job means the Controller was never fully installed.

2. **Are the database and app server running?** Run `bin/platform-admin.sh check-controller-health`; confirm ports with `ss -tlnp | grep -E "3388|8090"`. If down, start the **database first**, then the app server.

3. **Is it reachable?** Test from the Controller itself (`curl http://localhost:8090/controller/rest/serverstatus`). If localhost works but remote doesn't, it's the firewall or network (open 8090/8181).

|**Log to open**|**What it tells you**|
|---|---|
|&lt;platform_admin_home&gt;/logs/platform-admin-server.log|Why an EC job (install/start) failed|
|&lt;controller_home&gt;/logs/server.log|App server start-up errors|
|&lt;controller_home&gt;/db/logs/database.log|MySQL errors (e.g. InnoDB: Error)|

Also re-check host preparation: user limits (nofile 65535, nproc 8192), libaio, free disk space.

### 4. Keep 1-minute data for 8 hours

- **Property:** `metrics.min.retention.period` — change from the default 4 hours to 8 hours.

- **Where:** Administration Console (/controller/admin.jsp, log in as root) → **Controller Settings** → edit → **Save**.

- **Rule:** it must stay below `metrics.ten.min.retention.period` (default 48 hours) — 8 h is fine.

- **Risk to mention:** longer retention makes the database grow quickly — confirm disk capacity first. If 1-minute + 10-minute retention exceed `caches.retention.period`, the UI slows because data comes from the database instead of cache.

- Note: changing retention does not change the time-range options shown in the Controller UI.

---

[← Day 2](day-02-pre-deployment-planning-and-sizing.md) · [🏠 Home](../README.md) · [Day 4 →](day-04-ssl-ha-self-monitoring-upgrades.md)
