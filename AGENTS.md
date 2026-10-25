# Shared Agent Guidelines

This document contains general guidelines for agents used by any contributor to
repositories where these guidelines are present.

Always follow these guidelines unless otherwise specified. Before completing any task,
verify that all changes adhere to these rules.

## Critical Guidelines (`‼️`)

These guidelines are extra important and absolutely must **not** be broken under any
circumstances.

### Always Branch From `origin/main` (`‼️`)

Local `main` may have unpushed commits that will contaminate a new branch, so **always**
branch from `origin/main` instead. Before creating a branch, fetch and branch from
`origin/main`...

```sh
git fetch origin
git worktree add .wt/feature/my-feature -b feature/my-feature origin/main
```

This prevents unpushed local commits from leaking into PRs where they don't belong.

### Always Work in a Worktree (`‼️`)

Before making **any** change to a repository, create a worktree for it and do all of the
work there. **Never** edit files directly in the repo's main checkout. See
[Worktrees](#worktrees) for how.

## Git Workflow

### Submodule URLs

Always use SSH URLs (`git@github.com:`) in `.gitmodules` files, never HTTPS
(`https://github.com/`).

### Branch Strategy

- `main` → Protected branch, requires PR with passing checks, squash merge only.
- `release` → Protected branch, requires PR with passing checks, squash merge only.
- `pre-release` → Used to resolve merge conflicts before opening a PR to `release`.

Squash merge is the only merge option, and merged branches are deleted automatically. The
PR title becomes the commit message, keeping branch history clean with one commit per PR.

### CI Checks

All CI checks must be required and blocking. Every workflow job that runs on PRs must be
listed as a required status check in branch protection. No non-blocking or optional
checks. If a check runs, it must pass before merging. The only exceptions are forks of
existing repositories with existing CI configurations.

### Branch Naming

All branches must follow the format `<prefix>/<lowercase-kebab-case-name>`. Some example
prefixes include...

- `feature` → New features.
- `bugfix` → Bug fixes.
- `hotfix` → Urgent fixes.
- `chore` → Maintenance tasks.
- `docs` → Documentation updates.
- `refactor` → Code refactoring.
- `test` → Test additions or modifications.
- `pre-release` → Pre-release staging to fix conflicts.

Here are some complete examples...

- `feature/add-dark-mode`
- `bugfix/fix-login-error`
- `chore/update-dependencies`
- `pre-release/1.2.3`

Do **not** use dots in branch names, except for the version in `pre-release` branches. Use
`chore/bump-version` instead of `chore/release-0.1.0`.

### Worktrees

Every piece of work happens in its own worktree, so the main checkout always stays clean
on `main` and several sessions can work in the same repo without stepping on each other.

**Creating a Worktree**

Put worktrees in the `.wt/` folder at the repo root, named after the branch (see
[Branch Naming](#branch-naming)). The branch `feature/add-dark-mode` lives in
`.wt/feature/add-dark-mode/`. Create the worktree and its branch from a fresh
`origin/main`, then `cd` into it...

```sh
git fetch origin
git worktree add .wt/feature/add-dark-mode -b feature/add-dark-mode origin/main
cd .wt/feature/add-dark-mode
```

Run every command and make every edit from inside the worktree from then on. If the repo
has a `.env` file, copy it into the worktree root, since worktrees don't share untracked
files with the main checkout.

**Older `.worktrees/` Folders**

Some repos still keep worktrees in `.worktrees/` instead of `.wt/`. Treat both the same
way when looking for existing worktrees, and use `.wt/` for any new ones. **Never** move
or rename an existing worktree, especially while a session might be working in it. A
repo's `.gitignore` should ignore both folders (`/.wt/` and `/.worktrees/`).

**Parallel Sessions**

Before starting, check `git worktree list` for work already in progress. If another
session is working in the repo, create a separate worktree for your task instead of
reusing theirs.

**Removing a Worktree**

Once the PR is merged, remove the worktree and its local branch as part of
[Post-Work Cleanup](#post-work-cleanup).

### Development Workflow

1. Create a worktree with a new branch from `origin/main` (see [Worktrees](#worktrees)).
2. Make changes and commit.
3. Create a PR to `main`. GitHub Copilot reviews it automatically (see
   [Copilot Reviews](#copilot-reviews)).
4. Make sure the PR passes all checks (format, test, static analysis, branch naming).
5. Squash merge to `main`.
6. Run the [Post-Work Cleanup](#post-work-cleanup) checklist.

### Pull Requests

#### PR Monitoring

After opening a PR, continuously poll its status until it is fully merged...

1. Poll CI check results every 30 to 60 seconds until all checks complete.
2. If checks fail, read the CI logs, investigate the failure, fix the issue, and push
   updates.
3. Keep iterating (fix, push, poll, repeat) until all checks pass.
4. Wait for a clean [Copilot review](#copilot-reviews) of the latest commit before
   merging.
5. Once all checks pass and Copilot's review is clean, merge immediately.
6. After merging, verify the PR state is `MERGED` before moving on.
7. Do not end the conversation or move to the next task while a PR is still open.
8. If a PR is truly blocked by an external issue (e.g., an upstream bug), explicitly tell
   the user and get confirmation before deferring it.

#### Copilot Reviews

Every PR must be reviewed by GitHub Copilot before merging. Repos are configured to
request the review automatically, so in most cases you don't need to do anything to
trigger one.

**Automatic Reviews**

Copilot reviews a PR when it's opened and again after every push that changes code. The
review request appears within seconds of the push, and the review itself usually lands
within a few minutes. Pushes with no diff (e.g., empty commits) are skipped.

**Review Effort**

Every Copilot review runs at the Lite effort level, which costs several times fewer AI
credits than Balanced. Automatic reviews follow the effort level in the repo's Copilot
code review settings, but reviews requested through the REST or GraphQL API ignore that
setting and always run at Balanced. The API has no working effort parameter yet, so
**never** request a Copilot review through the API.

**Checking Review Status**

Before requesting anything manually, check where the review stands. This command shows the
head commit, whether a Copilot review is pending, which commit Copilot last reviewed, and
how many review threads are still unresolved...

```sh
gh api graphql -f o=<owner> -f r=<repo> -F n=<PR> -f query='
  query($o: String!, $r: String!, $n: Int!) {
    repository(owner: $o, name: $r) {
      pullRequest(number: $n) {
        headRefOid
        reviewRequests(first: 10) { nodes { requestedReviewer { ... on Bot { login } } } }
        reviews(last: 30) { nodes { author { login } commit { oid } } }
        reviewThreads(first: 100) { pageInfo { hasNextPage } nodes { isResolved } }
      }
    }
  }' -q '.data.repository.pullRequest | {
    head: .headRefOid,
    pending: ([.reviewRequests.nodes[].requestedReviewer.login]
      | index("copilot-pull-request-reviewer") != null),
    reviewed: ([.reviews.nodes[] | select(.author.login == "copilot-pull-request-reviewer")]
      | last | .commit.oid),
    unresolved: (if .reviewThreads.pageInfo.hasNextPage then "100+"
      else ([.reviewThreads.nodes[] | select(.isResolved | not)] | length) end)
  }'
```

Here's how to read the result...

- **Pending** → `pending` is `true`. A review is already queued or in progress, so wait.
- **Clean** → `reviewed` equals `head` and `unresolved` is `0`. The PR is ready to merge
  once CI passes.
- **Has comments** → `reviewed` equals `head`, but `unresolved` is above `0`. Address the
  comments (see below), and push any fixes, which triggers a new review.
- **Outdated** → `reviewed` doesn't equal `head` and `pending` is `false`. The automatic
  review didn't fire, so get one as described below.

Don't rely on `reviewDecision` or the review text to judge whether a review is clean.
Copilot's reviews always land as `COMMENTED`, never `APPROVED`, and the summary format
changes often (older guidance looked for "generated no new comments", which no longer
appears).

**Getting a Missing Review**

Only do this when the review is outdated and nothing is pending. Since API requests run at
Balanced (see Review Effort above), don't request one through the API. Instead, ask the
user to request it from the PR page with Lite selected in Copilot's effort picker. Closing
and reopening the PR doesn't trigger a review either. If no review arrives after that, the
repo may not have Copilot reviews set up at all, so tell the user rather than retrying in
a loop.

If Copilot's review says "the job was not started because a GitHub Actions budget is
preventing further Actions use", the account is out of Actions minutes. Tell the user
instead of retrying.

**Unreviewable PRs**

If Copilot's review says it "wasn't able to review any files", the PR only touches files
Copilot skips (e.g., `.gitignore`). That counts as a clean review, so merge once CI
passes. A review saying Copilot "encountered an error" is different, so tell the user and
ask them to request it again from the PR page at Lite.

**Never Comment `@copilot`**

A `@copilot` mention doesn't request a review. It wakes Copilot's coding agent
(`copilot-swe-agent[bot]`), which pushes its own commits to the branch. If that agent ever
pushes to your branch, stop and tell the user.

**Addressing Comments**

Read every inline comment and the review body, then reply to each comment individually.
For valid suggestions, make the fix, push, and reply with the commit SHA. For
disagreements, reply with a clear technical argument. Resolve every thread (via the
GraphQL `resolveReviewThread` mutation) so that none are left open at merge time.

### Post-Work Cleanup

After completing work in a repository (merging a PR, finishing a release, completing a
task), always run through this checklist before moving on. The repo should be left in a
fully clean state with no loose ends.

1. **Remove the worktree** → From the main checkout, run
   `git worktree remove .wt/<prefix>/<name>` and then `git branch -D <prefix>/<name>`.
   Only do this after confirming the PR is `MERGED`.
2. **Switch to main and pull** → Run `git checkout main && git pull`.
3. **Check branch** → Verify you are on `main`, not a stale feature branch.
4. **Check working tree** → `git status` should show nothing. There should be no modified
   files, no untracked files, and no staged changes.
5. **Check submodules** → `git submodule status` should show no `+` prefixes, which
   indicate the submodule is at a different commit than what the superproject expects. If
   dirty, run `git submodule update --recursive` to reset them, or commit the pointer
   updates if they are intentional.
6. **Check for unpushed commits** → `git log origin/main..HEAD` should be empty.
7. **Check for leftover worktrees** → `git worktree list` should only show worktrees that
   are still in use.

If any step reveals issues, fix them before moving on. Do not leave a repo in a half-clean
state. This applies after any workflow that ends with merged changes.

### Release Workflow

1. Update the `VERSION` file with the new version.
2. Update `CHANGELOG.md` with release notes.
3. Create or update the `pre-release` branch from `main`.
4. Merge `release` into `pre-release` to handle any conflicts.
5. Resolve conflicts if there are any, always keeping main's version and overwriting
   everything from `release`.
6. Create a PR from `pre-release` to `release`.
7. Set the PR title to exactly `Release X.Y.Z`, matching the `VERSION` file.
8. Limit the PR description to Summary and Changelog sections, with no "Test plan"
   checkboxes.
9. Make sure the PR passes all checks, including...
   - All standard checks (format, test, static analysis).
   - Version validation (proper semver bump).
   - Changelog entry validation.
   - Diff check (PR content must match `main` exactly).
10. Squash merge to `release`.
11. Let GitHub Actions automatically create the release and tag.

**Conflict Resolution**

When merging from `main` to `release`, always resolve conflicts by keeping main's version.
The `release` branch should be an exact copy of `main` at release time.

**Branch Protection Note**

If a merge fails with "required status checks are expected", the branch protection rules
may have outdated check names. Check names must match the `name:` field in workflow jobs
(e.g., "Check formatting", not "format").

### Versioning

- Do **not** prefix tags with a `v` (use `0.0.1`, not `v0.0.1`).
- Do **not** manually create tags. CI automatically creates tags when releasing.
- Do **not** manually create GitHub releases. Always let the publish workflow create them
  automatically. If the workflow fails, fix the issue and re-run the workflow.
- **Never** bump a dependency version in `rokit.toml` or `wally.toml` to a version that
  has not been fully released and published yet. Always ensure the dependency is available
  before referencing it. Bumping to an unreleased version breaks CI and other consumers.
- When releasing a tool that other repos depend on, always complete the full release
  workflow (merge, publish, verify availability) before bumping dependents to the new
  version.

## Code Style

### File & Folder Naming

- Match the dominant naming style already used in the repository. Consistency within a
  repo is the priority, so every folder and file should follow the same convention.
- If the repo predominantly uses PascalCase (e.g., Luau projects with `Source/`,
  `Scripts/`), use PascalCase for all folders and files, even for non-Luau files like
  Terraform (`.tf`), configuration, and infrastructure code (e.g., `Terraform/Main.tf`).
- If the repo predominantly uses kebab-case (e.g., web/CSS/TypeScript projects), use
  kebab-case for all folders and files (e.g., `src/my-module.ts`).
- Do not mix styles within a single repo. If you find inconsistency, standardize on the
  dominant style rather than preserving the exception.
- Files that must be a specific case for tooling compatibility are exceptions (e.g.,
  `.gitignore`, `package.json`, `rokit.toml`, `wally.toml`, `.luaurc`, `Makefile`,
  `README.md`, `LICENSE`, `CHANGELOG.md`, `VERSION`).

### Formatting

- Always use `indent_style = space` in `.editorconfig` files. Never use tabs for any file
  type.
- Only use Prettier when it's available, either as a dependency of the repo (e.g., in
  `package.json`) or installed on the machine. Prefer the repo's own version and config
  when both exist, and don't add Prettier to a repo just to format files.
- When Prettier is available, run `prettier --write "**/*.json"` to format all JSON files
  (respects `.editorconfig`).
- Every file should end in a single newline.
- Text should be LF normalized.
- Group all constants together without blank lines between them.
- Always read through existing code to match style.

### Ordering

- When things can be sorted alphabetically, always sort them (e.g., imports, table keys,
  function parameters).
- Constants should be placed above the module definition.

### Comments

- All comments should word-wrap at column 90, wrapping at word boundaries and never
  mid-word.
- In block comments, use a `> ` prefix for list items instead of indenting with spaces
  (e.g., `> directory - The path to check` instead of `  directory - The path to check`).

## Document Formats

These rules cover specific kinds of files. In any document, always write directory paths
with a trailing slash (e.g., `.vscode/`, `src/`, `.github/workflows/`). The slash makes it
clear that the path is a directory and not a file without an extension.

### README.md

- The title should be the exact repository name in lowercase kebab-case.
- The first sentence of the description must exactly match the GitHub repository
  description.
- All GitHub repository descriptions should end with a period (`.`).
- Additional details can follow the description (usage examples, etc.).
- Keep READMEs version-agnostic. **Never** include exact version numbers in install
  commands, dependency snippets, or usage examples. Use `@latest`, tool-managed install
  commands (e.g., `rokit add`, `wally install`), or omit versions entirely. Version
  numbers change frequently, and stale examples are worse than no examples.
- Do **not** include a License section (e.g., `## License` followed by "MIT"). The
  `LICENSE` file already covers it.

### CHANGELOG.md

If a repo already has an established changelog format (e.g., Keep a Changelog sections,
date-based versions, or a format enforced by its CI), follow that format instead. Look at
the existing entries and any changelog checks in CI before writing a new entry. Otherwise,
use the format below.

**Never** include a `0.0.0` entry. Version `0.0.0` is the initial development version and
is never released, so the changelog should start empty and the first entry should be
`0.0.1`.

```markdown
# Changelog

## 0.0.2

- Added a thing.
- Changed a thing.
- Fixed a thing.

## 0.0.1

- Initial release.
```
