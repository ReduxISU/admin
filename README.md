# admin

Branch policy for the ReduxISU org, as code. Powered by
[safe-settings](https://github.com/github-community-projects/safe-settings), which re-applies
this config on a schedule — so settings changed by hand in the GitHub UI get reverted on the
next run. Change policy here, not in repo settings.

## The policy today

Every managed repo's **default branch** requires a pull request with **1 approving review**.
Default branch names differ across the org (`CSharpAPI`, `ReduxAPI_GUI`, `main`), so the rule
targets the `~DEFAULT_BRANCH` special ref rather than a literal name.

| Repo | Covered | Required check | Admin bypass |
|---|:---:|:---:|:---:|
| `Redux` | yes | no — `rbs.yml` is still `soft: true` | no |
| `Redux_GUI` | yes | no — same | no |
| `quantumsolver` | yes | **`build-test / ci`** (the rbs pipeline) | no |
| `Redux_Build_System` | yes | no | **yes** — org admins can merge without review |
| `Redux_VR` | yes | no | **yes** — same, internal single-maintainer repo |
| `mcpredux` | no | — | — |
| `admin` (this repo) | no | — | — |
| *any new repo* | yes, automatically | no | no |

A required check is only meaningful over a workflow that can fail. Redux and Redux_GUI call the
rbs workflow with `soft: true`, which reports without failing, so requiring their check would gate
nothing; each gets the rule in `repos/<Repo>.yml` when it drops `soft`, and once all three have it
the rule moves into `suborgs/all-repos.yml`.

## Layout

```
.github/suborgs/all-repos.yml     the baseline ruleset; `*` matches every repo
.github/repos/<Repo>.yml          per-repo overrides, merged in by ruleset `name`
.github/settings.yml              org-level config — deliberately empty, see below
deployment-settings.yml           which repos the sync may touch
.github/workflows/                the scheduled sync
```

Precedence is repo > suborg > org. A `repos/<Repo>.yml` entry whose ruleset `name` matches the
suborg baseline merges into it; a new name is added alongside.

**The ruleset lives in `suborgs/`, not `settings.yml`, and that placement is load-bearing.**
safe-settings hardcodes `SCOPE.ORG` for rulesets in the org `settings.yml` and strips them before
cascading to repos — so a ruleset there is POSTed to `/orgs/{org}/rulesets` as an *organization*
ruleset, which requires the App to hold org Administration: write **and** a Team/Enterprise plan.
Rulesets in `suborgs/*.yml` and `repos/*.yml` are repo-scoped (`/repos/{owner}/{repo}/rulesets`),
which works on any plan and allows the per-repo bypass above.

## Making a change

1. Edit the YAML, open a PR, merge it.
2. Actions → **safe-settings sync** → **Run workflow**. The cron is every 4 hours; dispatch
   manually rather than waiting.
3. Read the job log — it names each repo it touched and what changed.

## Adding a repo to the org

Nothing to do. `deployment-settings.yml` uses `restrictedRepos.exclude`, a denylist, so new
repos are covered on the next sync. To opt one out, add it to that list.

## Credentials

The sync authenticates as a GitHub App owned by the org. It needs, on this repo:

- variable `SAFE_SETTINGS_GH_ORG` — `ReduxISU`
- variable `SAFE_SETTINGS_APP_ID`
- secret `SAFE_SETTINGS_PRIVATE_KEY` — the App's `.pem`, in full

The App needs repo **Administration: read & write**, repo **Checks: read & write**, repo
**Metadata: read**, repo **Contents: read**, org **Members: read**, and must be installed on
every repo it manages —
**plus `admin` itself**, which it reads this config from. `admin` appears in
`deployment-settings.yml` so it is never *managed*; that is separate from needing to *read* it.

Install the App on the **ReduxISU org only**. `syncInstallation` in safe-settings takes
`installations[0]` — the first installation, unfiltered — so an extra installation on a personal
account can silently make the sync target the wrong place. The `GH_ORG` env var does not prevent
this: it appears in upstream's docs but is referenced nowhere in the code, so it selects nothing.
It does **not** need org Administration — that is only required for organization-level
rulesets, which this config deliberately avoids so it works on any GitHub plan.

## Known upstream issue

The sync workflow downgrades probot to 13.4.7 after installing safe-settings. This is load-bearing:
safe-settings 2.1.19 bumped its probot dependency to ^14, which builds its logger in a lazy async
init, but never updated the code that calls `probot.log` synchronously right after `createProbot()`.
Without the pin every run fails immediately with `Cannot read properties of null (reading 'info')`.
Verified against 2.1.21 and unchanged upstream as of the last check. Remove the pin only after
confirming `full-sync.js` awaits initialization.
