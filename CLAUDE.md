# Claude Code Guidelines

Always follow these guidelines strictly and without exception. Before completing any task, verify that all changes adhere to these rules.

## Task Completion

When a user queues multiple tasks in a single message, follow through on all queued tasks automatically without requiring additional reminders.

## Pre-existing Issues

If you encounter pre-existing issues (bugs, incorrect comments, dead code, lint warnings, etc.)
while working on a task, always fix them. Do not skip them or label them as "pre-existing" to
justify leaving them broken. Every file you touch should be left better than you found it.

## CRITICAL: No External Repository Interactions

NEVER create PRs, issues, comments, or any other interactions on GitHub repositories that are
not owned by the user unless given explicit permission. This includes forked upstream repos,
third-party dependencies, and any repo not under the user's GitHub account. Always confirm
before interacting with external repositories.

## CRITICAL: Solve Problems at the Source

Never apply per-machine workarounds, local config hacks, or quick fixes that only work on one
computer. Always fix the root cause in the source code so it works automatically for everyone.
Tools and features must function correctly out of the box on any machine — if it requires
manual settings or local file edits to work, it's not a real solution. Think about the team:
every fix should be a permanent, portable software engineering solution, not a band-aid.

## CRITICAL: Clean Up After Fixing Difficult Bugs

When a difficult bug requires multiple exploratory attempts, always go back through all
changes after the fix is working and remove anything redundant. Keep only the minimal set of
changes needed for the fix. Verify after each removal that things still work. Exploratory
scaffolding, abandoned approaches, and debug artifacts must not be left in the codebase.

## CRITICAL: No Environment Assumptions

When working on a repository, never assume you can utilize context about the user's personal
environment, shell configuration, or other repositories. Each repo must be self-contained and
modular, only utilizing dependencies that are explicitly declared. Do not reference or depend
on tools, configs, or conventions that are not part of the project's own dependency tree.

## CRITICAL: No Hardcoded Paths

NEVER hardcode user-specific or machine-specific paths. This is absolutely unacceptable and unprofessional. Examples of what to NEVER do:

- Home directories: `/home/username`, `/Users/username`
- Tool storage paths: `~/.rokit/tool-storage/username/toolname/version/binary`
- Any absolute path that only exists on one specific machine

Always use:

- Environment variables (e.g., `process.env.HOME`)
- Relative paths
- Tool shims that resolve paths dynamically (e.g., just `lune` instead of the full path)
- Standard system paths that exist on all machines

Code must work on any machine, including CI environments. If you find yourself typing a username or a path from your local machine, stop immediately.

## CRITICAL: No Default Values in Configuration

NEVER hardcode default fallback values in configuration files (Docker compose, env vars,
infrastructure configs, etc.). All values must come from environment variables or `.env`
files — no `${VAR:-default}` patterns, no fallback constants, no "sensible defaults."

If a required value is missing, the service must fail to start with a clear error message,
not silently use a default. This applies to:

- Docker compose `command:` and `environment:` sections
- Port numbers, tokens, URLs, database paths
- Any configuration that varies between environments

The `.env` file (or GitHub Secrets in CI) is the single source of truth for all
configuration. If a value isn't there, it's a deployment error that should be caught
immediately — not papered over with a default that might be wrong.

## README Format

- The title should be the exact repository name in lowercase kebab-case
- The first sentence of the description must exactly match the GitHub repository description
- All GitHub repository descriptions should end with a period `.`
- Additional details can follow the description (usage examples, etc.)
- READMEs should be version-agnostic — never include exact version numbers in install
  commands, dependency snippets, or usage examples. Use `@latest`, tool-managed install
  commands (e.g., `rokit add`, `wally install`), or omit versions entirely. Version numbers
  change frequently and stale examples are worse than no examples.

## Git Workflow

### Submodule URLs

Always use SSH URLs (`git@github.com:`) in `.gitmodules` files, never HTTPS
(`https://github.com/`).

