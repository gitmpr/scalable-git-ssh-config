# Remote URL-based git identity and SSH key management

Automatically use the right git identity and SSH key for each project. This setup uses git's `includeIf "hasconfig:remote.*.url"` feature to switch identities based on remote URL, so the correct key is selected regardless of where a repo is cloned.

**Why this approach?**
- Solves SSH agent key exhaustion (servers reject connections after trying too many keys)
- Scales well with multiple keys and identities
- Handles multiple accounts on the same host (e.g. github.com, gitlab.com, gitea.example.com) without host aliases
- Works with `git clone` -- git writes the remote URL to `.git/config` before the fetch, so `core.sshCommand` is resolved correctly for the initial fetch
- Identity follows the remote URL, not the clone location -- repos cloned anywhere get the right identity automatically
- No wrapper scripts, environment variables, or external dependencies
- Works with submodules
- Requires git 2.36+

---

## Quick Setup

### 1. Create Directory Structure

Organize by git server, then by organization/group within that server:

```bash
mkdir -p ~/git/{github.com,gitlab.com,bitbucket.org,gitea.example.com}
mkdir -p ~/.config/git/identities
```

```
~/git/
├── github.com/
│   ├── personal-account/
│   │   ├── dotfiles/
│   │   └── side-projects/
│   ├── work-org/
│   │   ├── backend-service/
│   │   └── infrastructure/
│   └── client-org/
│       └── client-project/
├── gitlab.com/
│   └── work-group/
│       ├── web-platform/
│       └── deployments/
├── bitbucket.org/
│   └── work-workspace/
│       ├── backend-service/
│       └── infrastructure/
└── gitea.example.com/
    └── team/
        └── internal-tools/
```

This mirrors how git hosting services organize repositories (host > org/group > repo), making clone URLs predictable and navigation intuitive. Adding a new server or organization is just `mkdir`. The directory structure is for navigation -- identity is resolved from the remote URL.

### 2. Global git config

`~/.config/git/config`

```ini
# Default identity (used when no includeIf matches)
[user]
    name = Your Name
    email = your-default@email.com

# Identity per org — matched on remote URL, not clone path
[includeIf "hasconfig:remote.*.url:git@github.com:work-org/**"]
    path = ~/.config/git/identities/work.gitconfig
[includeIf "hasconfig:remote.*.url:git@gitlab.com:work-group/**"]
    path = ~/.config/git/identities/work.gitconfig
[includeIf "hasconfig:remote.*.url:git@bitbucket.org:work-workspace/**"]
    path = ~/.config/git/identities/work.gitconfig
[includeIf "hasconfig:remote.*.url:git@gitea.example.com:*/**"]
    path = ~/.config/git/identities/work.gitconfig

# Personal account on github.com overrides the default identity
[includeIf "hasconfig:remote.*.url:git@github.com:personal-account/**"]
    path = ~/.config/git/identities/personal.gitconfig

# Client with separate identity
[includeIf "hasconfig:remote.*.url:git@github.com:client-org/**"]
    path = ~/.config/git/identities/client.gitconfig
```

The pattern after `hasconfig:remote.*.url:` is matched against the remote URL using fnmatch glob syntax. Use `org/**` to match all repos in an org, or `*/**` to match all repos on a server.

### 3. Identity config files

`~/.config/git/identities/work.gitconfig`

```ini
[user]
    name = Your Professional Name
    email = you@company.com

[core]
    sshCommand = ssh -i ~/.ssh/id_work
```

`~/.config/git/identities/personal.gitconfig`

```ini
[user]
    name = Your Username
    email = you@personal.com

[core]
    sshCommand = ssh -i ~/.ssh/id_personal
```

`~/.config/git/identities/client.gitconfig`

```ini
[user]
    name = Your Professional Name
    email = you@client.com

[core]
    sshCommand = ssh -i ~/.ssh/id_client
```

**Personal account email**: GitHub and GitLab both offer a noreply address that keeps your real email private while still linking commits to your account. Find yours in account settings under the email section:

- GitHub: `123456+username@users.noreply.github.com`
- GitLab: `123456+username@users.noreply.gitlab.com` (or just `username@users.noreply.gitlab.com` for older accounts)

Using the noreply address is recommended for personal accounts if your commits are public. Work accounts typically use the company email directly.

