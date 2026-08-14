---
title: ShotGrid Sync — Controlling What Reaches ShotGrid
description: Stop caches and workfiles cluttering ShotGrid by syncing only versions that have reviewable media, with a bypass list for exceptions.
published: true
date: 2026-08-04T00:00:00.000Z
tags: [ayon, shotgrid, versions, sync, settings]
editor: markdown
dateCreated: 2026-08-04T00:00:00.000Z
---

# ShotGrid Sync — Controlling What Reaches ShotGrid

By default, **every** version published in AYON creates a Version entity in ShotGrid —
caches, workfiles, USD layers and everything else, not just the things people actually
review. On a busy show that buries the reviewable versions supervisors care about.

The studio build of the ShotGrid addon adds a filter: **only sync versions that have
reviewable media**, with a bypass list for anything that should always go through.

> This is off by default. With it off, the addon behaves exactly as it always has.
{.is-info}

---

## Where the settings are

**AYON web UI → Settings → Studio settings (or a project's settings) → ShotGrid →
Version Sync Settings**

Set it studio-wide and it applies everywhere; set it on a single project to override
just that show.

---

## The two settings

### Only Upload Versions with Reviewable to ShotGrid

Off by default. Turn it on and a version reaches ShotGrid only if it has reviewable
media attached in AYON — a movie, an image sequence, a thumbnail-able render.

Anything without a reviewable is skipped and noted in the service log.

### Bypass Product Names

A list of product names that always sync, even when the setting above is on. Use it for
things you need in ShotGrid regardless of whether anyone reviews them.

> **Entries match as a prefix, not exactly.** Entering `pointcache` also catches
> `pointcacheMain`, `pointcacheHi` and anything else starting with those letters.
>
> That's usually what you want — one entry covers a whole family of products. But it
> does mean a short entry catches more than you might expect: `render` would also match
> `renderMain`, `renderLayout` and `renderTurntable`.
{.is-warning}

Leave the list empty if you want the reviewable rule to apply to everything.

---

## A worked example

Say you want ShotGrid to show review media only, but you also track published camera
data there.

| Setting | Value |
|---|---|
| Only Upload Versions with Reviewable | ✅ on |
| Bypass Product Names | `camera` |

Result:

| Published | Reaches ShotGrid? | Why |
|---|---|---|
| `renderMain` with a review mov | ✅ | Has a reviewable |
| `pointcacheMain`, no review media | ❌ | No reviewable, not bypassed |
| `workfileLighting` | ❌ | No reviewable, not bypassed |
| `cameraMain` | ✅ | Starts with `camera` — bypassed |

---

## Checking it's working

The service log names every skip:

```
Skipping ShotGrid upload for version '<id>': no reviewable media found
and 'require_reviewable_for_sg_upload' is enabled.
```

If you see those and ShotGrid stops filling with caches, it's on and working.

> **Publishing order doesn't matter.** If a version is published first and its
> reviewable is uploaded afterwards, the version still reaches ShotGrid — the addon
> creates it at that point and then uploads the media. You don't have to change how
> anyone works.
{.is-info}

---

## Troubleshooting

| What you see | What to check |
|---|---|
| Turning it on made no difference — caches still appear in ShotGrid | The **services** probably weren't redeployed. See [If nothing changes](#if-nothing-changes) |
| A product you listed in Bypass still isn't syncing | Check spelling and case, and remember it's matched from the **start** of the product name. `Main` will not match `pointcacheMain` — only a leading match counts |
| More products bypassed than expected | A short entry is matching a whole family. Lengthen it — `cameraMain` rather than `camera` |
| A version with a review mov didn't sync | Confirm the media is attached as a **reviewable** in AYON, not just as a representation |
| Everything stopped syncing | Check the bypass list isn't empty *and* the reviewable requirement isn't on for a project where nothing publishes reviewables |

### If nothing changes

This feature lives in the ShotGrid **services**, not just the server addon. Uploading a
new addon version alone will not enable it — the **leecher** and **transmitter** service
images have to be rebuilt and restarted too.

The quickest check: the studio build reports version **`0.6.17-tma`**. If your running
services report `0.6.17+dev` or anything without the `-tma` suffix, they're stock
upstream and none of this applies. That's one for whoever maintains the services.

---

## Under the hood

For anyone maintaining the fork.

`taken-media/ayon-shotgrid` sits on top of [ynput/ayon-shotgrid](https://github.com/ynput/ayon-shotgrid)
with **three commits**, merged to `develop` via PR #1 from
`feat/PIPE-79-sg-reviewables-filter-override`.

| Commit | Change |
|---|---|
| `c56b758` | Adds `VersionSyncSettings` — the two settings above, and the skip check in `AyonShotgridHub` |
| `0a8f10e` | Fixes versions deferred by the filter never uploading. A `reviewable.created` event for a version with no ShotGrid ID now creates the Version first. Also downgrades a hard failure in `update_movie_paths` to a warning so one unsynced version can't abort a run |
| `ea2f6e3` | Stops the transmitter dispatching an AYON event on every comment poll — it now only does so when comments actually synced, or on failure. Tags the version `0.6.17-tma` |

### Deploying a change

| Changed | Delivered by |
|---|---|
| `server/settings/main.py` | The addon `.zip` uploaded to AYON |
| `services/shotgrid_common/` | Service images — used by **both** leecher and transmitter |
| `services/transmitter/` | Transmitter image |

A full deploy is: build and upload the addon package, then rebuild and restart the
leecher and transmitter.

### Merging upstream

The delta is small and contained. Files to watch when rebasing:

```
server/settings/main.py
services/shotgrid_common/ayon_shotgrid_hub/__init__.py
services/shotgrid_common/utils.py
services/transmitter/transmitter/transmitter.py
```

Re-apply the `-tma` version suffix afterwards — an upstream version bump will drop it,
and without it there's no way to tell a studio build from stock.