### Branch Strategy

- **main**: Protected branch, requires PR with passing checks, squash merge only
- **release**: Protected branch, requires PR with passing checks, squash merge only
- **pre-release**: Used to resolve merge conflicts before PRing to release

**Repository merge settings** (enforced via Terraform for all repos):
- `allow_squash_merge = true`
- `allow_merge_commit = false`
- `allow_rebase_merge = false`
- `squash_merge_commit_title = "PR_TITLE"`
- `squash_merge_commit_message = "PR_BODY"`
- `delete_branch_on_merge = true`

This ensures squash merge is the only option. The PR title becomes the commit
message, keeping branch history clean with one commit per PR.

### CI Checks

All CI checks must be required and blocking. Every workflow job that runs on PRs must be
listed as a required status check in branch protection. No non-blocking or optional checks —
if a check runs, it must pass before merging.

### Branch Naming

Feature branches must follow this format: `<prefix>/<lowercase-kebab-case-name>`

Valid prefixes:
- `feature/` - New features
- `bugfix/` - Bug fixes
- `hotfix/` - Urgent fixes
- `chore/` - Maintenance tasks
- `docs/` - Documentation updates
- `refactor/` - Code refactoring
- `test/` - Test additions or modifications

Examples:
- `feature/add-dark-mode`
- `bugfix/fix-login-error`
- `chore/update-dependencies`

**Important**: Do NOT use dots in branch names. Version numbers like `0.1.0` are not valid in
the suffix. Use `chore/bump-version` instead of `chore/release-0.1.0`.

### Development Workflow

1. Create a feature branch from main (e.g., `feature/my-feature`)
2. Make changes and commit
3. Create a PR to main and request a review from GitHub Copilot
4. PR must pass all checks (format, test, static analysis, branch naming)
5. Squash merge to main
6. Run the post-work cleanup checklist (see below)

**CRITICAL: Always branch from `origin/main`, never local `main`.** Local main may have
unpushed commits that will contaminate the new branch. Before creating a branch, always run
`git fetch origin` and branch from `origin/main`:
```
git fetch origin
git checkout -b feature/my-feature origin/main
```
This prevents unpushed local commits from leaking into PRs where they don't belong.

### Post-Work Cleanup

After completing work in a repository (merging a PR, finishing a release, completing a task),
always run through this checklist before moving on. The repo should be left in a fully clean
state with no loose ends.

1. **Switch to main and pull**: `git checkout main && git pull`
2. **Check branch**: Verify you are on `main` (not a stale feature branch)
3. **Check working tree**: `git status` should show nothing — no modified files, no untracked
   files, no staged changes
4. **Check submodules**: `git submodule status` should show no `+` prefixes (which indicate
   the submodule is at a different commit than what the superproject expects). If dirty, run
   `git submodule update --recursive` to reset them, or commit the pointer updates if they
   are intentional
5. **Check for unpushed commits**: `git log origin/main..HEAD` should be empty
6. **Delete merged local branches**: Clean up any local feature branches that have been merged
   (`git branch --merged main | grep -v main | xargs git branch -d`)

If any step reveals issues, fix them before moving on. Do not leave a repo in a half-clean
state. This applies after any workflow that ends with merged changes.

### Release Workflow

