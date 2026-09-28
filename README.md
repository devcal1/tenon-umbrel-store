# Tenon — Umbrel community app store

A private-use [Umbrel](https://umbrel.com) community app store holding the apps Tenon runs on its
own hardware. It contains three apps: **Joinr Backup**, **Joinr Registry** and **Joinr Finance**.

## ⚠️ Read this first

**Uninstalling an app deletes its data.** umbrelOS removes an app's data directory when the app is
uninstalled — no prompt, no undo:

- **Joinr Backup**: every backup it holds. Download the newest archive from the app's own page
  before you uninstall it, and keep a copy somewhere that is not this server.
- **Joinr Finance**: its database **and every backup of it**. Download the newest backup from
  Settings → Backups before you uninstall it, and keep a copy somewhere that is not this server.
- **Joinr Registry**: the stored images. Joinr Finance keeps running, but its next Update or a
  reinstall fails until a new version is released.

For the same reason, **the store id (`tenon`) and every app id (`tenon-joinr-backup`,
`tenon-joinr-registry`, `tenon-joinr-finance`) are permanent**. Umbrel prefixes every community
app's id with its store's id and treats a change to either as uninstall-and-reinstall. Renaming
one line in this repository would destroy that app's data.

## Adding it to your Umbrel

Umbrel dashboard → App Store → the `⋯` menu → **Community App Stores** → add:

```
https://github.com/devcal1/tenon-umbrel-store
```

## The apps

### Joinr Backup

**Joinr Backup** takes a weekly encrypted copy of the [Joinr](https://joinr.com.au) production
database and of the letterheads customers have uploaded to it — logos, banners and terms PDFs —
and keeps the newest sixteen on the server. One 7-Zip archive per run, AES-256, file names
encrypted too.

The files customers attach to quotes are kept beside the archives instead of inside them, in a
folder of separately encrypted files, because an attachment is uploaded once and never changed
where a letterhead is replaced in place. Each archive lists the attachments that existed when it
was taken and which file holds each one, so a full restore takes an archive and that folder
together.

It has a small status page — the last run, the next one, what each run took, the archives with a
download button, and a "Take a backup now" button — behind Umbrel's own login.

It needs four credentials placed in its data directory on the server before it can do anything.
They are put there over SSH, by a helper that runs on the operator's own machine, so that no
password travels through a browser or a chat window. Until they are there, the app's page says
which are missing and takes no backup.

Four further values are optional, placed the same way, and each one is off until it is there:

- `heartbeat-url`, the ping URL of a dead man's switch. Given one, the app pings it when a run
  starts and again when the run finishes, so that a server which has quietly stopped backing up is
  noticed by something other than this page.
- `nas-url` and `nas-password`, the address of an rsync folder to copy the finished archives and
  the attachments into, and its password. With both, every archive the far end is missing is sent
  after each run — so a fortnight with that machine switched off is put right by the first backup
  after it comes back on. Nothing there is ever deleted by this app.
- `nas-heartbeat-url`, a second dead man's switch for the copy alone, so that "the backup has
  stopped" and "the copy is not leaving the server" arrive as two different alarms.

With none of them the app behaves exactly as it did before they existed: it still takes and checks
its weekly backup, and nothing outside the server is watching it or holding a copy.

### Joinr Registry

**Joinr Registry** is a private Docker image registry that listens on the server's loopback only
(`127.0.0.1:4930`). Joinr Finance's image is built on the server and stored here, and Umbrel
installs and updates Joinr Finance from it. It has no web page (the Open button leads nowhere).
Nothing outside the server can reach it, and it runs on its own private Docker network, so other
apps' containers cannot reach it either.

- **Install it before Joinr Finance, and keep it running.** umbreld pulls every image at install
  and at update. An Update clicked while the registry is stopped stops Joinr Finance, bumps its
  manifest and only then fails the pull: the app stays stopped and Umbrel offers no Update to
  retry. As an app, the registry is started by Umbrel at every boot.
- Deleting images through its API is switched off, so a pinned version is never lost by accident.
- ⚠️ Uninstalling it deletes the stored images (see above).

### Joinr Finance

**Joinr Finance** is a self-hosted personal-wealth tracker: net worth, investments, cash and
budget, side income and dividends, super, property and loans, the monthly history and a FIRE
planner, with its own SQLite database in the app's data directory. Its source is the public
[joinr-fin](https://github.com/devcal1/joinr-fin) repository.

- **Backups:** a verified copy of the database every night at 02:30 (server time), keeping the
  newest copy of each of the last 14 days and of each of the last 12 months, plus copies taken by
  hand, before an import, before a restore and before an update. Settings → Backups lists them
  with a download button. ⚠️ They live in the app's data directory, so uninstalling the app
  deletes them (see above).
- **Copy to the NAS (optional):** every Sunday at 03:00 the app copies every backup it keeps to
  an rsync module on a NAS, only ever adding files, and proves each copy by listing the NAS back.
  It is off until its two files (the NAS address and password) are placed in the data directory
  over SSH with joinr-fin's `pnpm umbrel:nas-secrets` helper; umbrelOS Backups skip them.
- **Access:** no login of its own; Umbrel's login stays in front of every path. The app sits on a
  private Docker network that only Umbrel's app proxy joins, so other apps on the server cannot
  call it directly.
- **Where the image comes from:** not GitHub Container Registry. It is built on the Umbrel from
  the public joinr-fin repository and pushed to the Joinr Registry app on loopback; the compose
  file pins it by tag **and** digest.
- **Before you click Install or Update**, run `pnpm umbrel:status` in joinr-fin: it prints "Safe
  to click Update in Umbrel" only when the Joinr Registry app answers and holds the pinned digest.
- The operator's runbook (install, release, restore, rollback, troubleshooting) is joinr-fin's
  [`docs/deploy/RUNBOOK.md`](https://github.com/devcal1/joinr-fin/blob/main/docs/deploy/RUNBOOK.md).

## Ports

| Port | App | Notes |
|---|---|---|
| 4930 | Joinr Registry | host-only: bound to `127.0.0.1`, never published to the LAN |
| 4931 | Joinr Backup | through Umbrel's app proxy |
| 4932 | Joinr Finance | through Umbrel's app proxy |

Each was checked against every app manifest of every app store on the device and against the
host's listeners; joinr-fin's release script re-checks 4930 and 4932 before every release.

## What is in this repository, and what is not

```
umbrel-app-store.yml              the store's id and display name
tenon-joinr-backup/
  umbrel-app.yml                  the manifest — and `version:`, which is what makes Umbrel
                                  offer an update
  docker-compose.yml              the app_proxy service and the image, pinned by tag AND digest
  icon.svg                        the dashboard tile
  data/                           the data directory skeleton (.gitkeep files only)
tenon-joinr-registry/
  umbrel-app.yml                  the manifest (its version is the registry image's)
  docker-compose.yml              the registry image, pinned by tag AND digest, loopback only
  icon.svg                        the dashboard tile
  data/                           the storage skeleton (.gitkeep only)
tenon-joinr-finance/
  umbrel-app.yml                  the manifest — `version:` is the app version
  docker-compose.yml              app_proxy, the app on its private network, and the image from
                                  the loopback registry, pinned by tag AND digest
  icon.svg                        the dashboard tile
  data/                           the data directory skeleton: data/ and data/backups/
```

**This repository is public, so it holds no credential of any kind, and it never will.** It
carries only manifests, compose files, icons and empty data skeletons. Joinr Backup's source lives
in a private repository and its image is published to GitHub Container Registry, pinned here by
digest. Joinr Finance's source is public and its image never leaves the Umbrel.

## Releasing

### Joinr Backup

1. The image is built and pushed from the private repository, which prints the new `@sha256:`
   digest.
2. Update the image line in `tenon-joinr-backup/docker-compose.yml` with the new tag and digest.
3. Bump `version:` in `tenon-joinr-backup/umbrel-app.yml` and write `releaseNotes:` — **an update
   Umbrel never offers is an update nobody gets**, and it is the manifest version it compares, not
   the image.
4. Push. Umbrel polls for changes roughly every five minutes.

### Joinr Finance

1. In joinr-fin: bump `version` in `package.json` (every image change is a new version), then
   `pnpm umbrel:release`. It ships the source to the Umbrel over SSH, builds the image there,
   pushes it to the Joinr Registry app, and writes the tag, the digest and `version:` into this
   clone — never committing. It refuses a new image without a new version.
2. Write `releaseNotes:` in `tenon-joinr-finance/umbrel-app.yml` for that version and review the
   diff.
3. Commit and push this repository. Umbrel polls for changes roughly every five minutes.
4. `pnpm umbrel:status` must print "Safe to click Update in Umbrel"; then click Update.

### Joinr Registry

Only when its image moves: update the image line (tag and digest) in
`tenon-joinr-registry/docker-compose.yml` and `version:` in its manifest together, then push.
