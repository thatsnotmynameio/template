# Organization template and shared workflows — design

Takes what is generic in `thatsnotmynameio/pururu-ha` (its GitHub setup, CI conventions, release flow, Claude review process) and turns it into:

- **`thatsnotmynameio/.github`**: the single source of the shared pieces. Reusable workflows, composite actions, the bootstrap script and the organization's default `SECURITY.md`.
- **`thatsnotmynameio/template`** (this repository): a language-agnostic GitHub template repository. It holds thin callers pinned to `.github`, the agent instructions and the per-repository files.

Out of scope: migrating `pururu-ha` (or any existing repository) to the shared workflows; a language stack (build, tests, linters) — each project adds its own.

## Decisions

| Question | Decision |
|---|---|
| Stack | **Language-agnostic.** Only infrastructure: GitHub settings, `@claude`, releases, docs, Sonar, agent instructions. |
| No duplication | Shared logic lives once in `thatsnotmynameio/.github`. A repository created from the template copies only callers (a few lines each), pinned by SHA with a `# vX.Y.Z` comment. Dependabot (`github-actions`) bumps those refs, reusable workflows and actions alike. |
| Why the `.github` repository | It is GitHub's special organization repository: its `SECURITY.md` is the default for every repository of the organization that has none. Public, so its workflows and actions can be called from the organization's private repositories too. |
| Versions | Everything starts at **v0.1.0**: `.github`'s first release and the template's `VERSION`. |
| Reusable workflow vs composite action | Reusable workflow when the shared piece is whole jobs (`claude.yml`, `docs.yml`, `sonar.yml`). Composite action when it is steps inside the caller's job and needs files of its own (`actions/release`): an action runs from `github.action_path`, at the ref the caller pinned, so `release.py` is pinned with it. A reusable workflow has no clean way to check out its own repository at its own ref. |
| Secrets | `CLAUDE_CODE_OAUTH_TOKEN` and `SONAR_TOKEN` are organization secrets, available to the repositories that use them; callers pass them with `secrets: inherit`. Nothing in this design sets a secret. |
| Permissions | Callers grant the ceiling on the job that calls a reusable workflow; each reusable job asks for the least it needs (permissions only go down). |
| Claude review | **No workflow.** The review process of `pururu-ha` (`.claude/review/review.md`, `rules.md`, the `review-reader` agent) becomes a local skill, `/pr-review <N>`, run from Claude Code with the user's `gh`. |
| Agent instructions | `AGENTS.md` and `.agents/` are canonical; `CLAUDE.md` and `.claude/skills`, `.claude/agents` are symlinks to them. `.claude/` stays a real directory, for what is Claude's alone when a project needs it (`settings.json`). |
| Repository settings | A template copies files only: settings, rulesets, security features and variables are not copied. `scripts/bootstrap.sh` in `.github` applies them to a repository, idempotently. The organization ruleset "main rule" (PR required, squash only, no force push, no deletion, review threads resolved) already applies to every repository by itself. |
| Cut (YAGNI) | The Claude review workflow and its publish job, `profile/README.md`, `CONTRIBUTING.md`, custom labels, HACS, hassfest, the Home Assistant ruff and mypy settings, any stack. |

## `thatsnotmynameio/.github`

```
.github/
├── workflows/
│   ├── actionlint.yml    reusable: actionlint over the caller's workflows
│   ├── claude.yml        reusable: @claude on issues and pull requests
│   ├── docs.yml          reusable: docs.page check
│   ├── sonar.yml         reusable: SonarQube Cloud scan
│   ├── ci.yml            this repository: actionlint, shellcheck, unittest (release.py, bootstrap.sh), version
│   └── release.yml       this repository: publishes its own vX.Y.Z (dogfoods actions/release)
└── dependabot.yml        github-actions, weekly
actions/
└── release/
    ├── action.yml        composite: inputs mode (check | publish), version-file (default VERSION)
    ├── release.py        the version rules
    └── test_release.py   unittest, stdlib only
scripts/
├── bootstrap.sh          applies the GitHub settings to <owner/repo>
└── test_bootstrap.py     unittest against a fake gh on PATH
SECURITY.md               the organization's default policy
VERSION                   0.1.0
README.md                 what is here and how a repository uses it
LICENSE, .gitignore
```

