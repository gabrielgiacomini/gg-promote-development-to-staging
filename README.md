# promote-development-to-staging
A release handoff skill for opening, reusing, and inspecting
`development`-to-`staging` promotion pull requests across a project-defined
repository set. The workflow validates branch state, infers promotion PR
titles and descriptions, checks PR readiness, and reports blockers without merging or
verifying deployments.

The skill is project-agnostic. Target projects supply the repository set,
release surfaces, branch-protection rules, review policy, and ownership model
for post-merge verification.

## Install

The fastest cross-agent install path is the `skills` CLI:

```bash
npx skills add gg-skills/promote-development-to-staging
```

Drop this skill into a workspace as a Git submodule for pinned versions, or as a plain clone for latest `main`:

```bash
# Project-local, version-pinned:
git submodule add git@github.com:gg-skills/promote-development-to-staging.git .claude/skills/promote-development-to-staging

# OR project-local, latest main:
mkdir -p .claude/skills
git -C .claude/skills clone git@github.com:gg-skills/promote-development-to-staging.git

# OR user-level, available in every project on this machine:
mkdir -p ~/.claude/skills
git -C ~/.claude/skills clone git@github.com:gg-skills/promote-development-to-staging.git
```

Restart your agent or reload skills after installation. See the parent [`skills` catalog repo](https://github.com/gg-skills/skills) for the full catalog.

## When to use

- The user asks to promote `development` into `staging`.
- The operator needs staging promotion PRs opened, reused, inspected, or metadata-validated.
- The task is a release-candidate handoff before merge and deployment verification.
- A monorepo, multi-repo set, or submodule-aware release surface needs a readiness report.

Skip when the user wants the PR merged, wants direct pushes to `staging`, asks
for post-merge smoke tests, or is promoting from a source branch other than
`development`.

## How it operates

### Inputs

| Input | Purpose |
|-------|---------|
| Repository topology | Monorepo by default, multi-repo, or submodule-aware |
| Repository set | Repositories that release together |
| Release surfaces | Apps, packages, workspaces, or deployments affected |
| Source branch | Defaults to `development`; verified remotely before mutation |
| Target branch | Defaults to `staging`; verified remotely before mutation |
| Branch preservation | Source and target release branches are permanent and should not be deleted unless explicitly required and authorized |
| PR metadata policy | Optional title/body convention; otherwise inferred from lane, diff, scope, and handoff evidence |
| Review/check policy | Branch-protection and handoff-readiness criteria |
| Root/meta policy | Whether a root or meta repository also needs a tracking PR |
| Verification owner | Person, CI workflow, or follow-up skill responsible after merge |

### Outputs

| Output | Description |
|--------|-------------|
| Plain-language progress updates | Explains each step before and after it runs, with what changed and what the user should do next |
| Branch verification report | Confirms source and target branch existence for every repository |
| Promotion PR inventory | Existing or newly created PR numbers, URLs, titles, and head SHAs |
| Metadata report | Notes whether each PR title/body matches the inferred or documented metadata policy |
| Readiness report | Mergeability, review decision, status checks, comments, and blockers |
| Link handoff packet | PR, compare, checks, deployment/dashboard, logs, and runtime links when applicable |
| Handoff summary | What is ready, what is blocked, and who owns post-merge verification |

### Metadata and handoff behavior

The skill now infers PR metadata by default. A new PR gets a lane-derived title
and a body built from verified evidence: source and target SHAs, changed-file
risk, repository topology, release surfaces, review/check expectations,
downstream verification owner, and known unknowns. Use a project-provided title
or body pattern only when project docs or automation require it.

Progress reporting is part of the workflow. Before and after each meaningful
step, the agent explains what it is doing, why it matters, whether it changes
anything, what evidence was found, and what the user should do next. When a PR,
check, deployment, dashboard, log, compare, or runtime URL is discoverable, it
belongs in the handoff packet.

For `development` to `staging`, the skill stops at PR handoff. It validates PR
metadata, branch direction, head SHA, mergeability, reviews, checks, comments,
and risky changed files, then reports readiness or blockers. Merging, staging
deployment, and smoke verification are downstream responsibilities.

### External commands

The skill uses read-mostly Git and GitHub CLI commands, creating a PR only when
no matching open promotion PR exists:

```bash
git ls-remote --heads <repo-url> development staging
gh pr list -B staging -H development -R <owner>/<repo> --state open --json number,url,title,body,headRefOid
gh pr create -B staging -H development -R <owner>/<repo> --title "<inferred-title>" --body-file <inferred-body-file>
gh pr view <number> -R <owner>/<repo> --json state,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,comments,reviews,url,headRefOid
gh pr checks <number> -R <owner>/<repo> --watch=false
```

### Side effects

- May create GitHub pull requests from `development` to `staging`.
- May leave comments only if the project workflow or operator requests them.
- Does not merge PRs.
- Does not direct-push `staging`.
- Does not delete `development`, `staging`, or other source/target release branches unless project policy explicitly requires it and the user authorizes the exact repository and branch.
- Does not run or claim post-merge deployment verification.

### Mode toggles

| Mode | Behavior |
|------|----------|
| Discovery-only | Verify repositories, branches, and existing PRs without creating anything |
| PR handoff | Create or reuse PRs, validate inferred metadata, and report readiness |
| Blocker audit | Focus on mergeability, checks, reviews, and comments for already-open PRs |

## Operational flow

```mermaid
flowchart TD
    A([User asks to promote development to staging]) --> B[Resolve project scope]
    B --> C[Identify repos, release surfaces, metadata policy, review policy]
    C --> D[Verify development and staging branches in every repo]
    D --> E{All branches exist?}
    E -- No --> F[Stop with branch mismatch report]
    E -- Yes --> G[Compare source and target heads]
    G --> H{Source has changes?}
    H -- No --> I[Report no-op handoff and stop]
    H -- Yes --> J[Inspect open development to staging PRs]
    J --> K{Matching PR exists?}
    K -- Yes --> L[Reuse PR and record URL]
    K -- No --> M[Infer PR title and body from lane plus diff evidence]
    L --> N[Validate metadata and branch pair]
    M --> M2[Create PR with inferred metadata]
    M2 --> N
    N --> O[Inspect mergeability, reviews, checks, comments]
    O --> P{Ready for release owner?}
    P -- No --> Q[Report blockers and follow-up owners]
    P -- Yes --> R[Report ready staging handoff]
    Q --> S([Stop before merge])
    R --> S
```

### Phase map

| Phase | What the agent does | Stop condition |
|-------|---------------------|----------------|
| Scope | Finds repo topology, release surfaces, metadata policy, and review policy | Required project inputs are unknown |
| Branch verification | Confirms `development` and `staging` exist in every repository | Any branch is missing or ambiguous |
| Diff gate | Confirms the source branch has something to promote | Source and target are already equivalent |
| PR preparation | Reuses an open promotion PR or creates one with inferred title/body metadata | PR creation fails |
| Contract check | Verifies branch pair, inferred metadata, and head SHA for every PR | Metadata or branch pair is wrong |
| Readiness report | Summarizes mergeability, reviews, checks, comments, and blockers | None |
| Handoff stop | Ends before merge or deployment verification | Always stops here |

### Decision rules

- Explain each meaningful step before and after it happens, using beginner-friendly language.
- Include useful links and a clear next action whenever handing off a PR, deployment, check, or blocker.
- Reuse an existing open PR when the base is `staging` and the head is
  `development`.
- Create a PR only when the branches differ and no matching open PR exists.
- Infer PR title and body from the active lane, changed files, release surfaces, review/check expectations, and handoff owner before creating a new PR.
- Preserve source and target branches and warn the user not to click provider
  branch-deletion prompts for these permanent promotion branches.
- Report readiness; do not merge, push `staging`, deploy, or claim smoke-test
  results.
- Name the owner of post-merge verification when project policy provides one.

## Layout

```
.
+-- SKILL.md                              # entry point with PR handoff workflow
+-- agents/
|   +-- openai.yaml                       # agent / IDE descriptor
+-- references/
|   +-- development-to-staging-reference.md
|                                           # reusable command template and reporting contract
+-- assets/                               # skill icons and prompt sources
```

## Quick start

Read [`SKILL.md`](SKILL.md) first. It contains the branch defaults, required
project inputs, topology modes, PR metadata inference rules, plain-language
handoff requirements, useful-link inventory, PR creation/reuse rules, readiness
checks, and handoff report format.

Load [`references/development-to-staging-reference.md`](references/development-to-staging-reference.md)
when you need the full command template for branch discovery, PR creation or
reuse, metadata validation, and readiness inspection.

## Resources

- [SKILL.md](SKILL.md) - main PR handoff workflow and safety rules
- [agents/openai.yaml](agents/openai.yaml) - agent / IDE descriptor
- [references/development-to-staging-reference.md](references/development-to-staging-reference.md) - command template and reporting contract
- [assets/](assets/) - skill icons and icon-prompt sources

## Caveats

- When a PR URL is provided, the user should open it, read the description, inspect Files changed, check Checks, and follow the project approval/merge policy.
- When Vercel is relevant, include deployment URLs, dashboard/project links, check links, and logs when they can be discovered.
- Branch names default to `development` and `staging`, but both must be verified
  remotely before mutation.
- Source and target branches are permanent promotion branches. Do not delete
  them after the PR is created or merged unless an explicit project policy and
  user authorization require it.
- This is a handoff workflow. Stop before merge, deployment, or smoke testing.
- Do not create duplicate promotion PRs. Reuse an existing open PR when the base
  and head branches match.
- New PRs should not have empty descriptions; include inferred scope, diff evidence, handoff status, and unknowns.
- A mergeable PR can still be unready if review, checks, metadata policy, or release
  ownership is incomplete.
