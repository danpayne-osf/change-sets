# Salesforce Change Set Deployment Guide

**Deployment path:** Dev sandbox → UAT / Full sandbox → Production
**Who this is for:** admins running their first deployments, and the BAs and project managers who request them.

A change set moves *configuration* from one Salesforce org to another. It does not move data, and it cannot delete anything. Everything in this guide assumes the same release travels Dev → UAT → Production, validated at each stop.

Jargon is defined the first time it appears. If you only read one page, read the [Cheat sheet](#cheat-sheet) at the end.

---

## Contents

1. [Before you start](#1-before-you-start)
2. [Stage 1 — Connect the environments](#stage-1--connect-the-environments)
3. [Stage 2 — Create the outbound change set](#stage-2--create-the-outbound-change-set)
4. [Stage 3 — Add components](#stage-3--add-components)
5. [Stage 4 — Check dependencies](#stage-4--check-dependencies)
6. [Stage 5 — Write down the manual steps](#stage-5--write-down-the-manual-steps)
7. [Stage 6 — Upload and validate](#stage-6--upload-and-validate)
8. [Stage 7 — Deploy and verify](#stage-7--deploy-and-verify)
9. [Governance and safety rails](#governance-and-safety-rails)
10. [Component reference](#component-reference)
11. [Error decoder](#error-decoder)
12. [Copy-paste templates](#copy-paste-templates)
13. [Cheat sheet](#cheat-sheet)

---

## 1. Before you start

### The five things that surprise everyone

1. **Change sets add and update. They never delete.** Removing a field, layout, or rule from Production is a manual job in Production, done by hand, after the deployment.
2. **Change sets only work between related orgs.** A Production org and the sandboxes refreshed from it. You cannot send a change set to a client's org, a partner org, or an unrelated Developer Edition org.
3. **Once you upload a change set, it is frozen.** You cannot add a forgotten component to it. You clone it, add the component, and upload again — which is why version numbers belong in the name.
4. **A deployment is all-or-nothing.** If one component fails, nothing is applied. This is a safety net, not a bug.
5. **There is no undo button.** "Rollback" means a reverse change set you prepared in advance, or manual reversal. Decide your rollback plan before you deploy, not after.

### When *not* to use a change set

| Situation | Why change sets are the wrong tool |
|---|---|
| You need to delete metadata | Change sets cannot delete. Requires manual removal or a destructive-changes deployment via the Metadata API. |
| You are renaming an API name | Deploys as a *new* component and leaves the old one behind. |
| Very large releases | Hundreds of components become unmanageable to assemble and near-impossible to review by hand. |
| The same package goes to several unrelated orgs | Change sets are locked to one org family and one target per upload. |
| You need version history or an audit trail | A change set records what moved, not what changed or who changed it. |
| Emergency fix with a guaranteed rollback | You need something you can revert with confidence. |

For those cases the team should reach for a source-driven tool — Salesforce CLI with version control, or a deployment product like Gearset or Copado. Choosing and setting that up is out of scope here; the point is to recognise when a change set is the wrong shape for the job and escalate instead of forcing it.

### Who does what

| Role | Responsibility |
|---|---|
| BA / PM / requester | Writes the deployment request: what's changing, which ticket, what "working" looks like, who signs off UAT. Does not build the change set. |
| Admin (source org) | Builds the change set, runs dependencies, records the manual steps, uploads. |
| Admin with deploy rights (target org) | Validates, deploys, verifies. In Production this should be a named, deliberately short list of people. |
| Release approver | Says go / no-go. Should not be the same person who built the change set. |

Deploying requires the **Deploy Change Sets** permission (or Modify All Data) in the *target* org. If the Deploy button is missing, that's why.

---

## Stage 1 — Connect the environments

You do this once per pair of orgs. It survives sandbox refreshes of the *source*, but a refreshed sandbox is a new org — re-check it.

**Setup path:** Setup → Quick Find → **Deployment Settings**

### The rule that catches everyone

**The receiving org grants permission, not the sending org.** Authorisation is one-directional. If you are sending Dev → UAT, you log into **UAT** and allow inbound changes from Dev.

Steps, sending Dev → UAT:

1. Log into **UAT** (the target).
2. Setup → **Deployment Settings**.
3. Find **Dev** in the list of connections. Click **Edit**.
4. Tick **Allow Inbound Changes**.
5. **Save.**

Repeat for UAT → Production before release day: log into **Production**, allow inbound changes from UAT. Do this well ahead of time, not while the release window is ticking.

### Troubleshooting this stage

- **The target org isn't in the dropdown when I try to upload.** Inbound changes aren't authorised in the target. Go and tick the box there.
- **The org isn't in Deployment Settings at all.** It isn't part of this org family. Change sets won't work — see [when not to use a change set](#when-not-to-use-a-change-set).
- **It worked last month and now it doesn't.** The target sandbox was refreshed. Re-authorise.

---

## Stage 2 — Create the outbound change set

**Setup path (source org):** Setup → Quick Find → **Outbound Change Sets** → **New**

### Name it so the next person understands it

Because uploaded change sets freeze and get cloned, names need a version number or you will end up with three change sets called "Case Changes".

```
YYYY-MM-DD_TICKET_short-description_vN

2026-08-12_CRM-4412_case-routing_v1
2026-08-12_CRM-4412_case-routing_v2
```

### Write a real description

The description field is the only context the person deploying gets. Include:

- Ticket / work item reference
- One sentence on what this changes for users
- Who requested it and who signed off UAT
- **A pointer to where the manual steps are recorded**
- Anything unusual: a feature that must be enabled first, a data load that has to follow

A description saying "changes" is how deployments get deployed by someone who doesn't know what they're deploying.

### Before you build

Check whether the source sandbox is scheduled for refresh. **A sandbox refresh destroys the change sets inside it.** Half-built change sets do not survive.

---

## Stage 3 — Add components

**Where:** open the change set → **Change Set Components** → **Add**

The picker shows **one component type at a time**. Choose the type, tick the items, click Add, and repeat. It is repetitive and there is no bulk shortcut, so work from a written list rather than memory.

### Order matters for one thing

**Add profiles and permission sets last**, after every other component is in the set. A profile in a change set does not carry the whole profile — it carries *only the settings that relate to the components in that same change set*. Add the profile first and you export an almost-empty profile.

### A working method

1. Write the list of components before you open Salesforce — from the ticket, from your own change log, from a Setup Audit Trail export if you have to reconstruct it.
2. Add them type by type, ticking them off your list.
3. Add page layouts alongside the fields that appear on them.
4. Add profiles and permission sets last.
5. Compare the component count against your list. They should match.

### Things people forget to add

- The **page layout** a new field needs to appear on
- **Record types**, when a layout assignment depends on them
- The **folder** for a new report, dashboard, or email template
- The **object** itself, when the field is on a brand-new object
- **Tabs and app assignments** for a new object
- Custom **labels** referenced by a flow or formula

---

## Stage 4 — Check dependencies

**Where:** open the change set → **View/Add Dependencies**

Salesforce inspects your components and lists other metadata they reference. This catches a large share of failures before they happen — but treat it as a helpful assistant, not an authority.

### How to use it

1. Click **View/Add Dependencies**.
2. Read the list. Do not reflexively select all.
3. Add what your change genuinely needs.
4. **Deliberately exclude** things that would overwrite unrelated Production config — most often whole profiles, or page layouts nobody asked you to touch.
5. Run it **again** after adding anything, because new components bring new dependencies.

Selecting everything is how a small field change quietly overwrites a page layout that someone else spent a week on.

### What dependency checking will not catch

- **Hard-coded IDs** in formulas, flows, email templates, or default values. IDs differ between orgs, so these break silently *after* a successful deployment.
- **Features not enabled** in the target org.
- **Data**: custom setting values, reference records, anything a user typed into a record.
- **Folders** that need to exist for reports, dashboards, and templates.
- **Users, queues, or public groups** referenced by rules, dashboards, or record owners.
- **Record-type-specific picklist availability.**
- Whether the change actually *works* for a real user.

After running dependencies, read your component list one more time against the ticket. That five-minute reread catches more than the tool does.

---

## Stage 5 — Write down the manual steps

This is the stage people skip and the stage that causes the bad Monday. Some things simply do not travel in a change set. If they aren't written down, they don't happen.

Keep **two lists**: what must happen **before** the deployment, and what must happen **after**. Both belong somewhere the deployer will actually see them — the change set description should point at them.

### Picklist values and record types

- New picklist values on a **custom** picklist field travel with the field.
- Values on **standard** picklist fields (Case Status, Opportunity Stage, Lead Source, and friends) frequently need adding by hand in the target.
- **Record type availability** is the classic trap: the value deploys, but *which record types it's available for* often needs setting manually. Check every record type on the object, not just the one you were working in.
- Assigning record types **to profiles** is manual. New record type, nobody can use it — this is usually why.
- **Translations** for new values don't come along.

### Profiles, permission sets, and field-level security

- A profile in a change set carries **only the settings for components in that change set**. It is never a full profile copy — which is a good thing, and also means gaps.
- Deploying a profile **never removes** permissions from the target. It only adds and updates.
- **Permission sets deploy in full**, but **assigning them to users is manual**, every time.
- Field-level security arrives only for fields in the set, only for profiles in the set. Verify per profile after deploying.
- **After deploying, log in as a real user on each affected profile.** Not as an admin. Admins see everything, which is why admins are the worst testers of visibility changes.

### Flows, queues, and rules

- A **flow** usually arrives **inactive**. It deploys active only if the org allows deploying flows as active (Setup → Process Automation Settings) *and* test coverage requirements are met. Assume inactive, and check.
- Activating the new flow version does not always deactivate the old one. Check which version is active, and confirm you don't have two flows racing on the same trigger.
- **Queues** deploy, but **membership** — users and public groups — is usually manual, as is confirming the supported objects.
- **Assignment and escalation rules**: the rule and its entries deploy; **activation is manual**, and only one rule per object can be active. Entry *order* is worth verifying, because order changes behaviour.
- **Lightning record page** activation and assignment normally needs redoing in the target.

### Custom settings, custom metadata, and reference data

This distinction matters and is worth learning once:

| | Definition (the structure) | Records (the values) |
|---|---|---|
| **Custom setting** | Deploys | **Does not deploy** — this is data. Enter it manually or load it. |
| **Custom metadata type** | Deploys | Records *can* be added to a change set as components. Add them deliberately and verify they arrived. |
| **Reference data in normal objects** | n/a | Never deploys. Needs a data load. |

A deployment that succeeds but does nothing is very often an empty custom setting in the target org.

### The rest of the usual suspects

- **Feature enablement** in the target org — a component that depends on a switched-off feature fails to deploy at all.
- **Folders** for reports, dashboards, and email templates, plus their sharing.
- A **dashboard's running user** must exist in the target org.
- **Scheduled jobs** and scheduled Apex need re-scheduling.
- **Connected app, named credential, and integration secrets** never travel. Re-enter them.
- **Public group membership** and sharing rule recalculation.
- **Email deliverability** settings in sandboxes.

---

## Stage 6 — Upload and validate

### Upload (source org)

Open the change set → **Upload** → choose the target org → confirm.

The moment you upload, **the change set is frozen**. No edits. Need a change? Clone, adjust, bump the version, upload again. Uploading takes a few minutes and you get an email when it lands.

### Find it in the target org

**Setup path (target org):** Setup → Quick Find → **Inbound Change Sets**

It appears under **Change Sets Awaiting Deployment**. Open it and you get three options: **Validate**, **Deploy**, **Delete**.

### Always validate first

**Validate** runs the entire deployment as a dry run and changes nothing. Same engine, same errors, no consequences. There is no good reason to skip it for a Production deployment.

Validate gives you two things: a complete error list while it's still cheap to fix, and — if you ran tests — the ability to Quick Deploy later.

### Test levels

| Option | What it runs | When to use it |
|---|---|---|
| **Default** | No tests in sandboxes. In Production, local tests run automatically if the set contains Apex. | Config-only changes to a sandbox. |
| **Run local tests** | Every test in your org, excluding managed package tests. | The standard choice for Production. |
| **Run all tests** | Everything, including managed packages. | When a managed package is genuinely implicated. Slow. |
| **Run specified tests** | Only the classes you name. | Targeted checks; must still satisfy coverage rules. |

Production enforces Apex code coverage: **at least 75% across your Apex**, and every trigger needs some coverage. Coverage failures are the single most common reason a Production deployment stops dead — which is exactly why you validate early rather than at 6pm on release day.

### What a validation does to the org

While validation or deployment runs, Salesforce briefly locks **Setup metadata** so nobody can change config mid-check. **Users keep working normally with data.** Tell your admins not to make config changes during the window; you don't need to tell users anything.

### Quick Deploy

A successful validation that ran tests unlocks **Quick Deploy** on the **Deployment Status** page. It applies the already-validated components without re-running the test suite, turning an hours-long Production deployment into minutes.

Two things to know:

- **It expires.** The commonly cited window for change sets is 96 hours (4 days); Salesforce CLI validations are documented as valid for 10 days. Don't build a plan around the exact number — validate close to release, and verify the button is still there before you promise anyone a fast deployment.
- **It disappears if relevant metadata changes**, or if another deployment happens in between. Then you re-validate. This is normal.

The practical pattern: validate during business hours when you have colleagues around to help fix errors, then Quick Deploy in the evening when the blast radius is smallest.

---

## Stage 7 — Deploy and verify

### Order of operations

1. Confirm the go / no-go decision and that UAT sign-off actually exists.
2. Complete the **pre-deployment manual steps** — feature enablement first, always.
3. Deploy: **Inbound Change Sets → Deploy**, or **Quick Deploy** from Deployment Status.
4. Watch **Setup → Deployment Status**. It runs server-side; closing your laptop does not stop it. Don't start a second deployment — the org allows one at a time.
5. Complete the **post-deployment manual steps**.
6. Smoke test (below).
7. Announce, and record the deployment in your release log.

### Smoke test properly

Deployment success means "the metadata was accepted". It does not mean "it works".

- Log in as a **real user** on each affected profile, not as an admin.
- Do the actual thing the change was for, end to end.
- Confirm new fields are **visible and editable** to the people who need them.
- Trigger any new automation once and confirm it fired — then confirm no *old* automation fired alongside it.
- Check a report or list view that touches the change.

### If it goes wrong

- **Failed deployment:** nothing was applied. Read the error list, fix in the source org, clone the change set, bump the version, re-validate. Annoying, not dangerous.
- **Succeeded but broken:** this is the real risk. Deactivate the offending automation first — that's the fastest way to stop harm — then work through your prepared reversal.
- **Rollback:** there is no undo. Your options are the reverse change set you built in advance, or manual reversal from your own notes. Manual reversal without notes, under pressure, is how small incidents become long ones.

For anything that touches Production widely, build the reverse change set *before* you deploy. It takes twenty minutes and you will be very glad of it perhaps once a year.

---

## Governance and safety rails

### Who can deploy to Production

Keep the list short and named. Everyone else requests. The person who built the change set should not be the only reviewer of it, and ideally not the person who approves it.

### Deployment windows

Pick windows deliberately and write them down:

- **Good:** early-to-mid week, outside business hours, with the deployer available afterwards.
- **Bad:** Friday afternoon. Month-end and quarter-end for sales-heavy orgs. The day before a holiday. Any time nobody who understands the change is reachable.
- **Freeze periods:** agree in advance when nothing ships — quarter-end close, major campaigns, the week a Salesforce seasonal release hits your org.

Validation during business hours, deployment out of hours, is the pattern that gets you both help and safety.

### The go / no-go checklist

Nothing ships to Production until all of these are true:

- [ ] UAT sign-off recorded, by name
- [ ] Validated successfully against Production
- [ ] Pre- and post-deployment manual steps written down, in order, by someone other than the deployer where possible
- [ ] Rollback plan decided and, for wide-reaching changes, prepared
- [ ] Deployment window agreed and inside policy
- [ ] Named approver has said go
- [ ] Someone who understands the change is available afterwards

### Keep a release log

One row per deployment: date, change set name and version, ticket, who deployed, who approved, manual steps completed, outcome. When something breaks three weeks later, this is the first thing you'll want and the last thing you'll have if you didn't start it.

---

## Component reference

What travels, and what to watch for.

| Component | Travels with it | Watch out for |
|---|---|---|
| **Custom field** | Custom picklist values on that field | Needs the page layout to be visible; field-level security per profile; a required field fails if it isn't on a deployed layout |
| **Custom object** | Nothing automatically | Add fields, layouts, tab, and app assignment separately |
| **Page layout** | Nothing | *Assignments* per profile and record type need the profile and record type in the set, or manual work |
| **Record type** | Nothing | Picklist value availability per record type is often manual; profile assignment is manual; layout assignment per record type is separate |
| **Global value set** | Its values | Standard picklist field values frequently need manual addition |
| **Profile** | Only settings for components in this same set | Never a full profile copy; add profiles last; deploying never removes permissions |
| **Permission set** | Full contents | Assigning it to users is always manual |
| **Flow** | Nothing | Arrives inactive unless the org setting allows active deployment and coverage is met; deactivate the superseded version |
| **Apex class / trigger** | Nothing | 75% coverage required for Production; every trigger needs coverage; Production runs tests |
| **LWC / Aura component** | Nothing | Check API versions and referenced Apex, objects, and fields |
| **Validation rule** | Nothing | Fails if it references a field not in the set |
| **Email template** | Nothing | The folder must exist; merge fields must resolve in the target |
| **Report / dashboard** | Nothing | Folder must exist; folder sharing is manual; a dashboard's running user must exist in the target |
| **List view** | Nothing | Sharing and visibility settings are worth re-checking |
| **Custom setting** | Definition only | **Values are data and do not deploy** |
| **Custom metadata type** | Definition; records can be added as components | Add records deliberately, then verify they arrived |
| **Queue** | Nothing | Membership is manual; confirm supported objects |
| **Assignment / escalation rule** | Rule entries | Activation is manual; one active rule per object; entry order changes behaviour |
| **Lightning record page** | Nothing | Activation and assignment usually manual |
| **Sharing rule** | Nothing | Recalculation and referenced groups need checking |
| **Anything you want deleted** | — | **Not possible.** Manual removal in the target |

---

## Error decoder

### "The target org isn't in the dropdown when I upload"

Inbound changes aren't authorised in the target org. In the **target**: Setup → Deployment Settings → Edit the connection → tick **Allow Inbound Changes** → Save.

### "This change set is closed / cannot be edited"

Expected. Uploading freezes a change set. **Clone** it, make your change, bump the version in the name, upload again.

### "In field: field — no CustomField named X found"

A referenced field isn't in the set and doesn't exist in the target. Add it, re-run dependencies, clone and re-upload.

### "Field X must be included in the page layout" (or required-field errors)

A required field arrived without a layout that displays it. Add the page layout to the set, or reconsider making it required at field level.

### "An object of type Layout was named in layout assignment, but was not found"

The layout, or the record type the assignment depends on, is missing from the set. Add both.

### "Not available for deploy for this organization"

The target org doesn't have that feature enabled, or the edition differs. Enable the feature in the target **first**, as a pre-deployment manual step, then deploy.

### "Average test coverage across all Apex classes is XX%, at least 75% is required"

Org-wide Apex coverage is below the Production threshold. This needs developer work in the source org — writing or extending tests — and is not something to solve on release night. Catch it by validating early.

### "Test coverage of selected Apex trigger is 0%"

That trigger has no test coverage at all. Same answer: developer work, done ahead of time.

### "Another deployment is already in progress"

One deployment at a time per org. Check Setup → Deployment Status and wait for the running job. Don't cancel someone else's deployment without asking them.

### "Insufficient access rights on cross-reference id"

Something references a record, user, or owner that doesn't exist in the target — commonly a dashboard running user, queue member, or folder owner. Create or remap it in the target.

### "Duplicate value found: <field> duplicates value on record"

A component with that API name already exists in the target, or a uniqueness constraint conflicts. Compare against the target org before re-uploading.

### "Entity is not org-accessible" / "Entity 'X' not found"

The object or feature the component depends on is missing or not enabled in the target.

### The Quick Deploy button has disappeared

The validation expired, or metadata changed, or another deployment ran in between. Re-validate. Nothing is wrong.

### It deployed successfully, but users can't see the change

The most common outcome of a "successful" deployment, and almost always one of:

1. Field-level security wasn't deployed or the profile wasn't in the set
2. The field isn't on the page layout the user actually sees
3. A permission set was deployed but never assigned to anyone
4. A record type isn't assigned to their profile
5. A flow arrived inactive
6. A custom setting is empty in the target

Work down that list in order and you'll find it.

---

## Copy-paste templates

### A. Deployment request

For BAs, PMs, and anyone asking for a deployment.

```
DEPLOYMENT REQUEST

Ticket / work item:
Requested by:
Target environment:        Dev → UAT  /  UAT → Production
Requested window:

WHAT IS CHANGING
(One paragraph in plain language. What will users notice?)

WHY NOW
(Business driver, any deadline, and what happens if it slips.)

COMPONENTS I KNOW ABOUT
(Best-effort list. The admin will confirm and complete it.)
-
-

WHO IS AFFECTED
Profiles / teams:
User communication needed?   Yes / No
Training or documentation needed?   Yes / No

HOW WE KNOW IT WORKED
(Specific, testable. "A support agent can set Case Priority to Urgent
and the case routes to the Tier 2 queue within a minute.")
-

UAT SIGN-OFF
Signed off by:
Date:
Evidence / link:

RISK
Reversible?   Yes / No / Partly
If this breaks in Production, the immediate mitigation is:
```

### B. Manual steps runbook

Attach this to every deployment. Point at it from the change set description.

```
MANUAL STEPS — <change set name and version>
Deployer:                          Date:

BEFORE DEPLOYMENT
#   Step                                 Where            Owner   Done
1   Enable <feature>                     Setup → ...              [ ]
2   Confirm reverse change set prepared                           [ ]
3
4

DEPLOY
    Validate against target                                       [ ]
    Validation result: PASS / FAIL      Errors resolved: [ ]
    Deploy / Quick Deploy               Time started:
    Deployment Status: SUCCEEDED / FAILED

AFTER DEPLOYMENT
#   Step                                 Where            Owner   Done
1   Activate flow <name>, deactivate v<n>                         [ ]
2   Assign permission set <name> to <group>                       [ ]
3   Add picklist values to standard field <name>                  [ ]
4   Make record type <name> available to profiles <...>           [ ]
5   Populate custom setting <name>                                [ ]
6   Add members to queue <name>                                   [ ]
7   Activate assignment rule <name>, confirm entry order          [ ]
8   Re-schedule job <name>                                        [ ]
9

SMOKE TEST
Logged in as real user on profile:               Result:
Logged in as real user on profile:               Result:
End-to-end scenario tested:                      Result:
Old automation confirmed not firing:             Result:

OUTCOME
Complete / Rolled back / Partial — notes:
```

### C. Release note

```
RELEASE NOTE — <date>

WHAT CHANGED
<One or two sentences a non-Salesforce colleague would understand.>

WHO IT AFFECTS
<Teams and profiles.>

WHAT YOU NEED TO DO
<Nothing, or the specific action.>

DETAIL
Ticket:
Change set:
Deployed by:                Approved by:
Manual steps runbook:

IF SOMETHING LOOKS WRONG
Contact:                    Known issues:
```

---

## Cheat sheet

### Setup paths

| Purpose | Path |
|---|---|
| Authorise inbound changes (**in the target org**) | Setup → Deployment Settings |
| Build and upload (**source org**) | Setup → Outbound Change Sets |
| Validate and deploy (**target org**) | Setup → Inbound Change Sets |
| Monitor, and Quick Deploy | Setup → Deployment Status |
| Allow flows to deploy active | Setup → Process Automation Settings |

### The seven stages

1. **Connect** — in the *target* org, allow inbound changes from the source. One-directional.
2. **Create** — outbound change set in the source, versioned name, real description.
3. **Add** — one component type at a time. **Profiles and permission sets last.**
4. **Dependencies** — View/Add Dependencies. Add what you need, exclude what would overwrite unrelated config, re-run after changes.
5. **Manual steps** — two written lists: before and after.
6. **Upload and validate** — upload freezes the set. Validate before every Production deployment.
7. **Deploy and verify** — pre-steps, deploy, post-steps, smoke test as a real user, announce, log it.

### Golden rules

1. Change sets **add and update only**. Never delete.
2. **The receiving org authorises inbound.**
3. **Uploading freezes** the change set. Clone to change it.
4. **Add profiles last** — they only carry settings for components in the set.
5. **Validate first, every time,** into Production.
6. **All-or-nothing.** One failure means nothing deployed.
7. **There is no undo.** Prepare the reversal before you need it.
8. **Test as a real user, not as an admin.**

### Never travels in a change set

- Data of any kind, including **custom setting values** and reference records
- Deletions
- Permission set **assignments** to users
- **Record type assignment** to profiles
- Standard picklist values (usually)
- Flow **activation** (usually) — and the old version stays active
- Queue **membership**; assignment rule **activation**
- Integration secrets, named credentials, scheduled jobs
- Report, dashboard, and template **folders** you forgot to include

### Go / no-go

- [ ] UAT signed off, by name
- [ ] Validated against the target
- [ ] Manual steps written, before and after
- [ ] Rollback plan decided
- [ ] Window agreed, approver said go
- [ ] Someone who understands the change is around afterwards

### Deployment succeeded but nobody sees it — check in this order

1. Field-level security / profile not in the set
2. Field not on the layout the user sees
3. Permission set deployed but not assigned
4. Record type not assigned to the profile
5. Flow inactive
6. Custom setting empty
