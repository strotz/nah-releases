# nah

A CLI for working with stacked Git branches. Build a feature as a series of small,
dependent branches, keep them rebased, and submit and merge their pull requests.

nah supports trees, not just linear stacks. It manages its own branches and also
reads stacks created by Graphite (`gt`) and GitHub's `gh stack` extension, so you
can use them together or migrate gradually.

This repository hosts **public binaries and documentation**. The source repository
is private; you do not need access to it to install or use nah.

## Contents

- [Install](#install)
- [Quick start](#quick-start)
- [Command reference](#command-reference)
- [Working with stacks](#working-with-stacks)
- [GitHub and Graphite integration](#github-and-graphite-integration)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

## Install

Supported platforms: **macOS and Linux**, on **ARM64** and **x86-64**.
Git must be installed. Pull-request operations also require the GitHub CLI (`gh`),
authenticated with `gh auth login`. Graphite and gh-stack are optional integrations.

### Install script

```sh
curl -fsSL https://github.com/strotz/nah-releases/releases/latest/download/install.sh | sh
```

The script downloads the matching binary, verifies its SHA-256 checksum, and
installs it into `~/.local/bin`. It requires `curl`, `tar`, and either `sha256sum`
or `shasum`. Add the installation directory to your shell's `PATH` if needed:

```sh
export PATH="$HOME/.local/bin:$PATH"
nah version
```

Put the `export` line in your shell startup file (for example, `~/.zshrc` or
`~/.bashrc`) to keep it across sessions.

To choose an installation directory or pin a version, pass environment variables
to `sh`, not to `curl`:

```sh
curl -fsSL https://github.com/strotz/nah-releases/releases/latest/download/install.sh \
  | NAH_INSTALL_DIR="$HOME/bin" NAH_VERSION=v0.7.0 sh
```

Replace `v0.7.0` with the release tag you want. You can also download and inspect
`install.sh` from the release before running it.

### Manual installation

Download an archive and `checksums.txt` from the
[Releases page](https://github.com/strotz/nah-releases/releases).
Archive names follow `nah_<version>_<os>_<arch>.tar.gz`:

| Platform | OS | Architecture |
|---|---|---|
| macOS, Apple Silicon | `darwin` | `arm64` |
| macOS, Intel | `darwin` | `amd64` |
| Linux, ARM64 | `linux` | `arm64` |
| Linux, x86-64 | `linux` | `amd64` |

Verify the archive's checksum against its entry in `checksums.txt` using
`shasum -a 256 <archive>` on macOS or `sha256sum <archive>` on Linux. Then extract
it and install the `nah` executable into a directory on your `PATH`:

```sh
tar -xzf nah_0.7.0_darwin_arm64.tar.gz nah
mkdir -p "$HOME/.local/bin"
install -m 755 nah "$HOME/.local/bin/nah"
nah version
```

The archive above is an example for macOS on Apple Silicon; substitute your
version and platform.

### Upgrade

```sh
nah upgrade --check           # Check for a newer release
nah upgrade                   # Install the latest release
nah upgrade --version v0.7.0  # Install a specific release
```

## Quick start

Run nah inside an existing Git repository. Configure its trunk branch first:

```sh
nah config trunk
nah config prefix  # Optional branch-name prefix
```

Create two dependent branches, recording work on each:

```sh
nah new add-api
# Make changes to the API.
nah modify -a -m "Add API"

nah new add-ui
# Make changes to the UI.
nah modify -a -m "Add UI"

nah tree
nah submit
```

`nah submit` pushes the chain from trunk to the current branch and creates draft
PRs by default. Use `nah publish` to submit them ready for review instead.

To change the lower branch and update its descendants:

```sh
nah checkout              # Pick add-api from the interactive tree
# Make changes.
nah modify -a -m "Refine API"
nah submit                # Include descendants when prompted
```

To update from trunk, preview the operation first:

```sh
nah sync --dry-run
nah sync
```

## Command reference

Run `nah <command> --help` for flags and usage. `--verbose` (`-v`) enables
verbose output.

| Command | Purpose |
|---|---|
| `nah new <name>` | Create a child of the current branch and switch to it. |
| `nah tree` | Show the whole branch tree rooted at trunk. `--pr` adds PR status. |
| `nah path` | Show only the current branch's chain down to trunk. Supports `--pr`. |
| `nah checkout [branch]` | Switch by name or pick from the tree. Alias: `co`. `--path` limits the picker to the current chain. |
| `nah split [name]` | Move the top commit into a new child branch; `-n N` moves N commits. Prompts for a name if omitted. Existing children move under the new branch. |
| `nah modify` | Commit changes and restack descendants. `-a` stages all changes, `-u` stages tracked changes, `-p` stages interactively, `-m` sets the message. `--amend` amends; `--commit` (`-c`) creates a commit. |
| `nah restack` | Rebase the current branch and descendants onto their parents. `--all` covers the whole tree. `--continue` / `--abort` handle conflicts. |
| `nah track [branch]` | Record a branch as managed by nah. `--parent P` changes its recorded parent; `--gh` converts a linear chain to a GitHub stack. |
| `nah migrate` | Migrate Graphite branches to nah and convert each tree's mainline to a GitHub stack. `--dry-run` previews; `--no-gh` skips GitHub conversion. |
| `nah sync` | Fetch and fast-forward trunk, handle merged/closed branches, and restack. Supports `--all`, `--submit`, `--delete`, `--no-delete`, `--remote`, and `--dry-run`. |
| `nah submit` | Push the current chain with `--force-with-lease` and create or retarget PRs. `--upstack` / `--no-upstack` controls descendants; `--no-pr` only pushes. Supports `--remote` and `--dry-run`. |
| `nah publish` | Alias for `nah submit --publish`: create ready-for-review PRs and mark existing drafts ready. |
| `nah merge` | Show PR readiness and merge bottom-up. Supports `--dry-run`, `--yes`, `--publish`, and `--squash` / `--merge` / `--rebase`. Default: squash. |
| `nah doctor` | Diagnose stale stack metadata, missing branches, empty branches, and restack needs. `--fix` repairs supported catalog inconsistencies. |
| `nah config` | Configure trunk, prefix, display, backend, sync, and modify preferences. |
| `nah version` | Print the version, commit, and build date. |
| `nah upgrade` | Update the binary. `--check` only checks; `--version TAG` selects a release. |

## Working with stacks

### Branch relationships

nah stores its own links in Git config as `branch.<name>.nahParent` and
`branch.<name>.nahBase`. It also reads Graphite and gh-stack metadata, then falls
back to inferring relationships from commit history. Precedence is **nah,
Graphite, gh-stack, Git history**.

Branches shown as `[git]` have inferred relationships. Use `nah track` to record
them explicitly before trunk moves. `nah track --parent <branch>` records a new
parent; use `nah restack` to rebase onto that parent.

The tree marks branches that need restacking and frozen Graphite branches.
Trunk's `↓N` indicator reports how far it trails its remote-tracking branch as of
the last fetch. `nah tree --pr` fetches trunk and adds PR numbers, states,
reviews, and checks.

### Editing and resolving conflicts

`nah modify` creates a new commit by default, then restacks descendants. Configure
`nah config modify` to prefer amending, or choose per invocation with `--amend`
or `--commit`. A branch with no commits of its own always gets a new commit.
Without `-m`, a new commit opens your editor; an amend keeps its existing message.

Leftover unstaged and untracked changes are stashed during modify's restack and
restored when it completes, including after continue or abort.

If restacking stops at a conflict:

```sh
# Resolve conflicted files, then stage them.
git add <resolved-files>
nah restack --continue
```

Use `nah restack --abort` to abort the operation instead.

### Syncing with trunk

By default, `nah sync` operates on the current chain and its descendants.
`--all` includes every branch on trunk. It:

1. Fetches and fast-forwards trunk, stopping if local trunk has diverged.
2. Handles merged or closed branches according to `--delete`, `--no-delete`, or
   the configured preference (default: ask), re-parenting children as needed.
3. Restacks surviving branches. Conflicting rebases are aborted for those
   branches; their descendants are skipped while other branches continue.
4. With `--submit`, pushes the resulting scope and updates PRs. Submission is
   skipped if any branch could not be restacked.

**Deletion can remove local branches even for closed, unmerged PRs.** Review the
plan with `nah sync --dry-run` and use `--no-delete` to retain branches.
Dry-run still fetches remote refs, but does not apply the planned branch changes.

### Submitting and merging

Submission refuses branches that are not rebased onto their parents. Run
`nah restack` first. By default it creates draft PRs and retargets existing PRs
when their parent changes. Use `--publish` for ready-for-review PRs, `--draft`
to force drafts, or `--no-pr` for pushing only.

`nah merge` merges ready PRs bottom-up, delegating to `gh stack merge` or
`gt merge` for those backends. For native nah branches it uses `gh pr merge`,
updating trunk and rebasing, pushing, and retargeting remaining branches between
merges. Preview with `nah merge --dry-run`.

## GitHub and Graphite integration

### GitHub stacks

Requires authenticated `gh` and the gh-stack extension:

```sh
gh auth login
gh extension install github/gh-stack
```

Use `nah config backend` to select `gh` for new stacks from trunk, or convert an
existing linear chain with `nah track --gh`. GitHub stacks are linear: creating
a branch from a non-top stack member creates a native nah side branch instead.

Submission delegates to `gh stack submit --auto` and reconciles the remote stack
with the local catalog. If stacked PRs are not enabled for the repository,
submission falls back to a plain Git push.

### Graphite stacks

For Graphite-managed branches, nah delegates branch creation to `gt create`,
modification to `gt modify`, submission to `gt submit`, and merging to `gt merge`.
These operations require `gt`. Graphite's own prefix applies to `gt create`.

Frozen Graphite branches are treated as someone else's work: nah does not rebase
or submit them. Sync fetches them and updates their local tips to match the
remote, including force-pushed updates; their children can still be restacked.
Use `gt unfreeze` before taking ownership.

### Migrating from Graphite

```sh
nah migrate --dry-run
nah migrate
```

Migration records Graphite branches as nah branches and untracks them in
Graphite. It then converts each tree's deepest root-to-leaf chain into a GitHub
stack, leaving side branches managed by nah. Frozen branches and their subtrees
are left alone. Existing commits and PRs are retained.

Use `nah migrate --no-gh` to migrate to native nah metadata only. After migrating,
use `nah config backend` to choose how future stacks start from trunk.

## Configuration

Interactive configuration:

```sh
nah config trunk
nah config prefix
nah config tree
nah config backend
nah config sync
nah config modify
```

Settings use Git config. For example, `git config nah.remote upstream` sets a
repository-local default remote; use Git's `--global` option for shared defaults
where appropriate.

| Key | Meaning |
|---|---|
| `nah.trunk` | Trunk branch. |
| `nah.prefix`, `nah.prefixDate` | Branch prefix and whether to append today's date. |
| `nah.tree.trunkInfo`, `nah.tree.status`, `nah.tree.pr` | Include trunk information, worktree status, and PR information in tree output. |
| `nah.backend` | Backend for new stacks from trunk: `nah` (default), `gh`, or `gt`. Children inherit their parent's backend where supported. |
| `nah.remote` | Default remote; otherwise trunk's upstream remote, then `origin`. Per-command `--remote` takes precedence where supported. |
| `nah.publish` | Submit PRs ready for review by default. |
| `nah.mergeMethod` | Default merge method: `squash`, `merge`, or `rebase`. |
| `nah.sync.all` | Sync the whole tree by default. |
| `nah.sync.submit` | Submit after syncing by default. |
| `nah.sync.delete` | Merged/closed branch deletion policy: `ask`, `always`, or `never`. |
| `nah.modify` | Default modify mode: `commit` or `amend`. |

## Troubleshooting

- **`nah: command not found`:** add the installation directory to `PATH` and
  restart your shell, or source its startup file.
- **A branch needs restacking:** run `nah restack`, or `nah restack --all` for the
  entire tree.
- **PR operations fail:** check `gh auth status` and repository access. GitHub
  stacks additionally require the gh-stack extension; Graphite operations need `gt`.
- **Stack metadata looks wrong:** run `nah doctor` to inspect it; use
  `nah doctor --fix` for supported repairs after reviewing the diagnosis.
- **Unexpected Git behavior:** nah normally reads repositories in-process using
  go-git. Set `NAH_GIT=exec` to use the system `git` binary for every operation,
  for example `NAH_GIT=exec nah tree`.

For command-specific details, run `nah <command> --help`.
