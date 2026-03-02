# Directory-based git identity and ssh key management

Automatically use the right git identity and ssh key for each project. This setup uses git's built-in `includeIf` feature to switch identities based on directory structure.

**Why this approach?**
- Solves ssh-agent key exhaustion (servers reject connections after trying too many keys)
- Scales well with multiple keys and identities
- Handles multiple accounts on the same host (e.g. github.com, gitlab.com, gitea.example.com) without host aliases
- Works with `git clone` -- git resolves the identity config after creating the target directory, so the correct SSH key is used for the initial fetch
- No wrapper scripts, environment variables, or external dependencies
- Works with submodules
- May not work with GUI git clients (compatibility varies by client, test with your preferred tool)

---

## Quick Setup

### 1. Create Directory Structure

Organize by git server, then by organization/group within that server:

```bash
mkdir -p ~/git/{github.com,gitlab.com,gitea.example.com}
mkdir -p ~/.gitconfigs
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
└── gitea.example.com/
    └── team/
        └── internal-tools/
```

This mirrors how git hosting services organize repositories (host > org/group > repo), making clone URLs predictable and navigation intuitive. Adding a new server or organization is just `mkdir`.

### 2. Global git config

`~/.gitconfig`

```ini
# Default identity (used when no includeIf matches)
[user]
    name = Your Name
    email = your-default@email.com

# Base rule per server (covers all orgs on that server)
[includeIf "gitdir:~/git/github.com/"]
    path = ~/.gitconfigs/work.gitconfig
[includeIf "gitdir:~/git/gitlab.com/"]
    path = ~/.gitconfigs/work.gitconfig
[includeIf "gitdir:~/git/gitea.example.com/"]
    path = ~/.gitconfigs/work.gitconfig

# Override for specific orgs that need a different identity
[includeIf "gitdir:~/git/github.com/personal-account/"]
    path = ~/.gitconfigs/personal.gitconfig
[includeIf "gitdir:~/git/github.com/client-org/"]
    path = ~/.gitconfigs/client.gitconfig
```

Rules are evaluated top to bottom. More specific paths override broader ones, so `~/git/github.com/personal-account/` wins over `~/git/github.com/`.

### 3. Identity config files

`~/.gitconfigs/work.gitconfig`

```ini
[user]
    name = Your Professional Name
    email = you@company.com

[core]
    sshCommand = ssh -i ~/.ssh/id_work
```

`~/.gitconfigs/personal.gitconfig`

```ini
[user]
    name = Your Username
    email = you@personal.com

[core]
    sshCommand = ssh -i ~/.ssh/id_personal
```

`~/.gitconfigs/client.gitconfig`

```ini
[user]
    name = Your Professional Name
    email = you@client.com

[core]
    sshCommand = ssh -i ~/.ssh/id_client
```

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

# Self-hosted git servers
# Gitea/Gogs may use 'gogs' user instead of 'git'
Host gitea.example.com
    User gogs

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

During `git clone`, the **destination directory** determines which SSH key is used:

1. git creates the target directory (e.g., `~/git/github.com/work-org/project/`)
2. git changes into it and runs `git init`
3. `includeIf "gitdir:~/git/github.com/work-org/"` matches and loads the identity config
4. git uses the correct SSH key via `core.sshCommand` for the fetch

This means the key selection happens based on where you clone **to**, not where you run the command from.

**Note**: This works reliably in git 2.43+. A regression in git 2.44 affecting `includeIf` during clone operations was quickly fixed in subsequent releases.

---

## Verification

```bash
# Check identity that would apply in this directory (works inside and outside repos)
git whoami

# Verify which SSH key git actually uses during clone (without overriding anything)
GIT_TRACE=1 git clone git@github.com:work-org/repo.git

# Check identity and SSH config inside an existing repo
cd ~/git/github.com/work-org/some-repo && git whoami
```

`GIT_TRACE=1` logs the exact SSH command git uses. Look for the `run_command:` line -- it shows which `-i` key file is passed to ssh. This is preferable to `GIT_SSH_COMMAND` for verification, because that variable overrides `core.sshCommand` and bypasses the configuration you are trying to test.

---

## Adding a new identity

When a new client or account needs a separate identity:

