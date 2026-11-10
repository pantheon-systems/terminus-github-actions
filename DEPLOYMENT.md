---
contract_version: 2
kind: tool
service: terminus-github-actions   # a composite GitHub Action (action.yml) that installs Terminus; nothing runs on Pantheon infrastructure

trigger: merge-to-main              # every merge to main is auto-tagged and released; consumers pick it up only through their own pin
merged_implies_deployed: true       # a new tag + GitHub release exists within ~15 s of the merge. Whether any consumer RUNS it depends on how it pins (see consumers)
other_paths_to_prod:
  - "the `v1` BRANCH (not a tag) is what `uses: pantheon-systems/terminus-github-actions@v1` resolves to. Nothing in this repo moves it: it still points at 409451e (= v1.2.8, 2026-04-21), nine commits behind main. A push to `v1` changes what every @v1 consumer runs, with no workflow run here"
  - "a consumer using `@main` runs whatever main is on its next workflow run, release or no release"
  - "pushing or editing a tag/release by hand (the autotag step is the only thing that normally creates them)"
  # trap: a failed autotag run can leave a tag with no GitHub release. v1.2.11 and v1.2.12 are tags without releases (the runs on 34305c6 and 5a3632d failed in a stray `action-autotag@v0` step, removed in #55). Releases come from the same step that pushes the tag, so check both

pipeline:
  system: github-actions
  workflow: ".github/workflows/tag-release.yml"   # workflow name "Autotag and Release"
  jobs_in_order: [tag-release]
  typical_minutes: 0.25             # observed: run 42634330558 (v1.2.13) 14:24:16Z -> 14:24:29Z
  minutes_by_change: {}
  watch: "`gh run list -R pantheon-systems/terminus-github-actions --workflow tag-release.yml -L 3`; the run for the merge commit on main"
  concurrency_group: null           # none declared in the workflow
  cancel_in_progress: null          # n/a: no concurrency group
  blocked_by_waiting_run: null      # no gate, no queue
  approval_gate: null               # no GitHub Environment; the human gate is the PR review (and the merge queue)

release:
  mechanism: "merge to main (merge queue, squash) -> 'Autotag and Release' runs pantheon-systems/action-autotag (pinned by SHA, v1.3.0), which bumps the semver tag from the commit messages, pushes it and creates a GitHub release with generated notes (create-release defaults to true). The workflow does not set `push-major-version-branch`, so no `v1` branch or tag is touched"
  tag_scheme: "v-prefixed semver (v1.2.13). No floating `v1`/`v1.2` tag exists; `v1` is a branch (see other_paths_to_prod)"
  never: "move or delete a published tag or release: consumers pin tag SHAs and dependabot bumps them. Fix forward with the next release"
  procedure: "merge the PR. There is no separate release step"

consumers:                          # from `gh search code` over pantheon-systems (search capped at 30 hits per query, so not exhaustive) and a read of each hit's workflow; observed 2026-11-10
  - {repo: push-to-pantheon, pin: "@v1 (branch)", picked_up_by: "frozen at v1.2.8 until someone moves the v1 branch. action.yml and .github/workflows/deploy-pr.yml. push-to-pantheon is itself a public action, so its users run this transitively"}
  - {repo: documentation, pin: "@v1 (branch) in .github/actions/wait-for-deployment; SHA 409451e (v1.2.8) in .github/workflows/terminus-docs-update.yml", picked_up_by: "branch: frozen as above. SHA: dependabot"}
  - {repo: cu-push-to-pantheon, pin: "@main", picked_up_by: "immediately: every merge here changes what its next run executes. All 8 workflows that use the action pin @main (migration-live, upstream-release-live, deploy-to-live-on-{tag,one,all}, merge-upstream-on-{one,all,canary}-dev), including the deploy-to-live ones"}
  - {repo: "terminus-rsync-plugin, terminus-scheduled-jobs-plugin, nextjs-automation", pin: "SHA 409451e (v1.2.8)", picked_up_by: "dependabot or a hand edit"}
  - {repo: "pantheon-advanced-page-cache, search_api_pantheon, pantheon_secrets", pin: "SHA b0a005d (v1.2.9)", picked_up_by: "dependabot or a hand edit"}
  - {repo: go-ellis, pin: "SHA 34305c6 (v1.2.11)", picked_up_by: "dependabot or a hand edit"}
  - {repo: autopilot-ui-automation, pin: "SHA 8e024bd (v1.2.7)", picked_up_by: "dependabot or a hand edit"}
  - {repo: "terminus-autopilot-plugin, terminus-composer-logs-plugin", pin: "SHA 19469b3 (a main commit between v1.2.11 and v1.2.12, never tagged)", picked_up_by: "dependabot or a hand edit"}
  - {repo: "external users", pin: "@v1 per the README", picked_up_by: "same as the @v1 rows"}

