# admin

Branch policy for the ReduxISU org, as code. Powered by
[safe-settings](https://github.com/github-community-projects/safe-settings), which re-applies
this config on a schedule — so settings changed by hand in the GitHub UI get reverted on the
next run. Change policy here, not in repo settings.

## The policy today

Every managed repo's **default branch** requires a pull request with **1 approving review**.
Default branch names differ across the org (`CSharpAPI`, `ReduxAPI_GUI`, `main`), so the rule
targets the `~DEFAULT_BRANCH` special ref rather than a literal name.

| Repo | Covered | Admin bypass |
|---|:---:|:---:|
| `Redux` | yes | no |
| `Redux_GUI` | yes | no |
| `quantumsolver` | yes | no |
| `Redux_Build_System` | yes | **yes** — org admins can merge without review |
| `mcpredux` | no | — |
| `admin` (this repo) | no | — |
| *any new repo* | yes, automatically | no |

## Layout

```
.github/settings.yml              org-wide defaults — applies to every managed repo
.github/repos/<Repo>.yml          per-repo overrides, merged into the above by ruleset `name`
deployment-settings.yml           which repos the sync may touch
.github/workflows/                the scheduled sync
```

Precedence is repo > org. A `repos/<Repo>.yml` entry whose ruleset `name` matches one in
`settings.yml` merges into it; a new name is added alongside.

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

The App needs repo **Administration: read & write**, repo **Metadata: read**, repo
**Contents: read**, org **Members: read**, and must be installed on every repo it manages.
It does **not** need org Administration — that is only required for organization-level
rulesets, which this config deliberately avoids so it works on any GitHub plan.
