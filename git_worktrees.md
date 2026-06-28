# Git Worktrees Extension

This document describes how to extend the host/org/repo directory structure from the main guide to support git worktrees, enabling multiple branches of the same repository to be checked out simultaneously.

## Background

The standard setup in this guide organizes cloned repos at `~/git/{host}/{org}/{repo}/`. With git worktrees, this becomes `~/git/{host}/{org}/{repo}/{branch}/` -- each active branch lives in its own subdirectory, and the git object store lives in a shared `.bare/` directory at the repo level:

```
~/git/
├── github.com/
│   └── work-org/
│       └── backend-service/
│           ├── .bare/          git object store (bare clone)
│           ├── .git            file: "gitdir: ./.bare"
│           ├── main/           worktree for main branch
│           └── feature-x/     worktree for feature-x branch
└── gitlab.com/
    └── work-group/
        └── web-platform/
            ├── .bare/
            ├── .git
            └── main/
```

The identity and SSH key rules from the main guide continue to work without changes. The `hasconfig:remote.*.url` rules match against the remote URL stored in `.bare/config`, which is set during clone and remains unchanged regardless of how many worktrees exist.

## Tools

**[ghq-wt](https://github.com/gnur/ghq-wt)** is a fork of [ghq](https://github.com/x-motemen/ghq) that changes `ghq get` to clone into the bare+worktree layout instead of a standard checkout. The binary is still named `ghq`. The host/org/repo directory structure is preserved; ghq-wt adds the branch subdirectory level. `ghq list` returns 4-segment paths (`host/org/repo/branch`) to reflect this.

**[git-wt](https://github.com/ahmedelgabri/git-wt)** manages the worktree lifecycle after the initial clone. Key subcommands:

| Command | Action |
|---------|--------|
| `git wt add <branch>` | Fetch from remote, create worktree tracking `origin/<branch>` |
| `git wt add -b <name>` | Create a new local branch as a worktree |
| `git wt remove <path>` | Remove worktree and delete local branch (with confirmation) |
| `git wt migrate` | Convert an existing single-checkout repo to bare+worktree layout |
| `git wt status` | Show all worktrees and their state |
| `git wt doctor` | Diagnose configuration issues |

## The `.git` Pointer File

Each worktree subdirectory already contains a `.git` file pointing back to the bare store:

```
main/.git  →  gitdir: /abs/path/.bare/worktrees/main
```

git-wt and other tools locate the bare store from inside any worktree via `git rev-parse --git-common-dir`, which follows this chain.

There is an additional `.git` file at the repo container level (`backend-service/.git`), containing just `gitdir: ./.bare`. This serves two purposes:

1. git-wt can be invoked from the repo container rather than requiring you to cd into a worktree first
2. Editors and language servers that scan upward for a git boundary find it at the project root when the project root is opened directly

ghq-wt does not create this file. If you use ghq-wt for cloning, add it manually:

```bash
printf 'gitdir: ./.bare\n' > ~/git/github.com/work-org/backend-service/.git
```

Or use a shell wrapper that adds it automatically after every `ghq get` (see the optional tooling section below).

## Workflow

Clone a repository:

```bash
ghq get git@github.com:work-org/backend-service.git
# → ~/git/github.com/work-org/backend-service/.bare/
# → ~/git/github.com/work-org/backend-service/main/
```

Check out a second branch as a worktree (from inside any worktree):

```bash
cd ~/git/github.com/work-org/backend-service/main
git wt add feature-x
# fetches, creates ~/git/github.com/work-org/backend-service/feature-x/
```

Switch between branches by navigating the filesystem:

```bash
cd ~/git/github.com/work-org/backend-service/feature-x
```

Remove a worktree when done:

```bash
git wt remove ~/git/github.com/work-org/backend-service/feature-x
```

## Identity and SSH Keys

The `hasconfig:remote.*.url` rules match against the remote URL stored in `.bare/config`. This works correctly for existing repos cloned with plain git or git-wt. However, if you use ghq-wt for cloning, there is a URL normalization quirk that requires one extra step per git host.

### ghq-wt SSH URL normalization

ghq-wt (inheriting this behavior from upstream ghq) normalizes all SSH URLs to the `ssh://` protocol form before storing them. A clone of `git@github.com:work-org/repo.git` results in the remote being stored as:

```
ssh://git@github.com/work-org/repo
```

rather than the SCP form:

```
git@github.com:work-org/repo
```

The two forms are syntactically distinct to git's pattern matcher. If your `hasconfig` rules are written in SCP form (as shown in the main guide), they will not match remotes stored by ghq-wt.

This is a known upstream issue: [x-motemen/ghq#522](https://github.com/x-motemen/ghq/issues/522). There is currently no configuration option to preserve the original URL form.

### Workaround: per-host `insteadOf` rewrite

Add a `[url]` rewrite rule to your global git config for each git host you use with ghq-wt. This causes git to rewrite the `ssh://` form back to SCP form when reading the remote URL, so your `hasconfig` rules match correctly:

```ini
# ~/.config/git/config

[url "git@github.com:"]
    insteadOf = ssh://git@github.com/
[url "git@gitlab.com:"]
    insteadOf = ssh://git@gitlab.com/
```

With this in place, your existing `hasconfig` rules work without modification:

```ini
[includeIf "hasconfig:remote.*.url:git@github.com:work-org/**"]
    path = ~/.config/git/identities/work.gitconfig
```

**You must add one rewrite rule for every git host you clone from with ghq-wt**, including self-hosted instances. Forgetting to add the rule for a new host is the most common failure mode: clones succeed, but the wrong identity is applied silently.

The alternative is to write every `hasconfig` rule twice, once in SCP form and once in `ssh://` form, but the rewrite approach scales better as the number of hosts and identities grows.

Verify inside a worktree with `git config user.email` or `git whoami` (requires [git-whoami](https://github.com/gitmpr/git-whoami)).

## Optional: Shell Tooling

The [gitmpr/dotfiles](https://github.com/gitmpr/dotfiles) repository contains a `g` fish function that wraps ghq-wt and git-wt into a single interface:

- `g <url>` clones with `ghq get`, writes the `.git` pointer file, and cds into the default worktree
- `g` inside a bare-worktree repo opens an fzf picker for switching branches and creating or deleting worktrees

See `docs/git_worktrees.md` in that repo for details.
