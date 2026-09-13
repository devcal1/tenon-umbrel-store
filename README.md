# Tenon — Umbrel community app store

A private-use [Umbrel](https://umbrel.com) community app store holding the apps Tenon runs on its
own hardware. It contains one app.

## ⚠️ Read this first

**Uninstalling Joinr Backup deletes every backup it holds.** umbrelOS removes an app's data
directory when the app is uninstalled — no prompt, no undo — and that directory is where the
archives live. Download the newest archive from the app's own page before you uninstall it, and
keep a copy somewhere that is not this server.

For the same reason, **the store id (`tenon`) and the app id (`tenon-joinr-backup`) are
permanent**. Umbrel prefixes every community app's id with its store's id and treats a change to
either as uninstall-and-reinstall. Renaming one line in this repository would destroy the
archives.

## Adding it to your Umbrel

Umbrel dashboard → App Store → the `⋯` menu → **Community App Stores** → add:

```
https://github.com/devcal1/tenon-umbrel-store
```

## The app

**Joinr Backup** takes a weekly encrypted copy of the [Joinr](https://joinr.com.au) production
database and of every file customers have uploaded to it, and keeps the newest sixteen on the
server. One 7-Zip archive per run, AES-256, file names encrypted too.

It has a small status page — the last run, the next one, the archives with a download button, and
a "Take a backup now" button — behind Umbrel's own login, which is how the monthly off-site copy
gets made.

It needs four credentials placed in its data directory on the server before it can do anything.
They are put there over SSH, by a helper that runs on the operator's own machine, so that no
password travels through a browser or a chat window. Until they are there, the app's page says
which are missing and takes no backup.

A fifth value is optional: the ping URL of a dead man's switch. Given one, the app pings it when a
run starts and again when the run finishes, so that a server which has quietly stopped backing up
is noticed by something other than this page. Without one the app behaves exactly as it always
has, and nothing outside the server is watching it.

## What is in this repository, and what is not

```
umbrel-app-store.yml              the store's id and display name
tenon-joinr-backup/
  umbrel-app.yml                  the manifest — and `version:`, which is what makes Umbrel
                                  offer an update
  docker-compose.yml              the app_proxy service and the image, pinned by tag AND digest
  icon.svg                        the dashboard tile
```

**This repository is public, so it holds no credential of any kind, and it never will.** The app's
source code lives in a private repository; this store carries only the manifest, the compose file
and the icon. The image is published to GitHub Container Registry from that private repository and
pinned here by digest.

## Releasing

1. The image is built and pushed from the private repository, which prints the new `@sha256:`
   digest.
2. Update the image line in `tenon-joinr-backup/docker-compose.yml` with the new tag and digest.
3. Bump `version:` in `tenon-joinr-backup/umbrel-app.yml` and write `releaseNotes:` — **an update
   Umbrel never offers is an update nobody gets**, and it is the manifest version it compares, not
   the image.
4. Push. Umbrel polls for changes roughly every five minutes.
