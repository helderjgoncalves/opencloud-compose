# AGENTS.md — opencloud-compose (local fork)

Notes for future agents working in this repo. This is a **published fork** of
[`opencloud-eu/opencloud-compose`](https://github.com/opencloud-eu/opencloud-compose),
hosted at [`helderjgoncalves/opencloud-compose`](https://github.com/helderjgoncalves/opencloud-compose).
`origin` points at the fork; `upstream` points at the source. We keep our
changes as a small set of commits on top of `main` that get rebased forward
periodically.

## Deployment shape

- Host: Linux (QNAP Container Station), single Docker engine.
- DNS: AdGuard Home at `192.168.1.2` is the LAN resolver. Every container that
  needs to resolve our public domain names (`*.hgoncalves.uk`) has an explicit
  `dns: [192.168.1.2]` so it bypasses Docker's embedded resolver.
- Reverse proxy: **external** — the superproject's Traefik (`../traefik/`),
  routing `drive` to `127.0.0.1:9200` and `collabora` to `127.0.0.1:9980`. We
  use the `external-proxy/` overlay set, not this repo's bundled Traefik.
- Active `COMPOSE_FILE` (set in `.env`):
  ```
  docker-compose.yml:yjs/yjs.yml:external-proxy/opencloud.yml:weboffice/collabora.yml:external-proxy/collabora.yml:local/opencloud.yml:local/collabora.yml
  ```
  The `local/*` overlays are ours and must stay last so they win the merge.
  Everything before them is upstream's own documented layout.
- Public hostnames: `drive.hgoncalves.uk`, `collabora.hgoncalves.uk`.
  WOPI is **not** a separate host — since the collaboration service moved
  in-process, upstream derives `COLLABORATION_WOPI_SRC` from `OC_DOMAIN`, and
  `WOPISERVER_DOMAIN` no longer exists. If `wopi.hgoncalves.uk` still has a
  Traefik router or a DNS rewrite in AdGuard, both are vestigial.

## What is tracked locally vs ignored

Tracked (and intentionally committed despite being non-upstream):

- `local/*.yml` — our compose overlays. Every host-specific override lives
  here rather than in an upstream file; see below.
- `app/apps/*` — the OpenCloud Web extensions we commit, as both the
  upstream release zip and its extracted tree: `apollon` 1.1.0, `bpmn` 1.1.1,
  `calculator` 2.1.0, `external-sites` 2.2.0, `pastebin` 2.2.0, `unzip` 2.2.1,
  all from
  [`opencloud-eu/web-extensions`](https://github.com/opencloud-eu/web-extensions)
  releases, which ship a `sha256sum.txt` to check the zip against.
  `apollon` (UML) syncs live through the yjs server. Kept under version
  control so the set survives container recreation, with the zips alongside
  so an exact build is reproducible.
  `excalidraw` 1.1.0 stays installed under `OC_APPS_DIR` and is gitignored.
  The upstream bundle embeds Excalidraw's public Firebase web config for
  `excalidraw-room-persistence`, which GitHub secret scanning reports as a
  Google API key. Do not commit the zip or the extracted tree.
  Every manifest carries `name` and `version`; a manifest holding only a
  hashed `entrypoint` is an older build whose version is unknowable from the
  repo. `draw-io` is deliberately absent — 2.2.0 fails to load in the web UI.
  So are `json-viewer` 2.1.0, `maps` 3.1.1 and the community app `forms` 1.2.0
  ([`JulesMellot/opencloud-forms`](https://github.com/JulesMellot/opencloud-forms)):
  they show "Failed" in Admin settings → Apps on 9.0.0, most likely because
  they were built with an extension-sdk older than 8.1.0, so they were removed on 2026-10-28 (copies
  in `../backups/opencloud-9.0.0/apps-before-update-*.tgz`). Re-add one only
  from a release built with extension-sdk 8.1.0 or later — its
  `js/localSharedImportMap-*.mjs` lists `yjs` among the shared modules.
  The web service reads this directory only at startup, and the running
  `config.json` points at each app's hashed `remoteEntry-*.mjs`, so swap an
  app's directory and restart `opencloud` in one go; swapping early breaks
  that app until the restart.
  Note that the zips live inside `OC_APPS_DIR`, which mounts to the
  web-served `/var/lib/opencloud/web/assets/apps`, so they are publicly
  downloadable.

Ignored (see `.gitignore`):

- `/app/config/*` — runtime-generated `opencloud.yaml` with **real secrets**.
  Never commit this.
- `/app/data/*` — runtime state.
- `.env` — the deployed environment, ignored by upstream's own rule. It is
  never committed: the values that matter are recorded under *`.env` drift*
  in the rebase procedure below, which is the only copy outside the host. If
  `.env` is lost, rebuild it from `.env.example` plus that list.

The first two are our entire `.gitignore` delta, and `.gitignore` is the only
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

This covers edits you did not make deliberately. On 2026-10-02 an editor YAML
formatter rewrote `docker-compose.yml` simply on opening it: document marker
stripped, `command` flow sequence expanded to a block list, quotes dropped
from around `${...}`, `opencloud-net:` rewritten as `opencloud-net: null`.
Thirty-one lines, parsing to an identical object, and a guaranteed conflict on
every future rebase. Disable format-on-save for YAML in this workspace.

### 1. LAN DNS injection

`dns: [192.168.1.2]` on:
- `local/opencloud.yml` → `opencloud`
- `local/collabora.yml` → `collabora`

Required so containers resolve `*.hgoncalves.uk` via AdGuard Home (which has
the LAN-internal rewrites) instead of the Docker embedded DNS, which would only
see public records. Compose concatenates `dns` lists across files, so this adds
to upstream's (currently empty) list rather than replacing it.

Note: the `collaboration` (WOPI) service is no longer a standalone compose
service — since OpenCloud 7.2 it runs inside the main `opencloud` process
via `OC_ADD_RUN_SERVICES=collaboration`, so the DNS override on `opencloud`
already covers it. The `collabora-proof-key` sidecar (see below) doesn't
need DNS — it only talks to a local volume.

### 2. Memory cap on `opencloud`

`mem_limit: 3g` and `memswap_limit: 3g` in `local/opencloud.yml`. A stalled
embedded NATS bus sends every internal service into a retry loop that queues
events in memory; unbounded, that exhausts host RAM and pushes the NAS into
swap thrash on the array. Steady state is ~850MB. `memswap_limit` must equal
`mem_limit`, or the container spills into swap instead of being killed.

### 3. No Tika

`search/tika.yml` is left out of `COMPOSE_FILE` entirely (since 2026-10-28),
so there is no `tika` container, no `depends_on` on it, and search runs on
OpenCloud's default `basic` extractor with full-text search off. Nothing in
`local/` needs to override anything for this; earlier we kept
`search/tika.yml` and forced `SEARCH_EXTRACTOR_TYPE: basic` from
`local/opencloud.yml`, which still left `FRONTEND_FULL_TEXT_SEARCH_ENABLED`
on and restarted the idle Tika container on every `up`.

Why: on 2026-10-24 Tika had been re-enabled the day before because
the mtime cascade (opencloud-eu/opencloud#3561, fixed by reva #806 in 8.1.0)
could not be reproduced. A single-space v8 rescan of Hélder Gonçalves's
personal space, `opencloud search index --space <id> --force-rescan
--insecure`, then pinned OpenCloud at the 3g cap (~370% CPU) and the drive
stopped answering. Tika itself was idle by the end. The cap kept the host
up; the container was restarted and the rescan was killed. Logs from that
window are in `../opencloud-logs/2026-10-24-hgoncalves-rescan/`. #3589 stays
open. Do not add `search/tika.yml` back until that rescan can finish without
filling the cap.

### 4. Image tag pin

`OC_DOCKER_TAG=9.2.0` in the deployed (untracked) `.env`. Bump manually when
we decide to move, and update the value list under *`.env` drift* to match.
`YJS_DOCKER_TAG` is deliberately **not** pinned: like Collabora it tracks
upstream's default, which upstream moves in lockstep with OpenCloud.

### 5. CI workflow

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

### 6. Mail and guest links

Enabled on 2026-10-28, entirely through `.env` (values under *`.env` drift*):
`START_ADDITIONAL_SERVICES="notifications"` plus the `SMTP_*` settings, and
`OC_ENABLE_GUEST_LINKS=true`. Both services run inside the `opencloud`
process; there is no extra container. `weboffice/collabora.yml` appends
`collaboration` to `START_ADDITIONAL_SERVICES`, so the container must show
`OC_ADD_RUN_SERVICES=notifications,collaboration`.

Mail goes out through [Resend](https://resend.com) (free plan: 100/day,
3,000/month, counting every notification and digest, not only guest links)
as `noreply@hgoncalves.uk`, which has no mailbox. Resend verified the domain
through Cloudflare DNS: DKIM at `resend._domainkey`, the bounce MX and SPF at
`send`, DMARC at `_dmarc`. Port 465 is implicit TLS, which OpenCloud calls
`ssltls`. The API key is send-only and lives in `.env` as `SMTP_PASSWORD`
and nowhere in git.

Guest links (OpenCloud 9) let a user share with an email address that has no
account: the guest gets a single-use 30-minute link and a 24-hour session,
web only. They need the `auth_guest.service_account` that the 9.0
`opencloud init --diff` patch added to `opencloud.yaml`.

### 7. Collabora admin console off

`local/collabora.yml` gives `collabora` a `command:` of
`--o:admin_console.enable=false`. The image's entrypoint is `coolwsd` itself
and upstream sets no `command` for it, so this appends one flag instead of
overriding upstream's whole `extra_params`.

Why: the console (`/browser/dist/admin/admin.html`) is served on the public
`collabora` hostname, and with `COLLABORA_ADMIN_USER`/`_PASSWORD` empty in
`.env` upstream falls back to `admin`/`admin`. Until 2026-11-03 it accepted
that login from the internet. Nothing here uses the console.

Traefik (`../traefik/dynamic/routes.yml`, router `collabora-internet-deny`)
also answers 403 on the console and on Collabora's conversion API
(`/cool/convert-to` and friends) for requests that came through the
Cloudflare tunnel. OpenCloud reaches Collabora over the LAN, so it is not
affected.

### 8. Not enabled on purpose

- **imagor** (`thumbnails/imagor.yml`, HEIC/HEIF/AVIF previews): photos are
  Immich's job, not OpenCloud's. The overlay would add an uncapped container
  that every preview depends on, not only HEIC ones, and OpenCloud would not
  start without it.
- **Tika**: see section 3.

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
2. **Vendored Web extensions** — `app/apps/*`, the release zips plus ~7k
   lines of generated bundles, kept out of everything else's way. Add,
   upgrade or drop an extension by amending this commit rather than stacking
   a new one on top.
3. **CI** — `.github/workflows/validate.yml`, `.yamllint.yml`.
4. **This file.**

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
  never touches our untracked `.env`, so the drift is silent. Do not
  hand-patch it. **Regenerate `.env` from the new `.env.example` and re-apply
  our values**, which as of 2026-10-28 are exactly these twenty. This list is
  the only record of them outside the host — update it whenever one changes.
  `SMTP_PASSWORD` is the one value not written down here; take it from the
  old `.env` or issue a new send-only key in Resend:

  ```
  INSECURE=false                 OC_CONFIG_DIR=./app/config
  OC_DOCKER_TAG=<pin>            OC_DATA_DIR=/share/CACHEDEV1_DATA/Opencloud_App/
  OC_DOMAIN=drive.hgoncalves.uk  OC_APPS_DIR=./app/apps
  INITIAL_ADMIN_PASSWORD=admin   COLLABORA_DOMAIN=collabora.hgoncalves.uk
  COMPOSE_FILE=<see above>       COLLABORA_SSL_VERIFICATION=true
  COLLABORA_HOME_MODE=true       OC_ENABLE_GUEST_LINKS=true
  START_ADDITIONAL_SERVICES="notifications"
  SMTP_HOST=smtp.resend.com      SMTP_PORT=465
  SMTP_SENDER="OpenCloud <noreply@hgoncalves.uk>"
  SMTP_USERNAME=resend           SMTP_PASSWORD=<Resend API key>
  SMTP_AUTHENTICATION=plain      SMTP_TRANSPORT_ENCRYPTION=ssltls
  ```

  `COLLABORA_HOME_MODE` is load-bearing, not a preference. Upstream already
  passes `--o:welcome.enable=false` in `weboffice/collabora.yml`, but that
  setting is a no-op in CODE builds — Collabora treat the welcome dialog as
  intentional and added home mode (upstream issue #4876) as the only supported
  way to suppress it. Dropping the value brings the dialog back on every
  login, at the cost of a 10-document / 20-connection cap.

  Then diff the *effective* settings before and after — only intended changes
  should appear:
  `diff <(grep -E '^[A-Z_]+=' old.env | sort) <(grep -E '^[A-Z_]+=' .env | sort)`.
  Letting `.env` accumulate an old example's scaffolding is how
  `WOPISERVER_DOMAIN` survived years after upstream deleted the key.
- **Overlay assumptions.** If upstream renames a service, our overlay silently
  sets a key nothing reads. `docker compose config` still passes, so verify the
  values land: `docker compose config | grep 192.168.1.2` (expect one hit per
  container: `opencloud` and `collabora`), and
  `docker compose config | grep -E 'mem_limit|tika'` (expect `mem_limit`
  only — any `tika` hit means `search/tika.yml` crept back in).

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
reached through the OpenCloud proxy at `/yjs`, not a Traefik route, so it
needs no reverse-proxy or DNS work of its own:

```bash
docker compose ps yjs                       # running
docker compose config | grep WEB_OPTION_YJS_SERVER_URL   # wss://drive.hgoncalves.uk/yjs
```

## Upgrade to 9.2.0 — pinned 2026-11-17

`OC_DOCKER_TAG` moves to `9.2.0` on 2026-11-17 for the rolling security
release, after rebasing onto upstream `59ca288` (the compose default tag
bump is `5c9355f`). Reva 3.0.1 rejects node ids and revision keys that
leave their space. The same reva release merged workpackages back to
main, and there is no 9.2 migration guide, so run `opencloud init --diff`
with the 9.2.0 image and read the patch before starting the new process.
Do not run `search index --force-rescan`. Collabora and yjs tags are
unchanged by this rebase. The same-day production tag 9.1.1 is not the
image this host runs.

## Upgrade to 9.1.0 — pinned 2026-11-12

`OC_DOCKER_TAG` moved to `9.1.0` on 2026-11-12 because of security fixes,
after rebasing onto upstream `4b6c589`. 9.0.0 is exposed to an
unauthenticated remote crash of the whole process through concurrent requests
to the public web theme endpoint (GHSA-5j99-3p8p-vpjm, fixed in 9.1.0 by
deep-copying the theme defaults before merging), a stored XSS in the
notification bell via file names (GHSA-7w3h-283r-4g2x, web 8.2.2, which 9.1.0
bundles), and an authenticated search-query crash (GHSA-p6cc-4vh9-2hjq).

The release notes list no migration from 9.0.0: `docker compose pull && docker
compose up -d --remove-orphans`, then `docker compose images opencloud` should
show `9.1.0`. The same pull moves Collabora to 26.04.6.1.1 (upstream default).
9.1.0 also bounds bleve indexing memory (opencloud-eu/opencloud#3900), worth
retesting against the 3g cap before reconsidering Tika.

## Upgrade to 9.0.0 — deployed 2026-10-28

Running: `opencloudeu/opencloud-rolling:9.0.0`, `collabora/code:26.04.5.2.1`,
`opencloudeu/yjs:1.1.1`. The upgrade was scripted; the scripts, logs, and the
pre-upgrade copies of `app/config` and `.env` are on the host only, in
`../backups/opencloud-9.0.0/` (ignored by the superproject). A QNAP snapshot
of `Opencloud_App` was taken with OpenCloud stopped, before anything changed.

What 9.0 needed here:

- **Config patch.** `opencloud init --diff --insecure false` in a throwaway
  9.0.0 container over `app/config`, then `patch opencloud.yaml <
  opencloud.config.patch`. It added `auth_guest.service_account` and removed
  the `thumbnails.thumbnail` block, whose keys 9.0 no longer reads.
- **Thumbnails** moved into the webdav service. The cache now lives in
  `thumbnails/cache` and starts cold; the old `thumbnails/files` was deleted.
  Nothing else to set in the single binary.
- No search reindex. `GRAPH_SPACES_DEFAULT_QUOTA` (renamed to
  `OC_SPACES_DEFAULT_QUOTA` in 9.0) was never set here.

Mail and guest links were turned on afterwards as a separate step (section 6),
so a mail problem could not be mistaken for an upgrade problem.

## Upgrade to 8.1.0 — history

`OC_DOCKER_TAG` was moved to `8.0.1` on 2026-09-17 and to `8.1.0` on
2026-10-10. Do not re-run the 8.0 reindex below. `--force-rescan` is what
took the host down; see the Tika note above.

8.1.0 itself had no breaking changes and needed no migration. The step that
mattered was crossing 8.0: OpenCloud 8.0 changed the search index schema and
refuses an incompatible index at startup, so a host still on 7.5.x would have
needed a reindex, and that reindex is not automatic for compose deployments.
That crossing is done on this host. The command it required, which must not
be run again here:

```bash
docker compose pull && docker compose up -d
docker compose exec opencloud opencloud search index --all-spaces --force-rescan --insecure
```

`--force-rescan` restarts from zero on every retry, and this is the command
that exhausted host RAM. Search is expected to keep working on the existing
index. There is no supported downgrade from 8.0. The other 8.0 breaking
change is a GraphAPI response change that only affects third-party API
consumers, not the official web/desktop/mobile clients.

The same upgrade moved `yjs` from 1.0.0 to 1.1.0 (upstream's new default).
