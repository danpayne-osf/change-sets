# Salesforce Change Set Deployment Guide

Deployment reference for our Dev → UAT / Full → Production path. Written for admins running their first deployments and for the BAs and PMs who request them.

## What's here

| File | What it is |
|---|---|
| `index.html` | Landing page for the published site. Links to the three documents. |
| `change-set-guide.html` | Interactive handbook. Seven stages with a tickable checklist that remembers progress, plus searchable component and error references, templates, and governance. Open it in a browser — no build step, no dependencies. |
| `change-set-deployment-guide.md` | The same content as text. Use this for reading in the repo, diffing changes, and copying into tickets or wiki pages. |
| `change-set-cheat-sheet.html` | One-page summary. Prints to a single sheet — click **Print**. |

## Publishing it

The docs are plain static HTML — no build step, no dependencies.

1. **Settings → Pages**
2. Under **Build and deployment → Source**, choose **Deploy from a branch**
3. Branch `main`, folder `/ (root)`, then **Save**

Live in a few minutes at `https://<org>.github.io/<repo>/`. `index.html` is the entry point; `.nojekyll` tells Pages to serve the files as-is rather than running them through Jekyll.

If this repo grows to hold other things, move the docs into `/docs` and select that folder instead.

**Before publishing, decide who should be able to read this.** A Pages site is public on the internet by default *even when the repository is private* — site visibility and repo visibility are separate settings. Publishing privately, so only people with repo read access can reach it, requires GitHub Enterprise Cloud. Pages from a private repo at all requires Pro, Team, or Enterprise.

Nothing here is a credential, but it does describe our environments, process, and approval chain. Make that a decision rather than a discovery.

## How to use it

Working a deployment: open `change-set-guide.html`, start at **The run**, tick as you go. Hit an error: **Error decoder**, paste a fragment of the message. Unsure whether something travels in a change set: **Components**, search by name or symptom.

Requesting a deployment rather than performing one: read **Before you start**, then use template A from the **Templates** tab.

## Keeping it accurate

`change-set-deployment-guide.md` is the source of truth for wording. When something changes, edit the markdown first, then mirror it into the two HTML files.

Salesforce moves Setup paths and limits between releases. If the guide disagrees with the org in front of you, the org is right — open a PR and fix the guide.

Two things deliberately left loose, worth pinning down with our own observed behaviour:

- **Quick Deploy expiry.** Salesforce's classic figure for change sets is 96 hours; current CLI docs say 10 days. The guide tells people to confirm the button is still available instead of trusting a number.
- **Named roles and windows.** The governance section describes the shape of approvals, deployment windows, and freeze periods without naming anyone. Add our actual approvers, windows, and freeze dates.

## Conventions this guide assumes

- Change set naming: `YYYY-MM-DD_TICKET_short-description_vN`
- Every deployment has a written manual-steps runbook, linked from the change set description
- Every Production deployment is validated first
- Every deployment gets a row in the release log