### 4. SSH key generation

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_work
ssh-keygen -t ed25519 -f ~/.ssh/id_personal
ssh-keygen -t ed25519 -f ~/.ssh/id_client
```

**Important**: Public git hosting services (GitHub, GitLab, Bitbucket) enforce SSH key uniqueness per account -- each public key can only be registered to a single account on that service. You cannot reuse the same key pair across multiple accounts on the same host. Generate a separate key pair for each account.

**Azure DevOps exception**: As of 2026, Azure DevOps only accepts RSA keys. Use `ssh-keygen -t rsa -b 3072` for any key intended for Azure DevOps. See the [Azure DevOps SSH](#azure-devops-ssh) section for details.

### 5. SSH config

`~/.ssh/config`

```ini
# Optional: company or client specific includes
Include ~/.ssh/config.d/company-vpn
Include ~/.ssh/config.d/client-servers

# git hosting services - prevent SSH agent key exhaustion
Host github.com gitlab.com bitbucket.org
    IdentityAgent none
    IdentitiesOnly yes

# Self-hosted git servers - same agent exhaustion prevention applies
# Gitea/Gogs may use 'gogs' user instead of 'git'
Host gitea.example.com
    User gogs
    IdentityAgent none
    IdentitiesOnly yes

# Remote server administration (separate from git auth)
Host *.company.internal
    User admin
    IdentityFile ~/.ssh/id_work_admin

# Default settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

---

## How it works

During `git clone`, the **remote URL** determines which SSH key is used:

1. git creates the target directory and runs `git init`
2. git writes the remote URL to `.git/config`
3. git evaluates `includeIf "hasconfig:remote.*.url:..."` rules against the remote URL
4. The matching identity config is loaded, providing `core.sshCommand` for the fetch

This means the key selection is tied to the remote URL -- the same identity applies regardless of where on disk the repo lives.

**Note**: `hasconfig:remote.*.url` requires git 2.36+ (released April 2022).

---

## Using with ghq

