# GPG commit signing

Setting up GPG (or SSH) keys for signing git commits and tags. Covers key generation, git configuration, and integration with GitHub/GitLab verified badges.

## Topics to cover

- GPG key generation and management
- Configuring git to sign commits and tags
- Per-directory signing keys (complementing the includeIf setup from ssh_git_config.md)
- GitHub / GitLab GPG key registration
- SSH keys as signing keys (git 2.34+) as alternative to GPG
- Verifying signed commits locally
- Automating signing (commit.gpgsign = true)
- GUI client compatibility

## Related

- [README.md](README.md) - directory-based identity management