```bash
# 1. Create directory for the org
mkdir -p ~/git/github.com/new-client-org

# 2. Generate SSH key
ssh-keygen -t ed25519 -f ~/.ssh/id_new_client

# 3. Create config file
cat > ~/.gitconfigs/new-client.gitconfig << EOF
[user]
    name = Your Professional Name
    email = you@new-client.com
[core]
    sshCommand = ssh -i ~/.ssh/id_new_client
EOF

# 4. Add to ~/.gitconfig
echo '[includeIf "gitdir:~/git/github.com/new-client-org/"]' >> ~/.gitconfig
echo '    path = ~/.gitconfigs/new-client.gitconfig' >> ~/.gitconfig

# 5. Add public key to the git hosting account
cat ~/.ssh/id_new_client.pub
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

### URL format

Azure DevOps SSH URLs look different from standard git hosting services:

```
contoso@vs-ssh.visualstudio.com:v3/contoso/Platform/k8s-infra
```

- The SSH hostname is `vs-ssh.visualstudio.com` (not the web hostname `*.visualstudio.com`)
- The organization name appears as the SSH username (`contoso@`)
- The path is `v3/{org}/{project}/{repo}` -- `{org}` appears in both the username and the path
- The "project" level (`Platform`) maps to the second directory level under your org root

Use the web hostname for your directory structure:

```
~/git/
└── contoso.visualstudio.com/
    └── Platform/             # Azure DevOps project
        └── k8s-infra/        # repository
```

Clone from the project level:

```bash
cd ~/git/contoso.visualstudio.com/Platform
git clone contoso@vs-ssh.visualstudio.com:v3/contoso/Platform/k8s-infra
```

### SSH config and gitconfig

```ini
# ~/.ssh/config
Host vs-ssh.visualstudio.com
    IdentityFile ~/.ssh/id_rsa_azuredevops
    IdentitiesOnly yes
    IdentityAgent none
```

```ini
# ~/.gitconfig
[includeIf "gitdir:~/git/contoso.visualstudio.com/"]
    path = ~/.gitconfigs/contoso.gitconfig
```

If you work with multiple Azure DevOps organizations, each gets its own directory and gitconfig pointing to its own RSA key. The SSH hostname `vs-ssh.visualstudio.com` is shared across all Azure DevOps organizations, so key selection is handled by `core.sshCommand` rather than SSH config host entries.

---

## Debugging

git evaluates configuration in this order:

1. **Environment variables** (highest precedence) -- `GIT_SSH_COMMAND`, `GIT_AUTHOR_EMAIL`
2. **Repository config** -- `.git/config`
3. **Global config** -- `~/.gitconfig`
4. **System config** (lowest precedence) -- `/etc/gitconfig`

### Verify active identity

```bash
# Check active git identity and config resolution (works inside and outside repos)
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

**Wrong Identity**: The `includeIf` rules match on the path of the `.git` directory (the clone destination), not your current working directory. Use `git whoami` to check which identity and SSH key would apply -- it works both inside repos and in parent directories before cloning.

**GUI Clients**: Some GUI git clients may not respect `core.sshCommand`. Test with your preferred client to verify compatibility.

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

**GitHub Desktop overwrites gitconfig**: GitHub Desktop may silently add a `[user]` section to your global `~/.gitconfig`, overriding your `includeIf`-based identities. Check `~/.gitconfig` after installing or updating GitHub Desktop.

---

## Related Tools

[git-whoami](https://github.com/gitmpr/git-whoami) -- shows your effective git identity and SSH key for the current directory. Works both inside and outside git repositories. Outside a repo, it resolves `~/.gitconfig` `includeIf` rules against the current directory to show what identity and SSH key would be used when cloning there.

---

## Alternative Approaches

While the above approach is recommended, some alternatives exist:

**SSH Match with exec**: Uses `Match host github.com exec "pwd | grep /path/"` but doesn't work reliably in GUI git clients (VSCode, GitKraken, Tower, etc.) and has PATH dependencies.

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

**Environment variables with direnv**: Sets git credentials and SSH command per directory using `.envrc` files. Requires direnv installation and doesn't work with most GUI git clients.

```bash
# ~/git/github.com/work-org/.envrc
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_work -o IdentitiesOnly=yes"
export GIT_AUTHOR_NAME="Your Professional Name"
export GIT_AUTHOR_EMAIL="you@company.com"
export GIT_COMMITTER_NAME="Your Professional Name"
export GIT_COMMITTER_EMAIL="you@company.com"
```

---
