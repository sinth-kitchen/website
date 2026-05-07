# SINTH website (sinthkitchens.com)

This file is auto-loaded by Claude Code. It exists to give a complete pickup-from-here context — anyone (Claude or human) opening this repo cold should be able to make safe decisions without re-discovering the wider system.

---

## What this is

The single-page **SINTH marketing site** at `https://sinthkitchens.com` and `https://www.sinthkitchens.com`.

- One file — `index.html` (~32 KB, inline CSS, no JS framework, no build step).
- Three hero photos under `images/*.avif` (PLAN / SENSE / DECIDE sections).
- That's it. Don't add a bundler, framework, or build pipeline. The simplicity is the point.

If something feels like it wants Vite/Webpack/etc., push back — the dashboard project (`sinth-kitchen/dashboard`) is the place for app-level complexity.

---

## How it ships

**Hosted on Cloudflare Pages.** Push to `main` → Cloudflare auto-builds (no build command — it just serves the repo root) → live in ~30s.

| Layer | Value |
|---|---|
| Pages project | `sinth-website` |
| Pages subdomain | `https://sinth-website.pages.dev` |
| Custom domain (apex) | `https://sinthkitchens.com` |
| Custom domain (www) | `https://www.sinthkitchens.com` |
| Production branch | `main` |
| Build command | (none) |
| Build output dir | `/` (repo root) |

DNS for both apex and www is a CNAME to `sinth-website.pages.dev`, proxied through Cloudflare. Email records (Google Workspace MX, SPF, DKIM TXT) on the `sinthkitchens.com` zone are **untouched and must remain so** — never delete or "tidy up" MX/TXT records on this zone.

---

## Notifications

A single Cloudflare Notifications policy (`Sinth Pages deploys (#tech)`) posts to Slack `#tech` on **success / failure** for both this site and the dashboard project. "Started" events are suppressed to cut noise.

If you need to add a third Pages project to the same notifications, append its project ID to the policy's `filters.project_id` array via the Cloudflare API.

The Slack side is an incoming webhook from a workspace app named "Cloudflare Pages" in the SINTH workspace.

**A Cloudflare Worker reformatter was tried and explicitly torn down on 2026-05-06.** We had a Worker (`sinth-deploy-notifier`) that intercepted Cloudflare's verbose alert payload and re-emitted a one-liner to Slack. It worked, but the trade-off — yet another moving part to maintain — wasn't worth the cosmetic improvement. **Don't propose rebuilding it** unless deploy noise becomes a concrete operational problem; the standard Cloudflare format is fine.

---

## Credentials

All in **`.env`** (gitignored, alongside this file). The group-level **`../.env`** in `Sinth/` holds the same credentials shared across sibling projects (dashboard, audio-pipeline, kitchen-recorder, video-pipeline).

Key values you'll find there:

- `CLOUDFLARE_API_TOKEN` — scoped to Pages:Edit, DNS:Edit, Zone:Read, Notifications:Edit, Workers:Edit. **Expires 2027-03-31** — set a calendar reminder to rotate before then.
- `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ZONE_ID_SINTHKITCHENS`
- `CLOUDFLARE_PAGES_PROJECT_ID` — `9b93d359-066f-4509-b238-7b6c3c990fa4`
- `CLOUDFLARE_NOTIF_POLICY_ID`, `CLOUDFLARE_NOTIF_WEBHOOK_ID`
- `SLACK_WEBHOOK_TECH` — incoming webhook to `#tech`
- `SLACK_BOT_TOKEN` (`xoxb-...`) — bot user `cloudflare_pages` with `chat:write`. Use for ad-hoc Slack posts; bot must be `/invite`d to private channels.

Never echo secret values back to the user, never paste them into commit messages or PR descriptions, and never write them to non-`.env` files.

---

## Repo conventions

- One file per page if we ever grow beyond a single page. No SPA/router.
- Static assets in `/images/` (or another flat folder); reference by relative path.
- Don't add a `package.json` unless we *need* one — it'll bait future contributors into adding builds.
- Keep `index.html` self-contained (inline `<style>`). External fonts (Google Fonts) are fine.
- AVIF for photos (smaller than JPEG/PNG at the same quality).

---

## History — how we got here

**Why this section is long:** the deploy/domain setup has more story than the code does, and it's the part most likely to bite a future contributor.

