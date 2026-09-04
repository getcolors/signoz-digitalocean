# CLAUDE.md

## Repository

`signoz-digitalocean` is a **deployment**: desired state only, no source code.
It runs the `signoz` Package Skill (`../signoz`) against one DigitalOcean
Droplet serving `signoz-digitalocean.bigconfig.online`.

This is the second-provider deployment of the workspace Compute Provider
Standard (`../workspace/standards/compute-provider.md`): the same package that
runs `signoz-vultr`, on DigitalOcean by `provider-compute: digitalocean` and
the `digitalocean-*` keys alone. The hostname is deliberately distinct from
`signoz-vultr`'s so a rebuild of either can never collide with the other.

`colors.yml` is the only file to edit. Everything under `.colors/` is generated
and must never be edited, read as source, or committed. `.envrc.private` holds
every credential and must never be read, printed, or copied.

## Commands

```sh
./green build              # render .colors/signoz-digitalocean/ — contacts nothing
./green create --dry-run   # walk the workflow, skip every side effect
./green create             # converge for real — requires explicit authorization
./green delete             # guarded and destructive — separately authorized
```

`./red` and `./blue` run the same verbs against the same state; never run two
colours concurrently. `build` and `--dry-run` work with an empty environment
and are the safe way to check a `colors.yml` edit.

## Provider switching is a rebuild

Every real `create` and `delete` reads the recorded `params.provider` from
state before validating provider credentials and refuses when it differs from
`provider-compute`. To move this profile to another provider: `delete` on the
recorded provider, then edit `provider-compute` and `create`.

## The launchers are copies

The root `green`, `red` and `blue` are **copies** of
`.agents/skills/package-signoz-<colour>/<colour>`, not symlinks.
`npx skills update -p` rewrites the payloads and leaves the root files alone.
After every update copy each launcher over its root file. Never hand-edit a
SHA.

## What this deployment assumes

- The Cloudflare token can edit the `bigconfig.online` zone; this package only
  reads the zone and writes one record.
- The `signoz-backup` R2 bucket exists. This deployment owns only its
  `signoz-digitalocean/` prefix and must never delete the bucket or other
  objects.
- The droplet joins the account's default VPC for `ams3`, discovered at plan
  time; this deployment neither creates a VPC nor records a VPC UUID.
- Keygen mode: `~/.ssh/signoz-digitalocean` is generated and owned by the
  package, and the DigitalOcean account key named `signoz-digitalocean`
  belongs to this deployment's state. Cloning this repository elsewhere does
  not carry machine access; adding `digitalocean-ssh-keys` here would switch
  the deployment to opt-out mode.

## Secrets

`COLORS_PAR_SIGNOZ_ROOT_PASSWORD` is the root account's only record — the
account cannot be edited or deleted from the UI, and provisioning runs at
application startup only, so a change takes effect on the next recreate.

The OTLP ingestion bearer token and the Postgres password are generated on the
server and exist nowhere else. Read the token with
`ssh signoz-digitalocean cat /etc/signoz/ingestion.env`.

## Documentation

`index.html` carries the two analytics tags every repository page carries:
GA4 measurement ID `G-4VKP1WY4QJ`, whose explicit `page_title` must equal the
decoded HTML `<title>`, and the self-hosted Rybbit snippet
`<script src="https://rybbit.getcolors.ai/api/script.js" data-site-id="9fb9c41a6d49" defer></script>`.
Never add one tag without the other. `verification.md` records the live
verification this deployment passed; it carries no address, key id or token.

## Git

Work on the current branch. Do not commit or push unless explicitly authorized.
