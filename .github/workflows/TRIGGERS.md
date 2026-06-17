# GitHub Actions Triggers — Learning Report

A walkthrough of **every** `on:` trigger in [`learn-trigger.yml`](./learn-trigger.yml), tested
one at a time on this repo. Each trigger was armed by uncommenting it (one active at a time) and
pushing, then fired with the GitHub CLI / REST / GraphQL API, then verified in the Actions log.

- **Repo:** `donaldraph/actions-1`
- **Test branch:** `trigger-lab` — temporarily set as the repo **default branch** so triggers
  would "arm" there while keeping `main` free of ~30 test commits. Restored to `main` at teardown
  (see [Default-branch swap & teardown](#default-branch-swap--teardown)).
- **Workflow under test:** `Trigger Lab` (`.github/workflows/learn-trigger.yml`). Its `show-trigger`
  job prints `github.event_name` and dumps the full event payload.
- **Legend:** ✅ self-tested & verified · 🟡 manual/external (could not be fired by a single account
  / from the CLI) · 📄 doc-only (armed & YAML-valid, but not fireable in this environment)

### How triggers "arm"
Most events only start a workflow if the workflow file containing that `on:` trigger already exists
**on the repo's default branch** at the moment the event happens. That's the whole reason for the
`trigger-lab`-as-default trick: every commit below was pushed to `trigger-lab` (the temporary
default) *before* the firing action, so the trigger was armed.

---

## Summary table

| Trigger | Group | Result | Run / Note |
|---|---|---|---|
| workflow_dispatch | manual | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27648914931) |
| push | code | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27648947251) |
| pull_request | code | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649073171) |
| pull_request_target | code | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649107834) |
| pull_request_review | code | 🟡 | needs a 2nd account |
| pull_request_review_comment | code | 🟡 | needs a 2nd account |
| pull_request_review_thread | code | 🟡 | needs a 2nd account |
| merge_group | code | 📄 | needs a merge queue |
| schedule | timer | 📄 | cron can't be forced |
| create | lifecycle | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649474033) |
| delete | lifecycle | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649496317) |
| issues | issues | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649581354) |
| issue_comment | issues | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27649608097) |
| label | issues | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27658330314) |
| milestone | issues | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27658347666) |
| discussion | discussions | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27658415180) |
| discussion_comment | discussions | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659231751) |
| release | releases | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659253703) |
| registry_package | packages | 📄 | needs `write:packages` + a package publish |
| deployment | deployments | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659334363) |
| deployment_status | deployments | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659356382) |
| check_run | checks | 📄 | needs a GitHub App |
| check_suite | checks | 📄 | needs a GitHub App |
| status | checks | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659586192) |
| fork | repo meta | 📄 | needs another account |
| watch | repo meta | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659623724) |
| public | repo meta | 📄 | repo already public |
| gollum | repo meta | 🟡 | wiki repo not provisioned via API |
| page_build | repo meta | 📄 | needs Pages enabled |
| branch_protection_rule | repo meta | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659775784) |
| workflow_run | chaining | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659834160) |
| workflow_call | chaining | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659904124) (nested) |
| repository_dispatch | chaining | ✅ | [run](https://github.com/donaldraph/actions-1/actions/runs/27659945079) |

**21 fired & verified · 4 manual/external · 8 doc-only.**

---

## Code events

### `workflow_dispatch` ✅
- **What it does:** Adds a manual "Run workflow" button (and optional input form) in the Actions UI.
- **Fires when:** A human/API requests a manual run.
- **What I did:** `gh workflow run "Trigger Lab" --ref trigger-lab -f who=Donald`
- **Result:** Ran — `event_name = workflow_dispatch`.
  [run 27648914931](https://github.com/donaldraph/actions-1/actions/runs/27648914931)

### `push` ✅
- **What it does:** Runs on commits pushed to matching refs; supports `branches`/`tags`/`paths` filters.
- **Fires when:** A `git push` lands on a matching branch/tag.
- **What I did:** Added `trigger-lab` to the branch filter (`branches: [main, dev, trigger-lab]`),
  committed, and pushed — the arming push *is* the firing push.
- **Result:** Ran — `event_name = push`.
  [run 27648947251](https://github.com/donaldraph/actions-1/actions/runs/27648947251)

### `pull_request` ✅
- **What it does:** Runs on PR activity (opened/synchronize/reopened…) targeting matching branches.
  Runs against a **merge ref** using the workflow from the **base** branch.
- **Fires when:** A PR is opened/updated against a matching base branch.
- **What I did:** `branches: [trigger-lab]`, then a feature branch + `gh pr create --base trigger-lab`
  (PR #2).
- **Result:** Ran — `event_name = pull_request`.
  [run 27649073171](https://github.com/donaldraph/actions-1/actions/runs/27649073171)

### `pull_request_target` ✅
- **What it does:** Like `pull_request`, but runs in the **base repo's context with secrets** and uses
  the workflow/code from the base branch. Designed for trusted automation on fork PRs — powerful and
  easy to misuse.
- **Fires when:** Same PR activity types as `pull_request`.
- **What I did:** Pushed a new commit to PR #2's branch (a `synchronize`).
- **Result:** Ran — `event_name = pull_request_target`.
  [run 27649107834](https://github.com/donaldraph/actions-1/actions/runs/27649107834)

### `pull_request_review` 🟡 (manual/external)
- **What it does:** Runs when a review is submitted / edited / dismissed.
- **Fires when:** A reviewer submits an approval, "request changes", or a comment review.
- **What I did / why it didn't fire:** On your **own** PR, GitHub blocks *approve* and *request
  changes* (`"Can not approve your own pull request"`), and a self-authored **COMMENT** review does
  **not** raise the event. I submitted a COMMENT review on PR #2 → review recorded, **0** workflow runs.
- **To fire it (human):** Have a *second* account review the PR (approve/request-changes/comment).

### `pull_request_review_comment` 🟡 (manual/external)
- **What it does:** Runs when a comment is added to a specific line of a PR's diff.
- **Fires when:** Someone comments on a line in the "Files changed" view.
- **What I did / why it didn't fire:** Created a line comment on PR #2 via
  `POST /pulls/2/comments` → comment created, **0** workflow runs in this single-account setup.
- **To fire it (human):** A reviewer (ideally a second account) leaves a line comment on the diff.

### `pull_request_review_thread` 🟡 (manual/external) — *added; was missing from the file*
- **What it does:** Runs when a PR review thread is marked **resolved / unresolved**.
- **Fires when:** Someone resolves/unresolves a conversation thread on a PR.
- **What I did / why it didn't fire:** Resolved the thread via the GraphQL `resolveReviewThread`
  mutation (`isResolved = true`) → **0** workflow runs solo.
- **To fire it (human):** Resolve/unresolve a review thread (review threads generally need a second
  reviewer's comment to exist meaningfully).
- **Note:** This event is in GitHub's official "events that trigger workflows" list but was absent
  from the original YAML — I added it (commented) in the CODE EVENTS section.

### `merge_group` 📄 (doc-only)
- **What it does:** Runs the required checks for a PR that a merge queue has batched into a temporary
  `merge group` branch.
- **Fires when:** A PR is added to GitHub's **merge queue**.
- **Why not self-tested:** Requires a configured merge queue (a branch ruleset with "Require merge
  queue"), which isn't set up here. YAML armed and valid.

---

## Timer

### `schedule` 📄 (doc-only)
- **What it does:** Runs the workflow on a cron timer (always UTC). Only runs from the **default
  branch**.
- **Fires when:** The cron expression matches the current UTC time (e.g. `0 9 * * 1` = Mondays 09:00 UTC).
- **Why not self-tested:** Cron cannot be force-triggered on demand; GitHub also delays scheduled runs
  under load. YAML armed and valid. To observe it, leave it active and wait for the next tick.

---

## Branch / tag lifecycle

### `create` ✅
- **What it does:** Runs when a branch or tag is created. (No `types`; no `branches` filter support.)
- **What I did:** `git tag lab-v1 && git push origin lab-v1`.
- **Result:** Ran — `event_name = create`.
  [run 27649474033](https://github.com/donaldraph/actions-1/actions/runs/27649474033)

### `delete` ✅
- **What it does:** Runs when a branch or tag is deleted.
- **What I did:** `git push origin :refs/tags/lab-v1` (delete the tag).
- **Result:** Ran — `event_name = delete`.
  [run 27649496317](https://github.com/donaldraph/actions-1/actions/runs/27649496317)

---

## Issues & project management

### `issues` ✅
- **What it does:** Runs on issue lifecycle activity (opened/edited/closed/labeled…). Here filtered to
  `[opened, labeled]`.
- **What I did:** `gh issue create` (issue #3).
- **Result:** Ran — `event_name = issues`.
  [run 27649581354](https://github.com/donaldraph/actions-1/actions/runs/27649581354)

### `issue_comment` ✅
- **What it does:** Runs when a comment is created on an issue **or a PR** (PRs are issues too).
- **What I did:** `gh issue comment 3`.
- **Result:** Ran — `event_name = issue_comment`.
  [run 27649608097](https://github.com/donaldraph/actions-1/actions/runs/27649608097)

### `label` ✅
- **What it does:** Runs when a label is created / edited / deleted.
- **What I did:** `gh label create lab-label`.
- **Result:** Ran — `event_name = label`.
  [run 27658330314](https://github.com/donaldraph/actions-1/actions/runs/27658330314)

### `milestone` ✅
- **What it does:** Runs when a milestone is created / closed / edited / etc.
- **What I did:** `POST /repos/.../milestones` (`title: "Lab milestone"`).
- **Result:** Ran — `event_name = milestone`.
  [run 27658347666](https://github.com/donaldraph/actions-1/actions/runs/27658347666)

---

## Discussions

> Discussions were **disabled** on the repo; I enabled them with
> `gh repo edit --enable-discussions` (reversible).

### `discussion` ✅
- **What it does:** Runs when a discussion is created / answered / category-changed / etc.
- **What I did:** GraphQL `createDiscussion` in the "General" category (discussion #4).
- **Result:** Ran — `event_name = discussion`.
  [run 27658415180](https://github.com/donaldraph/actions-1/actions/runs/27658415180)

### `discussion_comment` ✅
- **What it does:** Runs when a comment is added to a discussion.
- **What I did:** GraphQL `addDiscussionComment` on discussion #4.
- **Result:** Ran — `event_name = discussion_comment`.
  [run 27659231751](https://github.com/donaldraph/actions-1/actions/runs/27659231751)

---

## Releases, packages & deployments

### `release` ✅
- **What it does:** Runs on release activity; here filtered to `[published]`.
- **What I did:** `gh release create lab-rel-v1 --target trigger-lab` (creates a tag + published release).
- **Result:** Ran — `event_name = release`.
  [run 27659253703](https://github.com/donaldraph/actions-1/actions/runs/27659253703)

### `registry_package` 📄 (doc-only)
- **What it does:** Runs when a package is published/updated to GitHub Packages.
- **Why not self-tested:** Requires the `write:packages` scope (the CLI token here has
  `repo, workflow, gist, read:org`) **and** actually publishing a package (npm/container/etc.).
  YAML armed and valid.

### `deployment` ✅
- **What it does:** Runs when a deployment is created (via API or another action).
- **What I did:** `POST /repos/.../deployments` with `required_contexts: []` (so it isn't blocked by
  missing checks).
- **Result:** Ran — `event_name = deployment`.
  [run 27659334363](https://github.com/donaldraph/actions-1/actions/runs/27659334363)

### `deployment_status` ✅
- **What it does:** Runs when a deployment's status changes (success/failure/…).
- **What I did:** `POST /repos/.../deployments/<id>/statuses` with `state=success`.
- **Result:** Ran — `event_name = deployment_status`.
  [run 27659356382](https://github.com/donaldraph/actions-1/actions/runs/27659356382)

---

## Checks & commit status

### `check_run` 📄 (doc-only)
- **What it does:** Runs when a check run is created/completed/rerequested/etc.
- **Why not self-tested:** Check **runs** can only be created by a **GitHub App** (the Checks API
  rejects a normal user PAT). YAML armed and valid.

### `check_suite` 📄 (doc-only)
- **What it does:** Runs on check-suite activity. Only `requested`/`rerequested` types start workflows.
- **Why not self-tested:** Check suites are produced by a checks-creating **GitHub App**; not
  reproducible with a PAT. YAML armed and valid.

### `status` ✅
- **What it does:** Runs when a **commit status** changes (the legacy "statuses" API — distinct from
  Checks).
- **What I did:** `POST /repos/.../statuses/<sha>` with `state=success` (works with a `repo`-scoped PAT).
- **Result:** Ran — `event_name = status`.
  [run 27659586192](https://github.com/donaldraph/actions-1/actions/runs/27659586192)

---

## Repo meta

### `fork` 📄 (doc-only)
- **What it does:** Runs when someone forks the repo (fires on the **upstream** repo).
- **Why not self-tested:** You can't fork a repo you own; needs a second account. YAML armed and valid.

### `watch` ✅
- **What it does:** Runs when someone **stars** the repo. The only event type is `started` (historical
  naming — "watch" actually means "star").
- **What I did:** `DELETE` then `PUT /user/starred/donaldraph/actions-1` (unstar→star for a fresh event).
- **Result:** Ran — `event_name = watch`.
  [run 27659623724](https://github.com/donaldraph/actions-1/actions/runs/27659623724)

### `public` 📄 (doc-only)
- **What it does:** Runs when a repo is changed from **private to public**.
- **Why not self-tested:** The repo is **already public**, and flipping visibility back and forth is a
  disruptive, repo-wide change I avoided. YAML armed and valid.

### `gollum` 🟡 (manual/external)
- **What it does:** Runs when a **wiki** page is created or updated.
- **What I did / why it didn't fire:** Wiki is enabled, but GitHub does **not** provision the
  `*.wiki.git` repo until the **first page is created in the web UI**, and there's no API for wiki
  content. Pushing to `actions-1.wiki.git` returned `Repository not found`.
- **To fire it (human):** Create the first wiki page in the browser. After that, `git push` to the
  wiki repo (or editing pages in the UI) fires `gollum`.

### `page_build` 📄 (doc-only)
- **What it does:** Runs when a **GitHub Pages** build completes.
- **Why not self-tested:** Requires enabling Pages with a source and triggering a build. Left disabled
  to avoid standing up Pages infrastructure. YAML armed and valid. To fire: enable Pages
  (`POST /repos/.../pages` with a source branch) and push to that branch.

### `branch_protection_rule` ✅
- **What it does:** Runs when a branch protection rule is created / edited / deleted.
- **What I did:** GraphQL `createBranchProtectionRule` for pattern `lab-bp-test` (I'm the owner = admin).
- **Result:** Ran — `event_name = branch_protection_rule`.
  [run 27659775784](https://github.com/donaldraph/actions-1/actions/runs/27659775784)
  *(rule deleted at teardown.)*

---

## Chaining & external triggers

### `workflow_run` ✅
- **What it does:** Runs when **another** named workflow starts/finishes. Lets you chain workflows.
- **What I did:** Pointed it at an existing workflow — `workflows: ["Repo Snapshot"]`,
  `types: [completed]`, `branches: [trigger-lab]`. My arming push ran "Repo Snapshot" (`on: push`);
  when it completed, `workflow_run` fired.
- **Result:** Ran — `event_name = workflow_run`.
  [run 27659834160](https://github.com/donaldraph/actions-1/actions/runs/27659834160)

### `workflow_call` ✅
- **What it does:** Makes this workflow **reusable** — another workflow calls it like a function via
  `uses:` and passes `inputs`/`secrets`.
- **What I did:** Made the inputs/secrets optional, added a temporary caller `lab-caller.yml`
  (`uses: ./.github/workflows/learn-trigger.yml`, `with: {environment: lab}`, `secrets: inherit`),
  and dispatched the caller.
- **Result:** A called reusable workflow runs **nested** under the caller and inherits the caller's
  `event_name` (so it shows as `workflow_dispatch`, not `workflow_call`). The nested job
  `call-trigger-lab / show-trigger` succeeded inside the caller run.
  [caller run 27659904124](https://github.com/donaldraph/actions-1/actions/runs/27659904124)
  *(caller workflow removed at teardown.)*

### `repository_dispatch` ✅
- **What it does:** Lets an external system start a workflow via the REST API with a custom event type.
- **What I did:** `POST /repos/donaldraph/actions-1/dispatches` with `event_type=my-custom-event`.
- **Result:** Ran — `event_name = repository_dispatch`.
  [run 27659945079](https://github.com/donaldraph/actions-1/actions/runs/27659945079)

---

## Default-branch swap & teardown

To keep `main` clean while still arming default-branch-only triggers:

1. Created `trigger-lab` from `main`'s last legitimate commit and set it as the repo **default branch**
   (`gh repo edit --default-branch trigger-lab`).
2. Did **all** per-trigger commits on `trigger-lab` (one trigger per commit, pushed separately).
3. Restored the original default branch at the end (`gh repo edit --default-branch main`).
4. `main` was rewound to its pre-test state and force-pushed, so it carries **none** of the trigger
   test commits. `trigger-lab` is left in place for review and was **not** merged into `main`.

**Other changes made (and how to undo):**
- **Discussions enabled** — undo: `gh repo edit donaldraph/actions-1 --enable-discussions=false`.
- **Branch protection rule `lab-bp-test`** — deleted at teardown.
- **Temporary caller workflow `lab-caller.yml`** — deleted at teardown.
- Test artifacts left behind on the repo (harmless; delete anytime): issue #3, discussion #4,
  milestone "Lab milestone", release/tag `lab-rel-v1`, label `lab-label`, two `lab` deployments,
  closed PR #2 (branch `pr-lab-1`), and a repo star.
