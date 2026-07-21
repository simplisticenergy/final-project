# Troubleshooting: Two Bugs Exposed by Recreating Containers

## Symptom

While standing up the version-controlled alert app (`splunk-apps/webapp_alerts`),
recreating the `splunk` container revealed that `index=webapp_logs` returned
zero results over **all time**, and the `forwarder` container was
crash-looping (`RestartCount` climbing, ~90s per cycle). Both problems were
latent — they were armed by an earlier `docker compose down`/`up` cycle that
morning, not by the alert change itself; the alert deploy just forced the
container recreation that surfaced them.

---

## Bug 1: The `webapp_logs` index definition didn't survive container recreation

**Cause:** The index was originally created by hand in Splunk Web, which
stores the definition (`indexes.conf`) under `/opt/splunk/etc/`. Only
`/opt/splunk/var` is on a named volume (`splunk-var`) — `etc/` is not — so
recreating the `splunk` container silently discarded the index definition
while the actual bucket data survived on the volume
(`/opt/splunk/var/lib/splunk/webapp_logs/`). The indexer **drops** events
forwarded to a nonexistent index, so ingestion appeared totally dead even
though the forwarder was (when running) connecting and shipping fine.

This had never bitten before because `docker compose up -d` only recreates
containers whose configuration changed — routine Jenkins deploys with an
unchanged compose file reused the existing container, so the UI-created
index quietly persisted until the first real recreation.

**Fix:** Define the index in version-controlled config instead of the UI:
`splunk-apps/webapp_alerts/default/indexes.conf`, in the app bind-mounted
into the indexer at `/opt/splunk/etc/apps/webapp_alerts`:

```ini
[webapp_logs]
homePath = $SPLUNK_DB/webapp_logs/db
coldPath = $SPLUNK_DB/webapp_logs/colddb
thawedPath = $SPLUNK_DB/webapp_logs/thaweddb
```

On restart, Splunk re-attached the existing buckets from the volume — the
historical events came back along with the index. Events forwarded during
the window when the index didn't exist were dropped and are unrecoverable.

---

## Bug 2: Forwarder crash-looping on a stale `splunk.secret`

**Cause:** The forwarder writes runtime files — notably `server.conf`, which
holds `sslPassword`/`pass4SymmKey` values encrypted with the instance's
`splunk.secret` — into the bind-mounted `splunk-config/` directory. The
`splunk.secret` itself lives in `etc/auth/` **inside** the container, which
is not bind-mounted. When the container was recreated after `docker compose
down`, it generated a fresh `splunk.secret`, but the bind-mounted
`server.conf` on the host still held values encrypted with the old one.
Every boot then failed to decrypt its own SSL key material:

```
ERROR Crypto - Decryption operation failed: AES-GCM Decryption failed!
ERROR SSLCommon - Can't read key file /opt/splunkforwarder/etc/auth/server.pem
```

so splunkd's management port (8089) never came up, the image's Ansible
provisioning task "Check for required restarts" got connection-refused,
retried, failed fatally, and `restart: unless-stopped` looped the container.

**Fix:** Delete the stale generated file from the host:

```bash
sudo rm splunk-config/server.conf
```

The forwarder regenerates it (encrypted with the *current* secret) on next
boot. `server.conf`, `web.conf`, and `migration.conf` are already gitignored
as runtime state — this bug is exactly why. **Operational rule:** after any
`docker compose down` (which discards the container's `etc/` and therefore
its `splunk.secret`), delete `splunk-config/server.conf` before bringing the
stack back up.

---

## How it was diagnosed

1. The new alert's scheduled search ran but matched nothing; an ad hoc
   search showed zero `access_combined` events despite fresh 404 traffic
   confirmed present in the Apache `access_log` file.
2. `GET /services/data/indexes` on the indexer showed no `webapp_logs`
   index at all, while `ls /opt/splunk/var/lib/splunk/` showed the old
   `webapp_logs` data directory intact on the volume (Bug 1).
3. `docker ps` showed the forwarder freshly restarted with
   `RestartCount=6`; `docker logs forwarder` showed the Ansible fatal on
   "Check for required restarts" with connection-refused against 8089.
4. The forwarder's `splunkd.log` showed the AES-GCM decryption failures
   starting at 02:06 — *before* the alert deploy — matching the morning's
   `down`/`up` cycle and pointing at the stale encrypted `server.conf`
   (Bug 2).
5. After both fixes and a restart: forwarder stable (`RestartCount` no
   longer climbing), `webapp_logs` present with its historical events, and
   the forwarder's log showing `Connected to idx=...:9997`.
