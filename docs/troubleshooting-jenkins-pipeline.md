# Troubleshooting: `jenkins/Jenkinsfile` Pipeline Failures

## Symptom

The `splunk-stack-deploy` Pipeline job (`Pipeline script from SCM`, script
path `jenkins/Jenkinsfile`) failed on every run, with a different error each
time a prior one was fixed. There were five separate, stacked bugs across
two layers: the Jenkins job/container configuration (not tracked in this
repo), and the `jenkins/Jenkinsfile` itself.

---

## Bug 1: Invalid branch specifier `refs/heads/**`

**Cause:** The job's SCM configuration had **Branch Specifier** set to
`refs/heads/**`, which is multibranch-job wildcard syntax. A plain
Pipeline job's single Git checkout expects one concrete ref and throws:

```
java.lang.IllegalArgumentException: Invalid refspec refs/heads/**
```

**Fix:** Changed the job's Branch Specifier to `*/main`. (Job
configuration, not a repo file.)

---

## Bug 2: `docker compose` — "unknown shorthand flag: 'f' in -f"

**Cause:** Jenkins runs as the official `jenkins/jenkins:lts` container. The
`docker` CLI binary and the Docker socket were bind-mounted in so `docker`
itself worked, but the Compose v2 CLI plugin directory was not, so `compose`
wasn't recognized as a subcommand — the CLI tried to parse `-f` as a flag to
plain `docker` instead and errored.

**Fix:** Recreated the Jenkins container with the host's plugin directory
mounted in, alongside the existing `docker` binary/socket mounts:

```bash
-v /usr/libexec/docker/cli-plugins:/usr/libexec/docker/cli-plugins
```

(Jenkins container run command, not a repo file.)

---

## Bug 3: "permission denied while trying to connect to the docker API"

**Cause:** The Deploy stage's `docker compose up` failed because the
container's default `jenkins` user (uid 1000) could reach `/var/run/docker.sock`
via the bind mount, but wasn't in the socket's owning group. On the host,
the socket is `root:docker` with GID `986`.

**Fix:** Recreated the Jenkins container again, adding:

```bash
--group-add 986
```

so `jenkins` gets supplementary access to the `docker` group's GID without
running as root. (Jenkins container run command, not a repo file.)

---

## Bug 4: Splunk crash-looping — `SPLUNK_PASSWORD` blank in the Deploy stage

**Cause:** `compose/.env` is gitignored (it holds the real Splunk admin
password) and therefore never exists in a fresh Jenkins `git checkout`.
`docker compose up` fell back to an empty string for `${SPLUNK_PASSWORD}`,
and Splunk's Ansible-driven provisioning refuses to start without one:

```
Exception: Splunk password must be supplied!
```

This also **overwrote the previously-healthy `splunk`/`forwarder`
containers** with the broken, password-less config, since `docker compose
up` on the same project recreates existing containers to match the new
config.

**Fix:**
1. Restored the stack by re-running `docker compose up -d --build` from a
   checkout that had the real `.env` in place.
2. Added a Jenkins credential (`Secret text`, ID `SPLUNK_PASSWORD`) holding
   the same value as `compose/.env`.
3. Updated `jenkins/Jenkinsfile` to inject it as a real environment
   variable, which takes precedence over any `.env` file value:
   ```groovy
   environment {
       SPLUNK_PASSWORD = credentials('SPLUNK_PASSWORD')
   }
   ```

---

## Bug 5: Health Check always timed out even though Splunk was healthy

Two independent problems in the Health Check stage, found together once
Bugs 1–4 were fixed and the stage could finally run against a working stack:

**5a. Networking:** Jenkins reaches Docker via the host's mounted socket —
it's a sibling container, not the host, and not Docker-in-Docker. `curl
http://localhost:8000` from inside the Jenkins container hit Jenkins' own
loopback interface, not the host's published port, and always failed to
connect.

