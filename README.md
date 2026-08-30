# Private Docker Monitoring

A private monitoring stack built with:

- [Uptime Kuma](https://github.com/louislam/uptime-kuma) for availability,
  certificate-expiry, and scheduled-job monitoring.
- [Dozzle](https://github.com/amir20/dozzle) for authenticated container log
  viewing.
- [Docker Socket Proxy](https://github.com/Tecnativa/docker-socket-proxy) to
  prevent monitoring services from sending write requests to the Docker daemon.

The stack is intended for a trusted private network. Do not publish its ports
directly to the internet.

## Secure setup

Copy the example configuration:

```bash
cp .env.example .env
```

Generate a Dozzle user. The command prompts for the password so it does not
appear in shell history:

```bash
docker run --rm -it amir20/dozzle:v10.7.5 \
  generate admin --user-roles none > dozzle-users.yml
```

Both `.env` and `dozzle-users.yml` are ignored by Git.

Start the stack:

```bash
docker compose pull
docker compose up -d
```

### First-run Kuma bootstrap

Keep `MONITORING_BIND=127.0.0.1` until the Kuma administrator account exists.
Forward the ports from a trusted client:

```bash
ssh -L 3003:127.0.0.1:3003 -L 3004:127.0.0.1:3004 user@monitoring-host
```

Open `http://127.0.0.1:3003`, create the Kuma administrator, and confirm that
`http://127.0.0.1:3004` requires the Dozzle login. You can then set
`MONITORING_BIND` in `.env` to a specific private interface address and run
`docker compose up -d` again. Never use `0.0.0.0` unless every host interface is
protected by an appropriate firewall.

## Docker container monitoring

Only the socket proxy receives the host's Docker socket. It permits the
read-only `CONTAINERS` and `INFO` API sections and explicitly rejects all
`POST` requests.

In Kuma, create a Docker Host using:

```text
tcp://docker-socket-proxy:2375
```

Dozzle is configured to use the same endpoint automatically. Shell access and
container actions are not enabled.

Read-only Docker API responses and container logs can still contain sensitive
environment or operational information. Keep Kuma, Dozzle, and the
`docker-api` network private even with these controls.

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