repo_properties:
  observed_at: 2026-11-10
  trust_level: TL4
  ccb_status: in-ccb
  owner: developer-experience
  repo_type: tool

gating_checks:                      # `gh api repos/pantheon-systems/terminus-github-actions/rules/branches/main`
  required: ["Wiz IaC Scanner", "Wiz Vulnerability Scanner", "Wiz Data Scanner"]   # org ruleset 'TL4 - Low'
  approvals: {required: 1, code_owner: true, last_push_approval: false, strict_up_to_date: false, merge_queue: true}   # org + repo ruleset 'Protect main' (merge queue: SQUASH, ALLGREEN, 60 min timeout)
  run_but_not_required: ["Terminus Github Action Tests (Terminus Setup, Terminus Login, Terminus Version)", "CodeQL"]   # the tests run on pull_request only, not in the merge queue
  conditional: []
  release_gate: "none. Whatever passes review and the queue is released on the spot"

environments:
  - {name: github-release, target: "github.com/pantheon-systems/terminus-github-actions/releases/<tag>", gates_prod: true}

artifact:
  image: null
  file: "action.yml at the tagged commit (a composite action; no build step)"
  tag_scheme: semver
  tag_mutable: false                # by convention; nothing enforces it
  retention: "tags and releases kept indefinitely"
  version_signal: "the tag a consumer's `uses:` line resolves to; `git ls-remote origin refs/tags/<tag> refs/heads/v1`"
  tag_to_sha: "git rev-list -n1 <tag>"

runtime: []                         # runs inside consumers' workflow runs on GitHub-hosted runners

deployed_when:                      # ALL must hold
  - "the 'Autotag and Release' run for the merge commit succeeded"
  - "`gh release view <tag> -R pantheon-systems/terminus-github-actions` exists, is not a draft, and `git rev-list -n1 <tag>` is the merge commit on main"
  - "consumers pinned to a branch: the `v1` branch contains the commit (`git merge-base --is-ancestor <merge-sha> origin/v1`). Today it does not, so @v1 consumers are NOT running main"

health:
  soak_minutes: 0                   # no fleet to watch; the signal is the next runs of consumers
  baseline: "the previous release's `Terminus Github Action Tests` run on its PR; consumers' workflow history"
  checks:
    - name: release exists and points at the merge commit
      kind: command
      run: "gh release view <tag> -R pantheon-systems/terminus-github-actions --json tagName,isDraft,isPrerelease,targetCommitish && git ls-remote https://github.com/pantheon-systems/terminus-github-actions refs/tags/<tag>"
      healthy: "tagName == <tag>, isDraft false; the ls-remote SHA is the merge commit"   # run for v1.2.13 on 2026-11-10: release present, Latest, published 2026-11-09T14:24:25Z
    - name: the action installs Terminus
      kind: delegate
      run: "the PR-time workflow 'Terminus Github Action Tests' (installs Terminus with `uses: ./`, runs `terminus -V` and `terminus art`), and the next run of a consumer on @main (cu-push-to-pantheon)"
      healthy: "green. Recent PR runs (2026-11-09) were green"
  known_baseline_errors:
    - "'Autotag and Release' failed on 34305c6 (2026-09-28) and 5a3632d (2026-11-09) because of a stray `action-autotag@v0` step. Both runs still pushed tags (v1.2.11, v1.2.12) but no release. Fixed in #55; v1.2.13 was clean"
  dashboards: []
  alerts: []