**Fix:** Recreated the Jenkins container with a host-gateway alias:
```bash
--add-host=host.docker.internal:host-gateway
```
and changed `HEALTHCHECK_URL` in the Jenkinsfile from `localhost` to
`host.docker.internal`.

**5b. Redirect handling:** Splunk Web's root URL (`/`) responds `303 See
Other` (redirecting to `/en-US/`), not a bare `200` — even when Splunk is
fully healthy. The health check's `curl` did not follow redirects, so it
would retry for the full timeout and fail no matter how long it waited.

**Fix:** Added `-L` to the curl invocation so it follows the redirect to
the page that actually returns `200`:
```bash
curl -s -L -o /dev/null -w "%{http_code}" "${HEALTHCHECK_URL}"
```

---

## Net result

Jenkins container run command changes (not tracked in this repo — document
here for anyone who needs to recreate the container):

| Flag | Purpose |
|---|---|
| `-v /usr/libexec/docker/cli-plugins:/usr/libexec/docker/cli-plugins` | Gives the container's `docker` CLI the `compose` subcommand (Bug 2) |
| `--group-add 986` | Lets the `jenkins` user access the host's `docker.sock` without running as root (Bug 3) |
| `--add-host=host.docker.internal:host-gateway` | Lets the Jenkins container reach ports published on the host (Bug 5a) |

Job configuration change (not tracked in this repo):
- Branch Specifier: `refs/heads/**` → `*/main` (Bug 1)
- Added credential: `Secret text`, ID `SPLUNK_PASSWORD` (Bug 4)

`jenkins/Jenkinsfile` changes:

| Before | After | Bug |
|---|---|---|
| `HEALTHCHECK_URL = "http://localhost:${SPLUNK_WEB_PORT}"` | `HEALTHCHECK_URL = "http://host.docker.internal:${SPLUNK_WEB_PORT}"` | 5a |
| No `SPLUNK_PASSWORD` binding | `SPLUNK_PASSWORD = credentials('SPLUNK_PASSWORD')` | 4 |
| `curl -s -o /dev/null -w "%{http_code}"` | `curl -s -L -o /dev/null -w "%{http_code}"` | 5b |

With all fixes in place, the pipeline runs Checkout → Validate → Deploy →
Health Check cleanly end to end, and `post { success }` reports a pass.

## How it was diagnosed

1. First run failed at Checkout SCM with `Invalid refspec refs/heads/**` —
   pointed directly at the job's Branch Specifier field (Bug 1).
2. Second run got past Checkout but failed at Validate with `unknown
   shorthand flag: 'f' in -f`; `docker exec`-ing into the Jenkins container
   and running `docker compose version` directly reproduced the same
   failure and confirmed the plugin was missing, while the host had it at
   `/usr/libexec/docker/cli-plugins/docker-compose` (Bug 2).
3. Third run got past Validate but failed at Deploy with a Docker API
   permission error; comparing the socket's group ownership (`docker`,
   GID 986) against the container's user (`id` showed only `jenkins`/1000)
   confirmed the missing group membership (Bug 3).
4. Fourth run got past Deploy but the Health Check stage timed out;
   `docker logs splunk` showed the Ansible inventory script raising
   `Splunk password must be supplied!`, and `ls` on the Jenkins workspace
   confirmed `compose/.env` was absent (only `.env.example`) — explaining
   the blank password (Bug 4).
5. After adding the credential binding, a manual `curl` from inside the
   Jenkins container to `http://localhost:8000` returned `000` (connection
   failure) even though `docker ps` showed Splunk healthy on the host —
   pointing at container network isolation rather than an application
   problem (Bug 5a).
6. Switching the test curl to `host.docker.internal` connected, but
   returned `303`; `curl -sI` showed a `Location` redirect to `/en-US/`,
   and `curl -L` confirmed that page returns `200` — explaining why the
   health check's exact-`200` match never matched without following
   redirects (Bug 5b).