1. Update VERSION file with new version
2. Update CHANGELOG.md with release notes
3. Create or update `pre-release` branch from main
4. Merge `release` into `pre-release` to handle any conflicts
5. Resolve conflicts if any (always keep main's version - overwrite everything from release)
6. Create a PR from `pre-release` to `release`
7. PR title must be exactly `Release X.Y.Z` (matching the VERSION file)
8. PR description should only contain Summary and Changelog sections (no "Test plan" checkboxes)
9. PR must pass all checks including:
   - All standard checks (format, test, static analysis)
   - Version validation (proper semver bump)
   - Changelog entry validation
   - Diff check (PR content must match main exactly)
10. Squash merge to release
11. GitHub Actions automatically creates the release and tag

**Conflict resolution**: When merging from main to release, always resolve conflicts by keeping
main's version. The release branch should be an exact copy of main at release time.

**Branch protection note**: If merge fails with "required status checks are expected", the
branch protection rules may have outdated check names. Check names must match the `name:`
field in workflow jobs (e.g., "Check formatting" not "format").

### PR Monitoring

**CRITICAL: Never leave a PR open. Every PR you create must be seen through to merge.**
After opening a PR, continuously poll its status until it is fully merged:

1. Poll CI check results every 30-60 seconds until all checks complete
2. If checks fail, read the CI logs, investigate the failure, fix the issue, and push updates
3. Keep iterating — fix, push, poll, repeat — until all checks pass
4. Wait for GitHub Copilot's review before merging. If you need to manually re-request a
   review (e.g. after pushing a fix), just comment `@copilot` in the PR:
   `gh pr comment <PR> --body "@copilot"`. GitHub's @ autocomplete will not suggest the
   handle, type it manually.
5. Once all checks pass and Copilot has reviewed, merge immediately
6. After merging, verify the PR state is MERGED before moving on
7. Do not end the conversation or move to the next task while a PR is still open
8. If a PR is truly blocked by an external issue (e.g., upstream bug), explicitly tell the
   user and get confirmation before deferring it

### PR Content

When creating a PR, review the actual diff (`git diff main...HEAD`) and all commits on the
branch — not just the latest commit. PRs often combine multiple commits, so the title and
description must describe the aggregate of all changes. Double-check that the PR title and
body accurately reflect what the diff contains.

## Commits

- Always break commits down into logical parts
- Do not co-author yourself in commits
- Use sentence case for commit messages (start with a capital letter)
- Do NOT use conventional commit prefixes like `test:`, `chore:`, `feat:`, `fix:`, etc.

## File and Folder Naming

- Match the dominant naming style already used in the repository. Consistency within a
  repo is the priority — every folder and file should follow the same convention.
- If the repo predominantly uses PascalCase (e.g., Luau projects with `Source/`,
  `Scripts/`), use PascalCase for all folders and files, even for non-Luau files like
  Terraform (.tf), configuration, and infrastructure code (e.g., `Terraform/Main.tf`).
- If the repo predominantly uses kebab-case (e.g., web/CSS/TypeScript projects), use
  kebab-case for all folders and files (e.g., `src/my-module.ts`).
- Do not mix styles within a single repo. If you find inconsistency, standardize on the
  dominant style rather than preserving the exception.
- Exceptions: files that must be a specific case for tooling compatibility (e.g.,
  `.gitignore`, `package.json`, `rokit.toml`, `wally.toml`, `.luaurc`, `Makefile`,
  `README.md`, `LICENSE`, `CHANGELOG.md`, `VERSION`).

## Formatting

- Always use `indent_style = space` in `.editorconfig` files — never use tabs for any
  file type
- Run `prettier --write "**/*.json"` to format all JSON files (respects `.editorconfig`)
- Every file should end in a single newline
- Text should be LF normalized
- Group all constants together without blank lines between them
- Always read through existing code to match style
- Hex color codes should always use lowercase letters (e.g., `#cc7722`, not `#CC7722`)
- Markdown code blocks must always use the full language name as the syntax hint for
  proper highlighting. Use `typescript` (not `ts`), `javascript` (not `js`),
  `bash` (not `sh`), `luau` (not `lua`), `python` (not `py`), etc. This applies
  to README files, documentation, and any markdown content.

## Comments

- All comments should word-wrap at column 90 (wrap at word boundaries, never mid-word)
- In block comments, use `> ` prefix for list items instead of indentation with spaces (e.g.,
  `> directory - The path to check` instead of `  directory - The path to check`)

## Operators

- Use compound assignment operators (`+=`, `-=`, `*=`, `/=`) instead of expanded form

## Print Statements

- Avoid using colons `:` in prints for stylistic reasons
- Structure everything in complete sentences
- Surround strings of interest in quotation marks `"`
- Use `[Usage]` instead of `Usage:` for usage messages
- Handle singular vs plural properly when displaying counts (e.g., "1 second" vs "2 seconds", "1 test suite" vs "2 test suites")

## Versioning

- Version tags should NOT have a "v" prefix (use `0.0.1`, not `v0.0.1`)
- Do NOT manually create tags - CI automatically creates tags when releasing
- Do NOT manually create GitHub releases - always let the publish workflow create them
  automatically. If the workflow fails, fix the issue and re-run the workflow.
- NEVER bump a dependency version in rokit.toml or wally.toml to a version that has not been
  fully released and published yet. Always ensure the dependency is available before
  referencing it. Bumping to an unreleased version breaks CI and other consumers.
- When releasing a tool that other repos depend on, always complete the full release workflow
  (merge, publish, verify availability) before bumping dependents to the new version.

## Changelog Format

CHANGELOG.md should follow this format. Never include a `0.0.0` entry — `0.0.0` is the
initial development version and is never released, so the changelog should start empty and
the first entry should be `0.0.1`.

```md
# Changelog

## 0.0.2
- Added a thing
- Changed a thing
- Fixed a thing

## 0.0.1
- Initial release
```

## Ordering

- When things can be sorted alphabetically, definitely do that (e.g., imports, table keys, function parameters)
- Constants should be placed above the module definition

## State Management and Persistence

Always consider what happens to in-memory state across process restarts, crashes, and
session boundaries. If runtime state (caches, selections, active modes) needs to survive
a restart, persist it to disk and load it back on startup. Think through edge cases:
> What if the process restarts mid-operation?
> What if the user switches modes and the old state is still saved?
> What if the cache is stale or from a previous configuration?

When adding a new stateful feature, always ask: "Does this break if the process restarts
right now?" If yes, persist it. When clearing or changing state, clean up all related
persisted data (not just the in-memory cache). Never leave orphaned state that causes
confusing behavior after a restart.

## Testing

Always write unit tests for new behavior. There is no class of feature that's "too
complex" or "too side-effectful" to test — authentication flows, disk reads/writes,
HTTP requests, UI rendering, and concurrency are all testable when you separate
decision logic from I/O.

Approach:

1. **Extract pure decision logic into a separate module.** If a component decides which
   onboarding steps to show, or which auth method to use, or how to merge two configs,
   that decision should live in a function that takes its inputs as arguments and
   returns a value — no I/O, no globals, no `Date.now()`. Tests then exercise it
   directly without touching the real subsystem.
2. **Mock the boundaries, not the logic.** Stub the secure-storage layer, the network
   client, the filesystem, the OAuth provider — whatever sits at the edge. Don't mock
   the function under test or assert on internal call counts; assert on observable
   outputs.
3. **Use state machines to verify truthiness.** When the feature has multiple
   conditions (e.g. "is theme picked AND is Discord authed?"), write a test that
   walks every combination of those booleans and asserts the expected branch. Loop
   over the 2^N truth table rather than picking a few cases — invariants ("X is
   always true", "Y never duplicates") catch bugs that case-by-case tests miss.
4. **Pin contracts that are easy to break.** If a function must be pure, write a test
   that calls it twice with the same input and asserts both results are equal. If a
   list must stay in a fixed order, assert the order. If an array shrinking would
   break the caller, document that with a regression test.
5. **For UI, test the data the UI consumes.** Don't try to assert on rendered pixels —
   test the helper that produces the props/state the component renders from. The
   React/Ink component itself becomes a thin shell over tested logic.

If a feature is hard to test, that's a signal to refactor — usually by extracting the
decision into a pure helper and pushing I/O to the edges. "Hard to test" is never an
acceptable reason to skip tests.

## Tools

- Use `fd` instead of `find`
- Use `rg` (ripgrep) instead of `grep`

