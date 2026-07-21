# Splunk Log Management Stack

## Overview

This project is a self-contained log management stack built with Docker
Compose: a sample Apache web application generates access/error logs, a
Splunk Universal Forwarder tails those logs and ships them to a Splunk
indexer, and Splunk Web is used to search the ingested data and view a
prebuilt dashboard. A Jenkins pipeline automates validating, deploying, and
health-checking the stack so it can be redeployed on a target VM with one
click instead of by hand. The goal is a minimal but realistic example of the
log source -> forwarder -> indexer -> dashboard pipeline that underlies most
production Splunk deployments.

## Architecture

The stack is defined in `compose/docker-compose.yml` as three services on a
single Docker bridge network, `logging-net`, so they can resolve each other
by service/container name:

- **`sample-app`** (`httpd:latest`) — a plain Apache server bound to
  `127.0.0.1:80` on the host (not exposed to the internet). Its startup
  command patches `conf/httpd.conf` so `CustomLog`/`ErrorLog` write to real
  files (`logs/access_log`, `logs/error_log`) in `combined` format, instead
  of the image's default of logging to stdout/stderr. Those files are
  written into the named volume `apache-logs`, mounted at
  `/usr/local/apache2/logs`.
- **`forwarder`** (`splunk/universalforwarder:latest`) — mounts the same
  `apache-logs` volume read-only at `/var/log/apache2` and tails
  `access_log`/`error_log` per `splunk-config/inputs.conf`. It bind-mounts
  the whole `../splunk-config` directory (read-write — required by the
  forwarder's own provisioning process) to
  `/opt/splunkforwarder/etc/system/local`, so `inputs.conf`,
  `outputs.conf`, and `props.conf` are version-controlled rather than baked
  into the image. It forwards events to the indexer over `splunk:9997`.
- **`splunk`** (`splunk/splunk:latest`) — the indexer/search head. It
  receives forwarded data on port `9997`, indexes it into the `webapp_logs`
  index on the named volume `splunk-var` (`/opt/splunk/var`), and serves
  Splunk Web on port `8000`. The `webapp_logs` index is defined in
  `splunk-apps/webapp_alerts/default/indexes.conf` (bind-mounted into
  `etc/apps/`), **not** created by hand in the UI — `/opt/splunk/etc` is
  not volume-backed, so a UI-created index definition is silently lost
  whenever the container is recreated (see
  `docs/troubleshooting-container-recreation.md`).

**Data flow:** `sample-app` writes Apache logs to the `apache-logs` volume
-> `forwarder` monitors those files and forwards events over TCP `9997`
(S2S protocol, compressed) -> `splunk` indexes them into `webapp_logs` with
sourcetypes `access_combined` (access log) and `httpd_error` (error log) ->
you search/view them in Splunk Web (port `8000`) via the
`webapp-logs-overview` dashboard.

**Volumes:**
- `apache-logs` — shared between `sample-app` (read-write) and `forwarder`
  (read-only); carries the raw log files.
- `splunk-var` — Splunk's persistent index/config storage on the `splunk`
  container.

**Secrets:** the Splunk admin password is supplied via `SPLUNK_PASSWORD` in
`compose/.env` (gitignored, never committed — see `compose/.env.example`
for the required variable). In the Jenkins pipeline this is instead
injected from a Jenkins credential, since a fresh checkout never has
`.env`.

## Prerequisites

- An AWS account with an EC2 instance (Ubuntu recommended) to deploy on.
  Splunk is memory-hungry — use at least a `t3.medium` (4 GB RAM) or
  larger.
- Docker Engine installed on that instance.
- The Docker Compose v2 CLI plugin (`docker compose ...`, not the standalone
  `docker-compose`).
- Security group / firewall rules on the instance allowing inbound access
  to port `8000` (Splunk Web) from your IP, if you want to reach the UI
  from your own machine rather than only via SSH tunnel.

## Setup instructions

1. **Clone the repo** onto the EC2 instance:
   ```bash
   git clone <this-repo-url>
   cd final-project
   ```
2. **Configure the environment file.** Copy the example and set a real
   password (Splunk enforces a minimum password strength):
   ```bash
   cp compose/.env.example compose/.env
   # edit compose/.env and set SPLUNK_PASSWORD to something Splunk will accept
   ```
   `compose/.env` is gitignored — never commit it.
3. **Start the stack:**
   ```bash
   cd compose
   docker compose up -d
   ```
   This builds/pulls all three images and starts `splunk`, `forwarder`, and
   `sample-app` on the `logging-net` bridge network.
4. **Generate some traffic** so there's something to see in the dashboard:
   ```bash
   curl http://127.0.0.1/
   curl http://127.0.0.1/does-not-exist   # generates a 404 for the error panels
   ```
5. **Log into Splunk Web** at `http://<ec2-public-ip>:8000` (or
   `http://localhost:8000` if tunneled over SSH) with user `admin` and the
   `SPLUNK_PASSWORD` you set in `compose/.env`.
6. **Verify ingestion** by searching `index=webapp_logs` in Splunk Web —
   you should see events with sourcetype `access_combined` and
   `httpd_error`.

If the forwarder isn't shipping data, see
`docs/troubleshooting-log-ingestion.md`.

## Jenkins pipeline

`jenkins/Jenkinsfile` defines a declarative pipeline (`splunk-stack-deploy`)
that automates redeploying the stack:

1. **Checkout** — pulls the repo via `checkout scm`.
2. **Validate** — runs `docker compose -f compose/docker-compose.yml
   config` to sanity-check the compose file before touching anything
   running.
3. **Deploy** — runs `docker compose -f compose/docker-compose.yml up -d
   --build` to (re)create the stack. `SPLUNK_PASSWORD` is injected from the
   Jenkins credential `SPLUNK_PASSWORD` (a `Secret text` credential),
   since `compose/.env` doesn't exist in a fresh Jenkins checkout and a
   Jenkins-injected env var takes precedence over any `.env` file anyway.
4. **Health Check** — polls `http://host.docker.internal:8000` with
   `curl -s -L` (following the `303` redirect Splunk Web's root URL
   returns) every 10 seconds for up to 120 seconds, until it gets HTTP 200.

**To trigger it:** run the `splunk-stack-deploy` job from the Jenkins UI
(manually, or via a configured SCM webhook/poll trigger on the `main`
branch). Jenkins must be set up with:
- A `Secret text` credential with ID `SPLUNK_PASSWORD` holding the Splunk
  admin password.
- Access to the host's Docker socket and the `docker` CLI's Compose v2
  plugin, and network reachability to `host.docker.internal` for the
  health check.

See `docs/troubleshooting-jenkins-pipeline.md` for the exact Jenkins
container/job configuration this depends on (not tracked in this repo,
since it lives in the Jenkins container/job setup rather than in files
here) and the bugs that came up standing it up.

## Dashboards & Alerts

`docs/dashboards/webapp-logs-overview.xml` is the source for the
**Webapp Logs Overview** dashboard, built against `index=webapp_logs`. It
has a time-range picker and a URI filter, and shows:

- Single-value panels: total requests, error rate (4xx/5xx as a % of
  requests), unique client IPs, and total Apache error-log events.
- Requests over time by HTTP status class (2xx/3xx/4xx/5xx), stacked.
- HTTP status code distribution (pie chart).
- Top 10 requested URIs and top 10 client IPs, with per-row error counts.
- Apache error-log events over time by severity, plus a table of the 25
  most recent errors.

**To view it:** in Splunk Web, go to **Dashboards** -> **Create New
Dashboard** (or **Import Dashboard**, depending on your Splunk version)
and paste in the contents of `docs/dashboards/webapp-logs-overview.xml`,
or use the Splunk REST API / CLI to create it programmatically from that
file. Once saved, it appears under **Dashboards** in the app you created
it in.

One scheduled alert is version-controlled in
`splunk-apps/webapp_alerts/default/savedsearches.conf`, a small Splunk app
that `docker-compose.yml` bind-mounts into the indexer at
`/opt/splunk/etc/apps/webapp_alerts` (alerts run on the indexer/search
head, not the forwarder, so it can't live in `splunk-config/`):

- **Webapp High Error Rate (4xx/5xx)** — runs every 5 minutes over the
  last 15 minutes and fires when 4xx/5xx responses exceed 15% of requests
  (the dashboard's red threshold), with a 10-request minimum so a single
  404 against near-zero traffic doesn't trigger it. Firings are recorded
  under **Activity** -> **Triggered Alerts** (`alert.track = 1`) and
  throttled to once per 30 minutes. No email/webhook action is configured
  because the stack has no SMTP server; add `action.email.*` settings (or
  edit the alert in Splunk Web) to attach one.

**To test it:** generate enough failing traffic to cross the threshold,
e.g. `for i in $(seq 1 20); do curl -s http://127.0.0.1/does-not-exist
>/dev/null; done`, wait for the next 5-minute schedule tick, and check
**Activity** -> **Triggered Alerts** in Splunk Web.

## Repository structure

```
.
├── compose/
│   ├── docker-compose.yml        # the three services, network, and volumes described above
│   ├── .env.example               # template for the required SPLUNK_PASSWORD variable
│   └── .env                       # gitignored — real secret, created locally from .env.example
├── jenkins/
│   └── Jenkinsfile                # Checkout -> Validate -> Deploy -> Health Check pipeline
├── splunk-apps/
│   └── webapp_alerts/             # Splunk app mounted into the indexer's etc/apps/
│       ├── default/
│       │   ├── savedsearches.conf # the "Webapp High Error Rate" scheduled alert
│       │   ├── indexes.conf       # webapp_logs index definition (survives container recreation)
│       │   └── app.conf           # app metadata (hidden from launcher, enabled)
│       └── metadata/
│           └── default.meta       # exports the alert globally (visible outside the app)
├── splunk-config/
│   ├── inputs.conf                # forwarder: which log files to monitor, into which index/sourcetype
│   ├── outputs.conf                # forwarder: where to ship data (splunk:9997)
│   ├── props.conf                  # sourcetype definitions (access_combined, httpd_error)
│   ├── server.conf                 # indexer-side runtime config (generated by Splunk on startup)
│   ├── web.conf                    # Splunk Web runtime config (generated by Splunk on startup)
│   └── migration.conf              # Splunk internal migration-state file (generated by Splunk)
├── docs/
│   ├── dashboards/
│   │   └── webapp-logs-overview.xml   # source XML for the Webapp Logs Overview dashboard
│   ├── troubleshooting-log-ingestion.md      # postmortem: why index=webapp_logs was empty
│   ├── troubleshooting-jenkins-pipeline.md   # postmortem: why the Jenkins pipeline kept failing
│   └── troubleshooting-container-recreation.md  # postmortem: index definition + splunk.secret lost on recreate
└── README.md
```