rollback:
  options:
    - when: "a release breaks consumers that run @main or have just bumped their SHA pin"
      kind: roll-forward
      steps: |
        1. Open a revert PR on main (revert the offending commit), get the usual approval, merge through the queue.
        2. 'Autotag and Release' cuts the next patch release (~15 s). @main consumers get it on their next run.
        3. Consumers that already pinned the bad SHA: re-pin to the previous release SHA (`git rev-list -n1 <previous tag>`) or to the new fix.
      time_to_rollback_minutes: 30  # inferred: PR review plus the queue
    - when: "the v1 branch was advanced to a bad commit"
      kind: retract-and-repin
      steps: |
        1. Record the target first (see capture_before_merge).
        2. Reset the branch to it: `git push --force-with-lease origin <previous-sha>:v1`. This needs a force push to a branch that consumers run directly; check what the rules allow.   # inferred: never exercised
      time_to_rollback_minutes: 5   # inferred
  not_restored:
    - "consumers that already ran the bad version: it only affects runs after the change"
    - "consumers pinned to the bad tag's SHA stay on it until re-pinned"
    - "the tag and release for the bad version remain (never delete)"
  one_way_changes: []               # checked: no server-side state
  currently_blocked: null
  last_exercised: null
  capture_before_merge: "`git ls-remote https://github.com/pantheon-systems/terminus-github-actions refs/heads/v1 refs/heads/main` and `gh release view --json tagName -R pantheon-systems/terminus-github-actions` (current latest)"
  who_can: ["@pantheon-systems/developer-experience: merge (with a code-owner approval), and admin on the repo for any branch reset. TODO(owner): who has force-push rights on the v1 branch"]
  verify: "`git ls-remote ... refs/heads/v1` returns the recorded SHA; a consumer run on @v1/@main goes green"

escalation:
  slack: "#ask-hosting"
  owner_team: developer-experience
  paged: false                      # inferred: no alerts exist
  issues: "https://github.com/pantheon-systems/terminus-github-actions/issues"

preflight:
  - "gh auth status (read on the repo; write to merge or reset v1)"

automation_blockers:
  - blocker: "`v1` is a branch with no documented or automated way to move it. Merging and releasing here leaves @v1 consumers (push-to-pantheon, documentation, and their users) on v1.2.8, so 'merged' does not mean 'delivered' for the main consumption path, and fixes such as the Node 24 pins never reach them"
    clears_when: "the owners decide how @v1 is maintained: a floating v1 tag moved by the release workflow (action-autotag `push-major-version-branch` or a tag-moving step), or a documented manual procedure, and the README says which"
  - blocker: "cu-push-to-pantheon pins `@main`, so every merge here changes its next run with no review on their side"
    clears_when: "it pins a release tag or SHA"
  - blocker: "a failed autotag run can leave a tag without a release (v1.2.11, v1.2.12), and there is no check that tag and release both exist"
    clears_when: "the workflow fails loudly or retries the release step when the tag exists without one"

verified: null                      # TODO(verify-on-deploy): pass 2 fills it from a real merge
---

# Deployment

This repo is a GitHub Action that installs and authenticates Terminus in a workflow. "Deploying" it means publishing a git tag and GitHub release. Nothing runs on Pantheon infrastructure. Whether a consumer runs the new release depends on how it pins the action.

## How a change reaches production