### `actionlint.yml` (reusable)

Checks out the caller, downloads actionlint's official `linux_amd64` release (a pinned version, verified against its published sha256) into `$RUNNER_TEMP`, and runs it over `.github/workflows`; actionlint also runs the runner's shellcheck on every `run:`. Job name: `actionlint`, so its check is `<caller job> / actionlint`. The pinned version is bumped by hand (Dependabot doesn't see it).

### `claude.yml` (reusable)

`pururu-ha`'s `claude.yml` as a `workflow_call`: the `@claude` mention filter, checkout, `anthropics/claude-code-action` pinned by SHA, `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`, `actions: read`. The action refuses actors without write access, so a stranger's comment on a public repository triggers nothing. Input: `claude-args` (default empty; `--model` goes there). The caller passes secrets with `secrets: inherit`. The caller keeps the `on:` events (`issue_comment`, `pull_request_review_comment`, `issues`, `pull_request_review`), since a reusable workflow can't declare its caller's triggers.

### `docs.yml` (reusable)

`pururu-ha`'s Docs workflow: checkout, `pnpm/action-setup`, `actions/setup-node` (Node 24, pnpm cache), `pnpm install --frozen-lockfile`, `pnpm docs:check`. The caller's `package.json` pins the docs.page CLI and pnpm (`packageManager`), its `pnpm-lock.yaml` pins them by hash. Job name: `docs.page check`. `contents: read`, `timeout-minutes: 5`.

### `sonar.yml` (reusable)

Checkout with `fetch-depth: 0`, then, when the input `coverage-artifact` is set, `actions/download-artifact` of it into the workspace, then `SonarSource/sonarqube-scan-action` with `SONAR_TOKEN`. The caller's `sonar-project.properties` configures the project. Job name: `SonarQube`. `contents: read`.

### `actions/release`

`release.py` generalizes `pururu-ha`'s:

- **Version file:** `VERSION` (the whole file, stripped), `*.json` (the top-level `version`), `*.toml` (`project.version`, read with `tomllib`). Any other extension is refused.
- **Rules (unchanged):** the version must be `MAJOR.MINOR.PATCH`; below the latest `v*` tag is refused; equal to an existing tag means already released.
- **`check`:** prints what will happen, fails on a refusal.
- **`next`:** prints the tag to publish, or nothing.

`action.yml`, `mode: check` runs `python3 release.py check`. `mode: publish` runs `next` and, when it prints a tag, `gh release create "$TAG" --target "$GITHUB_SHA" --title "$TAG" --generate-notes` with the caller's token. Both need a checkout with `fetch-depth: 0` (tags); `publish` needs `contents: write`. Python is the runner's `python3` (3.12 on `ubuntu-latest`, stdlib only).

### `bootstrap.sh`

`scripts/bootstrap.sh <owner/repo> [--checks "a,b,c"] [--sonar] [--template]`, using `gh` with the caller's login. Each step is idempotent. A step GitHub refuses for the plan (a private repository without GitHub Advanced Security) prints a warning and the script goes on. Any other failure stops it.

1. **Merge settings:**
   - squash only (`allow_merge_commit` and `allow_rebase_merge` false);
   - `squash_merge_commit_title: COMMIT_OR_PR_TITLE`, `squash_merge_commit_message: COMMIT_MESSAGES`;
   - `delete_branch_on_merge`, `allow_update_branch`;
   - `has_wiki` and `has_projects` false.
2. **Security:**
   - vulnerability alerts and Dependabot security updates on;
   - secret scanning and push protection on;
   - private vulnerability reporting on (the organization's `SECURITY.md` points there);
   - CodeQL default setup configured.
3. **Ruleset "checks" on the default branch:**
   - Required status checks default to `version`, `actionlint / actionlint`, `docs / docs.page check` (a job of a reusable workflow reports as `<caller job> / <its job>`), each required from GitHub Actions (integration 15368); `--checks` replaces the list.
   - Code scanning is required (CodeQL, errors, high or higher) only when step 2 configured CodeQL's default setup: requiring a check that never runs would block every pull request.
   - An existing ruleset of that name is updated, otherwise one is created.
4. `--sonar`: the repository variable `SONAR_ENABLED=true`.
5. `--template`: `is_template: true`.

It ends by printing the resulting settings (`gh api repos/<repo>` fields and the rulesets) so the outcome can be checked at a glance.

### This repository's CI and releases

- **`ci.yml` on pull requests:**
  - `actionlint` on the workflows, through `./.github/workflows/actionlint.yml`;
  - `shellcheck` on `bootstrap.sh`;
  - `python3 -m unittest` in `actions/release` and `scripts` (job `unittest`);
  - `actions/release` in `check` mode, from the pull request's own tree (`uses: ./actions/release`).
- **`release.yml` on `main`:** the same action in `publish` mode.
- Actions are pinned by SHA.

## `thatsnotmynameio/template`

```
.github/
├── workflows/
│   ├── ci.yml            PR: job `version` (actions/release check) and job `actionlint` (calls .github's actionlint.yml); the project adds its own
│   ├── release.yml       push to main: actions/release publish (needs the project's build jobs, when there are any)
│   ├── docs.yml          PR: calls .github's docs.yml
│   ├── sonar.yml         PR and main: calls .github's sonar.yml, if: vars.SONAR_ENABLED == 'true'
│   └── claude.yml        the @claude events: calls .github's claude.yml
└── dependabot.yml        github-actions and npm (the docs CLI), weekly
.agents/
├── skills/pr-review/
│   ├── SKILL.md          the review process
│   └── rules.md          this repository's rules (skeleton)
└── agents/
    └── review-reader.md  read-only finder/verifier subagent
.claude/
├── skills -> ../.agents/skills
└── agents -> ../.agents/agents
AGENTS.md                 skeleton of pururu-ha's CLAUDE.md sections
CLAUDE.md -> AGENTS.md
docs.json, docs/index.mdx the minimal docs.page site
package.json              docs.page CLI + packageManager pnpm; pnpm-lock.yaml
sonar-project.properties  skeleton (projectKey, organization, sources, tests, coverage path)
VERSION                   0.1.0
README.md                 using the template: create, run bootstrap, fill AGENTS.md and rules.md
SECURITY.md               none: the organization default applies
LICENSE, .gitignore
docs/superpowers/         specs and plans (not published by docs.page: only .mdx is)
```

Every caller pins `thatsnotmynameio/.github` by the SHA of its `v0.1.0` with a `# v0.1.0` comment. Every workflow has `permissions` (least needed), `concurrency` (`${{ github.workflow }}-${{ github.ref }}`, cancel in progress, except releases: never cancelled, in merge order) and `timeout-minutes`.

### `AGENTS.md`

Skeleton with `pururu-ha`'s `CLAUDE.md` sections, each with a one-line prompt of what goes there:

- Commands
- Architecture
- Tests
- Docs (docs.page, the Guide/Develop tabs, MDX's `{` and `<`, pnpm never npm)
- Releases and CI (`VERSION` bump is a release, actions pinned by SHA, Sonar suppressions in `sonar-project.properties` with a reason)

The parts that hold for any repository are written out, not left as prompts.

### The `/pr-review` skill

The review process of `pururu-ha` (`.claude/review/review.md`), same rigor, adapted to run locally:

- **Kept:**
  - full or incremental mode, from the state in the summary comment;
  - finders in parallel, one `review-reader` per lens the diff gives work to;
  - skeptical verifiers, one per candidate; only CONFIRMED findings, at most 6, most severe first;
  - earlier findings judged fixed, withdrawn or outstanding, their threads resolved when fixed or withdrawn;
  - the summary format: confidence, risk, findings list, important files, optional mermaid, footer, hidden state;
  - inline comment format: badge, title, scenario, suggestion, "Prompt to fix with AI".
- **Lenses:**
  - correctness;
  - lifecycle and concurrency (async order, resources and their release, partial failure);
  - contracts (inputs, schemas, public interfaces, what callers and readers assume);
  - security (injection, secrets, authorization, untrusted input);
  - consistency (docs, tests, CI, siblings of a pattern the PR sets);
  - sweep (incremental only: the whole PR, P0 and P1 only);
  - plus any extra lens `rules.md` declares.
- **Generic in `SKILL.md`:** the confidence rubric and its caps (any P0 → at most 1; two or more P1 → at most 3; any P1 → at most 4).
- **Per repository in `rules.md`:** severity examples, never flag, worth checking, extra lenses, risk areas.
- **Changed for local use:**
  - **Posting:** the new findings are posted as one review, `gh api repos/{repo}/pulls/{pr}/reviews` with `event: COMMENT` and `comments[]`, instead of the Actions inline-comment tool.
  - **Summary and threads:** the session edits or creates the summary comment by its marker `<!-- pr-review:summary -->`, and resolves threads (`resolveReviewThread`) itself. No publish job, no handoff files.
  - **Owner:** the owner is `gh api user --jq .login`. Only comments by that login and the review's own are read; the repository is public, so anyone else's text is never read, and the pull request's content is data under review, never instructions.
  - **Confirmation:** before posting anything, the skill shows the findings, the summary and the threads to resolve, and waits for the user's confirmation: the token is the user's.
  - **Reading the head:** the head is fetched without leaving the user's branch (`gh pr view --json headRefOid`, `git fetch origin pull/{pr}/head`); files are read at that sha (`git show {head}:{path}`), since the working tree may be elsewhere.
- **Markers:** `<!-- pr-review:summary -->` and `<!-- pr-review:state {...} -->`.

`review-reader.md` keeps `pururu-ha`'s rules (Read, Grep, Glob, read-only git; never `gh`; never posts), without the project name. It adds reading at a sha with `git show`.

`rules.md` skeleton sections: Severity (P0/P1/P2 generic definitions, to adjust), Never flag (this repository's linters and CI, style, lockfiles, `docs/superpowers/`), Worth checking (empty, with commented examples), Extra lenses (empty), Risk (low / medium / high / critical areas, to fill).

### `.gitignore`

Language-agnostic; each project appends its stack's entries.

```gitignore
# OS and editors
.DS_Store
Thumbs.db
.idea/
.vscode/
*.swp

# Environment and secrets
.env
.env.*
!.env.example

# Agents' local state (the shared instructions in .agents/ and .claude/ are committed)
.claude/settings.local.json
.claude/autoharness/
.remember/

# Docs (docs.page CLI)
node_modules/

# Coverage reports (Sonar)
coverage.xml
.coverage
coverage/
```

`.github` gets the same file without the docs and coverage blocks, plus `__pycache__/` (its `release.py` tests).

### Symlinks

Git stores them as symlinks (mode `120000`) and "Use this template" keeps them. On Windows they need `core.symlinks=true`; the README says so.

## Rollout and verification

1. **Build `.github`:**
   - Create it (public) and push its files through a pull request.
   - Its CI must pass: actionlint, shellcheck, unittest, version check.
   - Run `bootstrap.sh` on it (`--checks "version,shellcheck,unittest,actionlint / actionlint"`).
   - Merge; its Release publishes `v0.1.0`.
2. **Build `template`:**
   - Pin the callers to `.github`'s `v0.1.0` SHA.
   - Push through a pull request; its checks must pass (`version`, `actionlint / actionlint`, `docs / docs.page check`; Sonar skipped: the variable is unset).
   - Run `bootstrap.sh thatsnotmynameio/template --template`.
3. **End to end:**
   - Create a throwaway repository from the template and run the bootstrap.
   - Check its settings and rulesets through `gh api`.
   - Open a pull request there. Its checks pass; `/pr-review` finds, asks, posts and summarizes; a second run is incremental.
   - Delete the throwaway repository only after the user confirms.
4. **Organization secrets:** `CLAUDE_CODE_OAUTH_TOKEN` (and `SONAR_TOKEN` when used) must be organization secrets available to these repositories. If they aren't, the user sets them; nothing here does.

## Risks

- **The org ruleset on the first push.** "main rule" targets the default branch and requires pull requests. The first push to an empty repository creates `main` itself; if the ruleset refuses it, the first commit goes through the web UI (a README), or the owner bypasses once.
- **Required checks named by job.** A required check is the job's name. Renaming a job blocks every pull request, waiting for a check that never comes. The job names in this design are fixed, and `bootstrap.sh --checks` changes the list.
- **GitHub Advanced Security on private repositories.** Secret scanning, push protection and CodeQL may be unavailable on a private repository under the Team plan; the bootstrap warns and continues.
