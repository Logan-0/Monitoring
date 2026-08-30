# Private Docker Monitoring

This stack runs on the NAS and provides:

- [Uptime Kuma](https://github.com/louislam/uptime-kuma) for availability,
  certificate-expiry, and scheduled-job monitoring.
- [Dozzle](https://github.com/amir20/dozzle) for container log viewing.
- [Docker Socket Proxy](https://github.com/Tecnativa/docker-socket-proxy) for
  read-only access to the Docker API.

Uptime Kuma creates and manages its administrator account in the browser.
Dozzle uses its built-in login with a local user file. Its password generator
reads the password from a hidden terminal prompt and writes only the bcrypt hash
to `dozzle-users.yml`; the plaintext password is not stored or placed in shell
history.

Do not expose ports 3003 or 3004 to the public internet.

## Initial NAS setup

Run the Docker commands in this section from the Monitoring directory on the
NAS. Open the URLs later from a browser on the same private network.

1. Create the local environment file:

   ```bash
   cp .env.example .env
   ```

2. Open `.env` in your preferred editor or the NAS file-management GUI. Set
   `MONITORING_BIND` to the NAS's private LAN or VPN address. For example, if
   the NAS is reached at `192.168.2.10`, use:

   ```text
   MONITORING_BIND=192.168.2.10
   ```

   Leave the port values at 3003 and 3004 unless they conflict with another
   NAS service. Do not use `0.0.0.0` or a public address.

3. Generate the Dozzle account. This single block asks for the password without
   displaying it, generates the user file without allocating a Docker TTY, and
   removes the temporary shell variable:

   ```bash
   read -rsp 'Dozzle password: ' DOZZLE_PASSWORD
   printf '\n'
   printf '%s\n' "$DOZZLE_PASSWORD" | docker run --rm -i amir20/dozzle:v10.7.5 generate logan --user-roles none > dozzle-users.yml
   unset DOZZLE_PASSWORD
   ```

   `dozzle-users.yml` is ignored by Git and must stay on the NAS.

4. Download the pinned container images:

   ```bash
   docker compose pull
   ```

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Confirm that all three services are running:

   ```bash
   docker compose ps
   ```

## Browser setup

Replace `<NAS_PRIVATE_IP>` with the same address used for `MONITORING_BIND`.

1. Open `http://<NAS_PRIVATE_IP>:3003`.
2. Create the Uptime Kuma administrator account in its browser setup screen.
3. Open `http://<NAS_PRIVATE_IP>:3004` and sign in to Dozzle with username
   `logan` and the password supplied to the hidden prompt during setup.

## Docker container monitoring

Only Docker Socket Proxy receives the host's Docker socket. It permits the
read-only `CONTAINERS` and `INFO` API sections and explicitly rejects write
requests.

In Kuma, add a Docker Host through the Kuma browser UI with this endpoint:

```text
tcp://docker-socket-proxy:2375
```

Dozzle uses the same endpoint automatically. Container actions and shell access
remain disabled.

Container logs can contain sensitive operational information. Keep Kuma,
Dozzle, and the `docker-api` network private even with the restricted socket
proxy.

## Suggested monitors

| Type | Example target | Purpose |
|---|---|---|
| HTTP(s) | `https://example.com/health` | Verify the complete public request path |
| HTTP(s) | An internal health endpoint | Isolate application or proxy failures |
| Docker Container | An application container | Detect stopped or unhealthy containers |
| Docker Container | A database container | Detect database-container failures |
| Push | A generated Kuma push URL | Detect missing backups or scheduled jobs |

Enable certificate-expiry notifications for HTTPS monitors and configure at
least one tested notification channel.

For a scheduled backup, invoke the Kuma push URL only after the backup has been
created and verified:

```bash
HEALTHCHECK_URL='http://monitoring-host:3003/api/push/REPLACE_ME' \
  /path/to/verified-backup-script
```

Choose a heartbeat interval slightly longer than the job's schedule.

## Backup

Kuma stores monitor definitions, history, accounts, and notification settings
in the `kuma-data` volume. Stop Kuma briefly and archive that data:

```bash
mkdir -p backups
docker compose stop uptime-kuma
docker run --rm \
  --volumes-from uptime-kuma:ro \
  -v "$PWD/backups:/backup" \
  alpine \
  tar -czf /backup/kuma-data.tar.gz -C /app/data .
docker compose start uptime-kuma
```

Store a copy outside the monitored host. Never run `docker compose down -v`
unless permanent deletion of all monitoring data is intended.

## External outage coverage

A monitor cannot report the loss of its own host, power, or network connection.
Use an independent external monitoring service or a second instance in another
location for that failure case.

## Updating

Images are pinned to reviewed versions. Update one version at a time, read its
migration notes, back up persistent data, and then run:

```bash
docker compose pull
docker compose up -d
```
