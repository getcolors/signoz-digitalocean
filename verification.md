# Live verification

The record of the live verification this deployment passed, per the workspace
Compute Provider Standard §7-8. Commands, outcomes and timings only: no
address, key id or token appears here.

## What was verified

| | |
|---|---|
| Date | 2026-09-04 (UTC) |
| Provider | DigitalOcean, region `ams3` |
| Plan | `s-4vcpu-8gb`, image `ubuntu-24-04-x64` |
| Package | `getcolors/signoz` at `f171474` (pin commit `ad65f52`); first attempt at `6c87dba` |
| Mode | keygen (no `digitalocean-ssh-keys`), compute name = profile |
| State | R2, `signoz-digitalocean/signoz-infrastructure.tfstate` |

## Sequence and outcomes

| # | Command | Outcome |
|---|---|---|
| 1 | `./green create` (at `6c87dba`) | infrastructure 56 s, ssh-config 2 s, dns 5 s, then **exit 2 in the converge**: `docker compose up -d --wait` reported `container signoz-signoz-1 is unhealthy`; the application logged `migrate: migrations table is already locked` until its two-minute lock timeout. Postgres held no advisory lock: the lock is bun-migrate's row in `migration_lock`, left by an application process the play itself had killed two seconds after its first start (the pre-converge `flush_handlers` ran `Restart SigNoz` and then `Recreate the SigNoz application` back to back). A package bug that had passed on Vultr by timing alone; fixed as `f171474` (handlers flushed after the converge). Host recovery: `docker compose stop signoz`, `delete from migration_lock` (one row), with no application container running. |
| 2 | `./green create` (at `f171474`) | **exit 0.** Stages: infrastructure 4 s (idempotent), ssh-config 2 s, dns 3 s, ansible 105 s, acceptance 0.3 s (ingestion through the public path behind the bearer token, the telemetry schema, the `histogramQuantile` function). |
| 3 | `ssh signoz-digitalocean` | six containers up, application healthy, `migration_lock` empty, `/api/v1/health` 200. |
| 4 | `./red create` | **exit 0**, idempotent: ansible 56 s, acceptance 0.3 s. |
| 5 | `./blue create` | **exit 0**, idempotent: ansible 56 s, acceptance 0.3 s. |
| 6 | `COLORS_PAR_PROVIDER_COMPUTE=vultr ./green create` | **exit 2** before any credential or provider call: `state holds a digitalocean machine; set provider-compute back to digitalocean and delete first`. |
| 7 | `COLORS_PAR_PROVIDER_COMPUTE=vultr ./green delete` | **exit 2**, the same refusal, ahead of the prevent-destroy guard. |

Attempt 1 is a converge failure found by the port, not caused by it. Rows 4
and 5 prove the three colours manage one state interchangeably; 6 and 7 prove
Compute Provider Standard §4 on a live state. The manual lock recovery is
recorded here because the standard forbids hiding it: the package's own
documentation now names the symptom and the same recovery.

## Not verified here

- The Vultr side of the package, which `signoz-vultr` covers.
- The nightly metastore backup timer and its restore path: the schedule had
  not fired within the verification window.

## After verification

Deleted the same day under a fresh, explicit authorization, with the one-run
`COLORS_PAR_COMPUTE_PREVENT_DESTROY=false` override: cleanup play, DNS record,
`~/.ssh/config` block, droplet and firewall, account key, local keypair, in
that order, exit 0. Verified read-only afterwards that nothing named after the
profile survives at the provider. The repository, `colors.yml` and the R2
state remain, so the deployment is re-creatable with `./green create`.
