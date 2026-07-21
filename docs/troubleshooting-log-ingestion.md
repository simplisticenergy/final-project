# Troubleshooting: No Events in `index=webapp_logs`

## Symptom

Searching `index=webapp_logs` in Splunk Web returned zero results, even though
`docker compose ps` showed all three containers (`splunk`, `forwarder`,
`sample-app`) running and healthy, and `curl` requests to the sample app
returned HTTP 200.

There were three separate, stacked bugs. Fixing the first one only exposed
the second, and fixing the second only exposed the third.

---

## Bug 1: Apache was logging to stdout/stderr, not to files

**Cause:** The official `httpd:latest` image ships a default `httpd.conf`
that points its log directives at the container's stdout/stderr file
descriptors instead of real files:

```apache
ErrorLog /proc/self/fd/2
CustomLog /proc/self/fd/1 common
```

This is a common Docker convention (`docker logs sample-app` shows the
traffic), but it means nothing was ever written to
`/usr/local/apache2/logs/access_log` or `error_log` — the files the
forwarder's `inputs.conf` was configured to monitor. The shared
`apache-logs` volume only ever contained an `httpd.pid` file.

**Fix:** Override the `sample-app` container's command to patch
`httpd.conf` at startup, redirecting both directives to real files in the
`logs/` directory (using `combined` format to match the `access_combined`
sourcetype used in `props.conf`/`inputs.conf`):

```yaml
command: >
  sh -c "sed -i 's#CustomLog /proc/self/fd/1 common#CustomLog \"logs/access_log\" combined#' conf/httpd.conf &&
         sed -i 's#ErrorLog /proc/self/fd/2#ErrorLog \"logs/error_log\"#' conf/httpd.conf &&
         exec httpd-foreground"
```

---

## Bug 2: The forwarder was crash-looping on read-only config mounts

**Cause:** The `splunk/universalforwarder` image runs an Ansible-driven
provisioning script on every container start, which includes a task that
recursively `chown`s everything under `$SPLUNK_HOME` (including
`etc/system/local/`). The original compose file bind-mounted the three
`.conf` files as read-only (`:ro`):

```yaml
- ../splunk-config/outputs.conf:/opt/splunkforwarder/etc/system/local/outputs.conf:ro
```

The `chown` failed immediately on the read-only files
("Read-only file system"), retried ~37 times, then failed fatally and
exited — which caused Docker's `restart: unless-stopped` policy to keep
restarting the container in a loop. `splunkd` never actually started on
most boots. It happened to work once, on the very first `docker compose
up`, purely by timing luck, which is why the forwarder looked healthy
during initial testing in Phase 2.

**Fix (attempt 1, insufficient on its own):** Removed `:ro` from the
three mounts so the `chown` could succeed.

---

## Bug 3: Individual file bind-mounts broke Ansible's atomic config writes

**Cause:** Even with the mounts writable, the forwarder's provisioning
also *rewrites* `inputs.conf` itself (to add its own required
`[splunktcp://9997]` stanza), using the standard atomic-write pattern:
write to a temp file, then `rename()` it over the target. On Linux, you
cannot `rename()` on top of a file that is itself the target of an
individual bind mount — the kernel returns `EBUSY`:

```
OSError: [Errno 16] Device or resource busy:
  '/opt/splunkforwarder/etc/system/local/.ansible_tmpXXXXinputs.conf'
  -> '/opt/splunkforwarder/etc/system/local/inputs.conf'
```

This produced a faster, second crash-loop after Bug 2 was fixed.

**Fix:** Mount the whole `splunk-config/` directory at
`etc/system/local/`, instead of mounting each `.conf` file individually:

```yaml
volumes:
  - ../splunk-config:/opt/splunkforwarder/etc/system/local
```

Files inside a bind-mounted *directory* are ordinary files as far as the
container's process is concerned, so `rename()` within that directory
works normally. Only the mount point itself (the directory) is special.

---

## Net result

`compose/docker-compose.yml` changes:

| Service | Before | After |
|---|---|---|
| `sample-app` | no `command:` override | `command:` patches `CustomLog`/`ErrorLog` to write real files |
| `forwarder` | 3 individual `:ro` file mounts | 1 read-write directory mount for all of `splunk-config/` |

With both fixes in place:
- `splunkd` starts cleanly (`RestartCount=0`) on every boot.
- Apache writes real `access_log`/`error_log` files to the shared volume.
- The forwarder tails those files and ships them to the indexer on `9997`.
- `index=webapp_logs` returns events with the correct `access_combined` /
  `httpd_error` sourcetypes.

## How it was diagnosed

1. Confirmed `webapp_logs` index existed via the Splunk REST API
   (`GET /services/data/indexes`), ruling out a missing-index issue.
2. Confirmed the forwarder's `outputs.conf` was correctly connecting to
   the indexer by reading `splunkd.log` for
   `AutoLoadBalancedConnectionStrategy ... Connected to idx=...:9997`.
3. Found the shared `apache-logs` volume contained no log files at all
   (only `httpd.pid`) by `docker exec`-ing into both `sample-app` and
   `forwarder` and listing the mounted directory — pointed at Bug 1.
4. After fixing Bug 1, the forwarder container was found to be
   restarting repeatedly (`RestartCount` climbing); `docker compose logs
   forwarder | grep -i fatal` surfaced the read-only `chown` failure
   (Bug 2).
5. After removing `:ro`, the container was still restarting, just faster;
   the same log grep surfaced the `EBUSY` rename failure (Bug 3).
6. After switching to a directory mount, `splunkd` stayed up
   (`RestartCount=0`), and a REST API search against `webapp_logs`
   returned real events.