### 2026-05-06 — initial setup
- Imported HTML from upstream `tjread/sinth-website` (single file, "Add files via upload" commit). The file was named `SINTH_homepage_v01.html`; **renamed to `index.html`** so Cloudflare Pages serves it at `/`.
- Created public GitHub repo `sinth-kitchen/website`.
- Created Cloudflare Pages project (initially named `website` — see rename below).
- Apex (`sinthkitchens.com`) had been pointing at a **broken GoDaddy Website Builder origin** (HTTP 522). The 2 A records were deleted and replaced with a CNAME to the Pages subdomain. www had been a CNAME to apex; updated similarly. Email DNS records preserved.
- Three referenced images (`plan-chef-writing-menu.avif`, `sensor-array-stainless-brigade.avif`, `decide-analytics-moment.avif`) were **missing from upstream's upload**. Generated dark-themed placeholder AVIFs labeled PLAN / SENSE / DECIDE so the layout didn't break.
- Tripped over a Cloudflare gotcha: the Cloudflare Pages GitHub App is installed on the `sinth-kitchen` org with `repository_selection: "selected"`. New repos won't auto-deploy until they're explicitly added to that list. Fix: https://github.com/organizations/sinth-kitchen/settings/installations/125434845 → add the repo, or switch to "All repositories".

### 2026-05-06 — Cloudflare Worker experiment, then revert
- Built `sinth-deploy-notifier` Cloudflare Worker to reformat the verbose Slack alerts into a one-liner. Worked end-to-end (got `✅ website → production deploy succeeded (a4b6ffc) · live: https://sinthkitchens.com · view build`).
- Then deleted everything because the user prioritized zero-maintenance over message prettiness.
- Lesson preserved: when creating a Cloudflare Notifications webhook destination that should route through a custom transformer, **create it with the transformer URL from the start.** A destination created with a `hooks.slack.com` URL gets `type: "slack"` and the URL field becomes effectively immutable; PUT requests return success but silently keep the Slack URL.

### 2026-05-07 — Pages project rename
- Renamed Pages project `website` → `sinth-website` (recreated, since Pages projects can't be renamed in place; involved detaching custom domains, creating new project, reattaching domains, deleting old project, updating DNS CNAMEs and the notification policy filter).
- **GitHub repo + local dir intentionally NOT renamed** — they stay as plain `website`. The dashboard is `sinth-kitchen/dashboard` on GitHub but `sinth-dashboard` on Pages; this matches that pattern (apps get the `sinth-` prefix where they show up as project names; repo/dir names are the kebab description).
- Real images delivered by Tim and committed (`ee99de5`), replacing placeholders.

---

## Operational quick reference

```bash
# All commands assume CLOUDFLARE_API_TOKEN and CLOUDFLARE_ACCOUNT_ID
# are in your environment (source .env or grep them out).

# List recent deployments
curl -sS -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/sinth-website/deployments?per_page=5"

# Trigger a manual deploy (without a git push)
curl -sS -X POST -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/sinth-website/deployments"

# Inspect a custom domain's status
curl -sS -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/sinth-website/domains/sinthkitchens.com"

# Check what's actually being served (skip cache)
curl -sSI "https://sinthkitchens.com/?cb=$(date +%s)"
```

---

## Sibling projects

In `/Users/tuf/SynologyDrive/AI Projects/Sinth/`:

- `dashboard/` — the React app dashboard, deploys to `mvp.sinthkitchens.com`. Behind Cloudflare Access (login required). Same notification policy.
- `sinth-MVP-dashboard/` — local clone of `tjread/sinth-MVP-dashboard`. Reference only; **NOT** the canonical dashboard repo.
- `audio-pipeline/`, `kitchen-recorder/`, `video-pipeline/` — backend / hardware fleet projects, share the group-level `.env`.

---

## When to reach for the dashboard's docs instead

If the question is about Cloudflare Notifications API quirks, the GitHub App selected-repos trap, the webhook destination type immutability, or general Cloudflare Pages deploy plumbing — those gotchas all apply here too and are documented (in more detail) in `dashboard/CLAUDE.md` if it exists, or in the Claude Code memory at `~/.claude/projects/-Users-tuf-SynologyDrive-AI-Projects-Sinth-dashboard/memory/`.