1. **PR to `main`.** One approval from a code owner (`@pantheon-systems/developer-experience`). Only the three Wiz scanners are required checks. The tests ("Terminus Github Action Tests") run on the PR but don't block merging, so check they're green. They install Terminus with `uses: ./`, run `terminus -V` and `terminus art`, and run the reusable `test-terminus-version.yml`.
2. **Merge.** The repo only allows squash merges, through a merge queue.
3. **"Autotag and Release"** (`.github/workflows/tag-release.yml`) runs on the push to `main`. It uses `pantheon-systems/action-autotag` to pick the next semver tag from the commit messages (`v1.2.13` is a patch bump), push it, and create the GitHub release. About 15 seconds. Nothing gates it.
4. **Consumers pick it up by pin:**
   - `@main`: on their next run (cu-push-to-pantheon, in all 8 of its workflows, including its deploy-to-live ones).
   - a SHA or tag: when dependabot or a person re-pins.
   - `@v1`: never by this pipeline. `v1` is a **branch** that still points at v1.2.8 (409451e, 2026-04-21), and no workflow moves it. The README tells everyone to use `@v1`, and push-to-pantheon and the docs repo do.

Other paths to production: pushing to the `v1` branch, and anything that creates or edits tags and releases by hand.

## How to confirm it deployed

```bash
# release exists, is not a draft, and the tag is the merge commit
gh release view <tag> -R pantheon-systems/terminus-github-actions --json tagName,isDraft,isPrerelease
git ls-remote https://github.com/pantheon-systems/terminus-github-actions refs/tags/<tag>

# where the moving refs are
git ls-remote https://github.com/pantheon-systems/terminus-github-actions refs/heads/main refs/heads/v1
```

For `@v1` consumers, the change is only live if `v1` contains the merge commit. Today it doesn't.

## How to confirm it's healthy

Health belongs to the consumers. The signals are the repo's PR-time tests and the next runs of workflows that use the action (cu-push-to-pantheon on `@main` is the quickest). A broken action shows up as a failed "Install Terminus" or "Authenticate Terminus" step in those runs. There are no dashboards or alerts.

`'Autotag and Release'` was red on 34305c6 (2026-09-28) and 5a3632d (2026-11-09) and still pushed tags. v1.2.11 and v1.2.12 have no GitHub release. #55 removed the stray step; v1.2.13 ran green.

## Rollback

Releases are normally fixed forward. Revert the change on `main` through a PR; the merge cuts the next patch release in seconds. Never delete or move a published tag.

1. **Before merging**, record the targets: `git ls-remote https://github.com/pantheon-systems/terminus-github-actions refs/heads/v1 refs/heads/main` and the current latest release (`gh release view --json tagName -R pantheon-systems/terminus-github-actions`).
2. **`@main` consumers or a freshly bumped pin:** open a revert PR, merge it, and let autotag release it. Consumers pinned to the bad SHA re-pin to the previous release SHA (`git rev-list -n1 <previous tag>`) or the fix.
3. **`@v1` consumers:** a release doesn't reach them. If someone advanced the `v1` branch to a bad commit, reset it to the recorded SHA (`git push --force-with-lease origin <previous-sha>:v1`; never exercised, and check what the branch rules allow).

**Verify:** `refs/heads/v1` and the latest release match the recorded values, and a consumer run is green.

**What this doesn't restore:** runs that already executed the bad version, and consumers pinned to its SHA.

Rollback has never been exercised.

## Escalation

- Owner: `developer-experience` (code owners `@pantheon-systems/developer-experience`).
- Nobody is paged and there are no alerts.
- Issues: https://github.com/pantheon-systems/terminus-github-actions/issues
- Slack: `#ask-hosting`.

## Open owner questions

- TODO(owner): what is the intended way to move `v1`? It last moved on 2026-04-21, and v1.2.9 to v1.2.13 never reached it.
- TODO(owner): is it intended that cu-push-to-pantheon runs `@main`?
- TODO(owner): who can force-push to the `v1` branch (for rollback)?

## Suggested pipeline changes

- Make `v1` a floating tag moved by the release workflow (or document the manual step), and correct the README to match.
- Have the workflow fail if a tag exists without a release.
- Ask cu-push-to-pantheon to pin a tag or SHA.