[ghq](https://github.com/x-motemen/ghq) organizes cloned repositories at `{root}/{host}/{org}/{repo}` -- exactly the structure this guide uses. Set `GHQ_ROOT=~/git` (environment variable) or `ghq.root = ~/git` (git config) and ghq integrates with no additional setup:

```bash
# In git config
git config --global ghq.root ~/git

# Or as an environment variable (e.g. in ~/.bashrc or config.fish)
export GHQ_ROOT=~/git        # bash
set -gx GHQ_ROOT ~/git       # fish
```

Repos already cloned under `~/git/` are immediately visible to ghq:

```bash
ghq list                          # lists all repos ghq finds under ~/git
ghq list --full-path              # with absolute paths
```

Cloning via ghq places repos in the right directory automatically, so `hasconfig` identity rules apply from the first fetch:

```bash
ghq get git@github.com:work-org/repo.git        # → ~/git/github.com/work-org/repo/
ghq get git@gitlab.com:work-group/project.git   # → ~/git/gitlab.com/work-group/project/
```

`ghq look repo` (or a wrapper that does `cd $(ghq list --full-path repo | head -1)`) jumps directly to any repo by name.

---

## Verification

`git whoami` is a custom command from the [git-whoami](https://github.com/gitmpr/git-whoami) tool. Install it first before using it -- see the [Related Tools](#related-tools) section below.

```bash
# Check identity that would apply in this repo
git whoami

# Verify which SSH key git actually uses during clone
GIT_TRACE=1 git clone git@github.com:work-org/repo.git

# Check identity inside an existing repo
cd ~/git/github.com/work-org/some-repo && git whoami
```

`GIT_TRACE=1` logs the exact SSH command git uses. Look for the `run_command:` line -- it shows which `-i` key file is passed to ssh. This is preferable to `GIT_SSH_COMMAND` for verification, because that variable overrides `core.sshCommand` and bypasses the configuration you are trying to test.

---

## Adding a new identity

When a new client or account needs a separate identity:

```bash
# 1. Create directory for the org (for navigation)
mkdir -p ~/git/github.com/new-client-org

# 2. Generate SSH key
ssh-keygen -t ed25519 -f ~/.ssh/id_new_client

# 3. Create config file
cat > ~/.config/git/identities/new-client.gitconfig << EOF
[user]
    name = Your Professional Name
    email = you@new-client.com
[core]
    sshCommand = ssh -i ~/.ssh/id_new_client
EOF

# 4. Add to ~/.config/git/config
echo '[includeIf "hasconfig:remote.*.url:git@github.com:new-client-org/**"]' >> ~/.config/git/config
echo '    path = ~/.config/git/identities/new-client.gitconfig' >> ~/.config/git/config

# 5. Add public key to the git hosting account
cat ~/.ssh/id_new_client.pub
```

---

## Bitbucket

Bitbucket organizes repositories under workspaces. The SSH URL format is:

```
git@bitbucket.org:{workspace}/{repo}.git
```

Bitbucket also has an optional "projects" layer for grouping repos within a workspace, but projects are not part of the clone URL. A single `hasconfig` rule per workspace covers all repos in it:

```ini
# ~/.config/git/config
[includeIf "hasconfig:remote.*.url:git@bitbucket.org:work-workspace/**"]
    path = ~/.config/git/identities/work.gitconfig
```

---

## Azure DevOps SSH

Azure DevOps has two quirks compared to other git hosting services.

### RSA-only SSH keys

As of 2026, Azure DevOps still only accepts RSA SSH keys. Ed25519 keys (the current best-practice default) are silently rejected during key upload or authentication. Generate a dedicated RSA key for Azure DevOps:

```bash
ssh-keygen -t rsa -b 3072 -f ~/.ssh/id_rsa_azuredevops
```

The corresponding gitconfig uses it via `core.sshCommand`:

```ini
[user]
    name = Your Name
    email = you@company.com
[core]
    sshCommand = ssh -i ~/.ssh/id_rsa_azuredevops
```

### URL formats

Azure DevOps organizations can be accessed through two different URL formats:

**dev.azure.com** (current format):

```
git@ssh.dev.azure.com:v3/contoso/Platform/k8s-infra
```

- SSH hostname: `ssh.dev.azure.com`
- SSH user: `git`
- Web URL: `https://dev.azure.com/contoso/`

**{org}.visualstudio.com** (legacy format):

```
contoso@vs-ssh.visualstudio.com:v3/contoso/Platform/k8s-infra
```

- SSH hostname: `vs-ssh.visualstudio.com`
- SSH user: the organization name (`contoso@`)
- Web URL: `https://contoso.visualstudio.com/`

Both use the same path structure: `v3/{org}/{project}/{repo}`. The "project" level (`Platform`) maps to the second directory level under your org root.

An organization can be reachable on both URLs simultaneously (controlled via Organization Settings → Overview). Whichever URL you clone from becomes the remote URL stored in the repo, which determines which `hasconfig` rule matches.

### SSH config and gitconfig

```ini
# ~/.ssh/config

# Azure DevOps (dev.azure.com)
Host ssh.dev.azure.com
    IdentityFile ~/.ssh/id_rsa_azuredevops
    IdentitiesOnly yes
    IdentityAgent none

# Azure DevOps (visualstudio.com legacy)
Host vs-ssh.visualstudio.com
    IdentityFile ~/.ssh/id_rsa_azuredevops
    IdentitiesOnly yes
    IdentityAgent none
```

```ini
# ~/.config/git/config

# dev.azure.com organization (matches git@ssh.dev.azure.com:v3/contoso/...)
[includeIf "hasconfig:remote.*.url:git@ssh.dev.azure.com:v3/contoso/**"]
    path = ~/.config/git/identities/contoso.gitconfig

# visualstudio.com organization (legacy URL, matches contoso@vs-ssh.visualstudio.com:...)
[includeIf "hasconfig:remote.*.url:contoso@vs-ssh.visualstudio.com:*/**"]
    path = ~/.config/git/identities/contoso.gitconfig
```

If you work with multiple Azure DevOps organizations, each gets its own `hasconfig` rule and gitconfig. Because `hasconfig` matches on the remote URL, you don't need per-org SSH host entries -- the `core.sshCommand` in each gitconfig handles key selection.

---

## Debugging

git evaluates configuration in this order:

1. **Environment variables** (highest precedence) -- `GIT_SSH_COMMAND`, `GIT_AUTHOR_EMAIL`
2. **Repository config** -- `.git/config`
3. **Global config** -- `~/.config/git/config`
4. **System config** (lowest precedence) -- `/etc/gitconfig`

### Verify active identity

`git whoami` requires the [git-whoami](https://github.com/gitmpr/git-whoami) tool to be installed first.

```bash
# Check active git identity and config resolution
git whoami

# Show where individual config values come from
git config --show-origin user.email
git config --show-origin core.sshCommand

# List all git config in effect
git config --list
```

### Test SSH authentication

```bash
# Test SSH connectivity to git server
ssh -T git@github.com

# Verbose SSH debugging
ssh -vvv git@github.com

# Test a specific key explicitly
ssh -i ~/.ssh/id_work -T git@github.com

# Print all SSH config that would apply for a given host
ssh -G github.com
ssh -G gitea.example.com
```

### Trace git operations

```bash
# Show SSH command used by git
GIT_TRACE=1 git fetch
GIT_TRACE=1 git clone git@github.com:work-org/repo.git

# Show git config loading during operations
GIT_TRACE_SETUP=1 git clone git@github.com:work-org/repo.git

# Force specific SSH key for a single operation
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_work -vvv" git clone git@github.com:work-org/repo.git
```

### Common issues

**SSH Agent Key Exhaustion**: After an SSH agent offers several keys, some servers reject further attempts -- commonly configured at a limit of 5 keys. Ubuntu ships with an ssh-agent bundled into GNOME Keyring that is difficult to disable without breaking other applications. WSL on Windows has no agent by default but can use the native Windows ssh-agent service. Solution: set `IdentityAgent none` and `IdentitiesOnly yes` in `~/.ssh/config` for git hosting services.

**Wrong Identity**: `hasconfig` rules match on the remote URL stored in `.git/config`. Use `git whoami` ([git-whoami](https://github.com/gitmpr/git-whoami), requires installation) or `git config --show-origin user.email` to verify which identity is active. Check that the remote URL matches your `hasconfig` pattern exactly with `git remote -v`.

**No remote URL**: `hasconfig` rules only fire inside repos that have a remote configured. For local-only repos with no remote, fall back to the default identity or use `gitdir`-based rules (see [Alternative Approaches](#alternative-approaches)).

**SSH directory and file permissions**: SSH refuses to use keys with overly permissive permissions and warns with `UNPROTECTED PRIVATE KEY FILE`. Ensure:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*          # private keys
chmod 644 ~/.ssh/id_*.pub      # public keys
chmod 644 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/config
```

**HTTPS URL instead of SSH**: If `git clone` prompts for a password, you likely copied the HTTPS URL from the web interface instead of the SSH URL. Use `git@host:org/repo.git`, not `https://host/org/repo.git`.

**Wrong SSH user or port**: Most git hosting services use `git` as the SSH user (e.g., `git@github.com`). Gitea/Gogs instances may use `gogs` or a custom user. Some self-hosted instances run SSH on a non-standard port -- use `ssh://git@host:2222/org/repo.git` or configure the port in `~/.ssh/config`.

**GitHub Desktop overwrites gitconfig**: GitHub Desktop may silently add a `[user]` section to your global `~/.config/git/config`, overriding your `includeIf`-based identities. Check `~/.config/git/config` after installing or updating GitHub Desktop.

---

## Related Tools

[git-whoami](https://github.com/gitmpr/git-whoami) -- shows your effective git identity and SSH key for the current directory. Works both inside and outside git repositories.

---

## Alternative Approaches

**`gitdir`-based identity** (git built-in, all versions): Uses `includeIf "gitdir:~/git/..."` to select identity based on clone path rather than remote URL. Requires a consistent directory structure and a separate rule for any repo cloned outside the standard tree. Useful for local-only repos with no remote.

```ini
[includeIf "gitdir:~/git/github.com/work-org/"]
    path = ~/.config/git/identities/work.gitconfig
[includeIf "gitdir:~/git/github.com/personal-account/"]
    path = ~/.config/git/identities/personal.gitconfig
```

**SSH Match with exec**: Uses `Match host github.com exec "pwd | grep /path/"` but doesn't work reliably across environments and has PATH dependencies.

**Multiple host aliases**: Creates SSH aliases like `github-work`, `github-personal` to distinguish multiple accounts on the same host. This clutters SSH config and can break with submodules, but avoids any directory structure requirements.

```ini
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_work

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_personal
```

**Environment variables with direnv**: Sets git credentials and SSH command per directory using `.envrc` files. Requires direnv installation.

```bash
# ~/git/github.com/work-org/.envrc
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_work -o IdentitiesOnly=yes"
export GIT_AUTHOR_NAME="Your Professional Name"
export GIT_AUTHOR_EMAIL="you@company.com"
export GIT_COMMITTER_NAME="Your Professional Name"
export GIT_COMMITTER_EMAIL="you@company.com"
```

---
