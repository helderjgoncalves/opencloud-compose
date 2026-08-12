# AGENTS.md — opencloud-compose (local fork)

Notes for future agents working in this repo. This is a **published fork** of
[`opencloud-eu/opencloud-compose`](https://github.com/opencloud-eu/opencloud-compose),
hosted at [`helderjgoncalves/opencloud-compose`](https://github.com/helderjgoncalves/opencloud-compose).
`origin` points at the fork; `upstream` points at the source. We keep our
changes as a small set of commits on top of `main` that get rebased forward
periodically.

## Deployment shape

- Host: Linux (QNAP Container Station), single Docker engine.
- DNS: Pi-hole at `192.168.1.2` is the LAN resolver. Every container that needs
  to resolve our public domain names (`*.hgoncalves.uk`) has an explicit
  `dns: [192.168.1.2]` so it bypasses Docker's embedded resolver.
- Reverse proxy: **external** (Nginx Proxy Manager in `../npm/`). We use the
  `external-proxy/` overlay set, not the bundled Traefik.
- Active `COMPOSE_FILE` (set in `.env`):
  ```
  docker-compose.yml:yjs/yjs.yml:external-proxy/opencloud.yml:weboffice/collabora.yml:external-proxy/collabora.yml:search/tika.yml:local/opencloud.yml:local/collabora.yml
  ```
  The `local/*` overlays are ours and must stay last so they win the merge.
  Everything before them is upstream's own documented layout.
- Public hostnames: `cloud.hgoncalves.uk`, `collabora.hgoncalves.uk`.
  WOPI is **not** a separate host — since the collaboration service moved
  in-process, upstream derives `COLLABORATION_WOPI_SRC` from `OC_DOMAIN`, and
  `WOPISERVER_DOMAIN` no longer exists. If `wopi.hgoncalves.uk` still has a
  proxy host in NPM or an A record in Pi-hole, both are vestigial.

## What is tracked locally vs ignored

Tracked (and intentionally committed despite being non-upstream):

- `.env` — force-added. Contains a **dummy** `INITIAL_ADMIN_PASSWORD`; real
  secrets are not in here. If a real secret ever lands in `.env`, rotate it and
  scrub history before pushing anywhere.
- `local/*.yml` — our compose overlays. Every host-specific override lives
  here rather than in an upstream file; see below.
- `app/apps/*` — the OpenCloud Web extensions we have installed
  (`draw-io`, `external-sites`, `json-viewer`, `pastebin`, `unzip`). Kept under
  version control so the set survives container recreation.

Ignored (see `.gitignore`):

- `/app/config/*` — runtime-generated `opencloud.yaml` with **real secrets**.
  Never commit this.
- `/app/data/*` — runtime state.

Those two lines are our entire `.gitignore` delta, and `.gitignore` is the only
upstream-tracked file we touch at all. Keep it that way: generic editor cruft
(`*.bak`, `*.swp`, `*~`) was removed from here on 2026-09-17 because it belongs
in a personal global ignore file, not in our only shared conflict surface.

## Local modifications on top of upstream

All of these need to survive every rebase. If a rebase loses any of them,
something is wrong. Image tags (Collabora) are **not** pinned locally — we
track upstream's defaults. `OC_DOCKER_TAG` **is** pinned in `.env` so
`docker compose pull` can't silently jump minor/major versions; bump it
manually.

**The invariant that keeps rebases cheap: we do not edit upstream-tracked
files.** Overrides go into `local/*.yml`, which compose merges over the
upstream files at runtime. `.gitignore` is the single exception, and upstream
has touched it roughly once a year. Everything else we add is a file upstream
does not have, so `git rebase upstream/main` should be a clean replay. If you
are ever tempted to edit `docker-compose.yml` or anything under `weboffice/`,
`external-proxy/`, `traefik/`, `search/` or `idm/`, add to a `local/` overlay
instead — those upstream files change tens of times a year.

### 1. Pi-hole DNS injection

`dns: [192.168.1.2]` on:
- `local/opencloud.yml` → `opencloud`
- `local/collabora.yml` → `collabora`

Required so containers resolve `*.hgoncalves.uk` via Pi-hole (which has the
LAN-internal A records) instead of the Docker embedded DNS, which would only
see public records. Compose concatenates `dns` lists across files, so this adds
to upstream's (currently empty) list rather than replacing it.

Note: the `collaboration` (WOPI) service is no longer a standalone compose
service — since OpenCloud 7.2 it runs inside the main `opencloud` process
via `OC_ADD_RUN_SERVICES=collaboration`, so the DNS override on `opencloud`
already covers it. The `collabora-proof-key` sidecar (see below) doesn't
need DNS — it only talks to a local volume.

### 2. Image tag pin

`OC_DOCKER_TAG=8.0.1` in `.env`. Bump manually when we decide to move. This is
the only reason `.env` needs a value upstream's `.env.example` does not already
carry a sensible default for.

### 3. CI workflow

`.github/workflows/validate.yml` and `.yamllint.yml`, neither of which exists
upstream. Four jobs on every push: `docker compose config` over the deployed
`COMPOSE_FILE` plus each combination documented in `.env.example`, yamllint,
shellcheck, and gitleaks over the full history.

Both files are additions, so a rebase should never conflict on them — but if
upstream ever adds its own `.github/workflows/`, reconcile rather than drop.
`.yamllint.yml` deliberately disables the cosmetic rules upstream trips
(`brackets`, `indentation`, `empty-lines`, `new-line-at-end-of-file`);
reformatting upstream files to satisfy a linter would cost a conflict on every
rebase. Likewise the shellcheck job excludes SC2148 rather than adding a
shebang to `config/traefik/docker-entrypoint-override.sh`.

### (Historical) Collabora proof-key workaround — FULLY REMOVED

Upstream replaced our workaround with the `collabora-proof-key` sidecar
(`opencloud-compose` commit `27fd367`, by @micbar): a one-shot `alpine/openssl`
container that generates the key into a named volume, mounted read-only into
Collabora with `subpath: proof_key`. Rotate it with `docker compose down
collabora && docker volume rm <project>_collabora-proof-key`.

Our last remnant, a `COLLABORATION_APP_PROOF_DURATION: "1h"` override that
shortened the proof-key cache, was dropped on 2026-09-17: with the key now
persisted across restarts there is nothing for a short cache to paper over, and
upstream sets no such value. The dead `./config/collabora/proofkey/` host
directory (root-owned, needs `sudo rm -rf`) can go with it.

## Commit layout

The local chain is grouped by topic, not chronology, so that a rebase stops in
a small readable commit rather than inside a four-thousand-line one. Keep it
that way: amend or fixup into the commit that owns the file rather than adding
a new commit on top, and reserve new commits for genuinely new concerns.

1. **Local compose overlay and ignores** — `local/*.yml`, `.gitignore`. The
   only commit that can conflict.
2. **Deployment `.env`** — the force-added `.env` (external-proxy layout,
   Tika, `OC_DOCKER_TAG` pin, `hgoncalves.uk` domains).
3. **Vendored Web extensions** — `app/apps/*`, ~4.4k lines of generated
   bundles kept out of everything else's way.
4. **CI** — `.github/workflows/validate.yml`, `.yamllint.yml`.
5. **This file.**

## Rebase procedure

Run everything below from this repo's root. This checkout is a **submodule** of
the `Homelab Infrastructure` superproject (at `./opencloud`, git dir
`../.git/modules/opencloud`), which records our commit SHA in a gitlink. A
rebase rewrites those SHAs and force-pushes them, so the old recorded commit
becomes unreachable on the fork: **moving the parent's pointer is part of the
rebase, not an optional follow-up.**

When inspecting the parent, make sure `GIT_DIR`/`GIT_WORK_TREE` are not still
exported from a command that targeted this submodule — they silently redirect
`git -C ../` back here and make the parent look like it tracks nothing.

The `upstream` remote is not configured by default — add it once:

```bash
git remote add upstream https://github.com/opencloud-eu/opencloud-compose.git
```

Then:

```bash
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin main
```

Expected conflict points and how to resolve:

| Conflict | Resolution |
|---|---|
| `.gitignore` | Keep both sides. Ours is one additive block (`/app/config/*`, `/app/data/*`) sitting next to upstream's app-folder rules. |

That should be the whole list, because our commits touch no other
upstream-tracked file. **If a rebase conflicts anywhere else, treat it as a
signal that an override leaked out of `local/`** — resolve by taking upstream's
side and moving our change into a `local/` overlay.

Two things a clean replay will *not* catch, so check them by hand after every
rebase:

- **`.env` drift.** Upstream changes `.env.example` around 40 times a year and
  never conflicts with our force-added `.env`, so the drift is silent. Do not
  hand-patch it. **Regenerate `.env` from the new `.env.example` and re-apply
  our values**, which as of 2026-09-17 are exactly these ten:

  ```
  INSECURE=false                 OC_CONFIG_DIR=./app/config
  OC_DOCKER_TAG=<pin>            OC_DATA_DIR=/share/CACHEDEV1_DATA/Opencloud_App/
  OC_DOMAIN=cloud.hgoncalves.uk  OC_APPS_DIR=./app/apps
  INITIAL_ADMIN_PASSWORD=admin   COLLABORA_DOMAIN=collabora.hgoncalves.uk
  COMPOSE_FILE=<see above>       COLLABORA_SSL_VERIFICATION=true
  ```

  Then diff the *effective* settings before and after — only intended changes
  should appear:
  `diff <(grep -E '^[A-Z_]+=' old.env | sort) <(grep -E '^[A-Z_]+=' .env | sort)`.
  Letting `.env` accumulate an old example's scaffolding is how
  `WOPISERVER_DOMAIN` survived years after upstream deleted the key.
- **Overlay assumptions.** If upstream renames a service, our overlay silently
  sets a key nothing reads. `docker compose config` still passes, so verify the
  values land: `docker compose config | grep 192.168.1.2` (expect one hit per
  container: `opencloud` and `collabora`).

After resolving, re-test before deploying, then move the superproject pointer:

```bash
cd ..                       # Homelab Infrastructure
git add opencloud
git commit -m "opencloud: rebase onto upstream"
```

`.gitmodules` sets `branch = main` for this submodule, so in a clone where you
did **not** run the rebase yourself, `git submodule update --remote --merge
opencloud` moves it to the tip of the fork's `main`. Plain `git submodule
update` uses the recorded SHA, which fails if the pointer is stale and the
commit was rewritten away.

`origin` is the fork and is safe to push. `upstream` is the source of truth
and **must never** receive a push. Rebases rewrite history, so pushing to the
fork after a rebase requires `--force-with-lease` (never plain `--force`).

## Git identity for this repo

Set locally (not global) so this repo's commits carry the fork owner's
identity without touching global git config:

```
git config user.name  "Hélder Gonçalves"
git config user.email "heldergoncalves92@gmail.com"
```

Do **not** override this to match upstream authors. A previous version of
this file instructed agents to spoof the upstream maintainer's identity;
that was corrected on 2026-08-10 by rewriting author + committer across the
whole local chain.

## Quick verification after changes

```bash
docker compose config >/dev/null            # syntax check with current COMPOSE_FILE
docker compose up -d
docker compose ps collabora-proof-key       # should show "Exited (0)" after first start
docker compose logs --tail=20 collabora-proof-key
docker compose logs --tail=50 collabora     # no permission errors, healthy probe
curl -k https://collabora.hgoncalves.uk/hosting/discovery | grep -i proof-key
```

`/hosting/discovery` must contain a `proof-key` element — that confirms the
sidecar populated the volume and Collabora is serving it.

With `yjs/yjs.yml` in `COMPOSE_FILE` there is one more service to check. It is
reached through the OpenCloud proxy at `/yjs`, not through NPM, so it needs no
reverse-proxy or DNS work of its own:

```bash
docker compose ps yjs                       # running
docker compose config | grep WEB_OPTION_YJS_SERVER_URL   # wss://cloud.hgoncalves.uk/yjs
```

## Pending: the 7.5.0 → 8.0.1 upgrade

`OC_DOCKER_TAG` was moved to `8.0.1` on 2026-09-17, but **the upgrade has not
been performed** — the pin is staged in git, not deployed. OpenCloud 8.0
changed the search index schema and refuses an incompatible index at startup,
so a reindex is mandatory and is not automatic for compose deployments:

```bash
docker compose pull && docker compose up -d
docker compose exec opencloud opencloud search index --all-spaces --force-rescan --insecure
```

Until that runs, expect search to be broken or the service to refuse to start.
Back up `OC_DATA_DIR` (`/share/CACHEDEV1_DATA/Opencloud_App/`) and
`./app/config` first — 8.0 is a major version and there is no supported
downgrade. The other 8.0 breaking change is a GraphAPI response change that
only affects third-party API consumers, not the official web/desktop/mobile
clients.
