# Railway: auto-updating hermes while keeping user data

This fork (`chrisamber/hermes-agent`) runs on Railway as the **hermes-agent** service
and is a fork of **NousResearch/hermes-agent**. This doc explains how upstream updates
reach Railway and, critically, **why user data survives a redeploy**.

## Where user data lives

- The Dockerfile sets `HERMES_HOME=/opt/data` and the `hermes` user's home is `/opt/data`.
- Railway mounts a **persistent volume at `/opt/data`** on the hermes-agent service.
- Therefore **all user state lives on that one Railway volume**, and it persists across
  every deploy automatically — as long as the two rules below hold.

> The separate `MongoDB` service has its own volume and is not touched by app deploys.

## The two rules that keep data safe

1. **Never delete the `/opt/data` volume** and never change its mount path. The volume —
   not the container — is what holds the data. Containers come and go on each deploy; the
   volume stays.
2. **The Dockerfile must NOT contain a `VOLUME` directive** (e.g. `VOLUME /opt/data`).
   A Docker `VOLUME` directive makes the image declare an *anonymous* volume that shadows
   Railway's persistent mount, so each deploy starts empty. Removing that directive is the
   single patch this fork carries on top of upstream (commit "remove unsupported VOLUME
   instruction from Dockerfile"). It must survive every upstream sync.

## How updates flow (PR-gated)

```
daily cron / manual dispatch
  -> .github/workflows/sync-upstream.yml
  -> fetch NousResearch/hermes-agent@main
  -> replay this fork's Dockerfile patch on top (cherry-pick)
  -> GUARD: abort if a VOLUME directive is present
  -> open / update PR  sync/upstream -> main
  -> YOU review the PR (esp. the Dockerfile) and merge with "Create a merge commit"
  -> Railway (connected to main) auto-builds and deploys
  -> /opt/data volume persists -> user data retained
```

Nothing deploys unattended. If the patch can't be replayed (Dockerfile conflict) or a
`VOLUME` directive reappears, the workflow **fails, opens/updates a tracking issue, and
does not open a deployable PR**.

## Railway side (one-time setup)

1. Connect this repo to the **hermes-agent** service with a deploy trigger on `main`
   (Railway dashboard -> hermes-agent -> Settings -> Source -> Connect Repo, branch `main`).
   This is what makes "merge the PR -> deploy" work.
2. Leave the `/opt/data` volume attached to hermes-agent. Do not delete it.
3. Optional resilience: set a health check path and overlap so the old instance keeps
   serving until the new one is healthy.

## First catch-up is special

This fork is thousands of commits behind upstream. The **first** big update should be done
manually with a fresh backup of `/opt/data` in hand — config/layout may have changed across
that range in ways beyond the volume rules. Once caught up, the daily workflow handles small
incremental syncs.

## Recommended backup (belt-and-suspenders)

Add a Railway `preDeployCommand` on hermes-agent (or a scheduled job) that snapshots
`/opt/data` to a Railway bucket before each deploy, so even a bad release can't permanently
lose data. (Not yet configured.)
