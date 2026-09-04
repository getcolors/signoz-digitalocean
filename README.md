# signoz-digitalocean

Desired state for a single-node [SigNoz](https://signoz.io/) observability
stack on DigitalOcean, serving **https://signoz-digitalocean.bigconfig.online**.

Built by the [`signoz`](https://github.com/getcolors/signoz) Package Skill in
any of its three colours (`package-signoz-green`, `-red`, `-blue`, installed
under `.agents/skills/`): OpenTofu manages the droplet in the region's default
VPC, its cloud firewall and a proxied Cloudflare `A` record; Ansible converges
ClickHouse, ClickHouse Keeper, a Postgres metastore, the schema migrator, the
SigNoz application, the `signoz-otel-collector` ingester and Caddy.

This is the signoz package's second-provider deployment under the workspace
Compute Provider Standard: the same package that runs `signoz-vultr`, selected
onto DigitalOcean by `provider-compute` alone.

## Use

```sh
direnv allow               # once, after cloning
./green build              # render .colors/signoz-digitalocean/
./green create --dry-run   # walk the workflow, no side effects
./green create             # converge
```

`./red` and `./blue` run the same verbs against the same state; never run two
colours concurrently.

## Sending telemetry

SigNoz community edition has no ingestion keys, so Caddy gates the OTLP paths
with a bearer token generated on the server:

```sh
token=$(ssh signoz-digitalocean sed -n 's/^SIGNOZ_INGEST_TOKEN=//p' /etc/signoz/ingestion.env)

OTEL_EXPORTER_OTLP_ENDPOINT=https://signoz-digitalocean.bigconfig.online
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer ${token}"
```

Only OTLP/HTTP is published. gRPC on 4317 stays on loopback.

## Signing in

`https://signoz-digitalocean.bigconfig.online` — the root account is the
`signoz-root-email` in `colors.yml`, with `COLORS_PAR_SIGNOZ_ROOT_PASSWORD`
from `.envrc.private`. That variable is the account's only record: the root
user cannot be edited or deleted from the UI.

## Recovery

A daily timer dumps the Postgres metastore to
`r2:signoz-backup/signoz-digitalocean/`. Telemetry is deliberately not backed
up. Restore is `./green create` plus:

```sh
gunzip -c metastore-<stamp>.sql.gz | \
  ssh signoz-digitalocean docker compose -f /opt/signoz/compose.yml exec -T metastore \
    psql -U signoz -d signoz
```

## Credentials

Seven `COLORS_PAR_*` variables in the gitignored `.envrc.private`; the header
of `colors.yml` lists them. Never export `COLORS_PAR_PROFILE`.

## Safety

`compute-prevent-destroy: true` guards deletion; lifting it requires a one-run
`COLORS_PAR_COMPUTE_PREVENT_DESTROY=false` override and separate authorization.
Changing `provider-compute` on this profile is refused while a machine is in
state: a provider switch is a delete followed by a create, never an apply.
