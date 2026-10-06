# Ansible Security Update

A small, standalone Ansible project for applying **targeted security package updates** on Debian/Ubuntu servers.

Use it when a CVE affects specific packages (for example `curl` or `openssl`) and you want to update only those packages on hosts where they are already installed.

## What this does

- Connects to your servers over SSH
- Checks which of the specified packages are installed
- Updates only those installed packages to the latest available version via `apt`
- Skips packages that are not installed on a host

It does **not** install new packages or run a full system upgrade.

## Prerequisites

- [uv](https://docs.astral.sh/uv/) (installs Ansible for you — no separate Ansible setup needed)
- SSH access to your target hosts
- A user with `sudo` privileges on each target host
- Target hosts running Debian or Ubuntu (`apt` package manager)

## Project structure

```
.
├── ansible.cfg
├── inventories/
│   └── hosts.yml
├── playbooks/
│   └── security_update.yml
├── pyproject.toml
└── uv.lock
```

## Setup

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/) if you do not have it yet:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

2. Install Ansible into a local virtual environment:

```bash
uv sync
```

3. Edit `inventories/hosts.yml` and replace the example host with your own:

```yaml
all:
  children:
    webservers:
      hosts:
        myserver.example.com:
          ansible_host: 10.0.0.5
          ansible_user: ansible
```

4. Verify connectivity:

```bash
uv run ansible all -m ping
```

If your SSH user needs a password for `sudo`, add `--ask-become-pass` to playbook commands below.

## Usage

All commands use `uv run` so Ansible runs from the project's managed environment.

Update specific packages on all hosts in the inventory:

```bash
uv run ansible-playbook playbooks/security_update.yml \
  --extra-vars "packages=['curl','openssl']"
```

Limit the run to a single host:

```bash
uv run ansible-playbook playbooks/security_update.yml \
  --limit myserver.example.com \
  --extra-vars '{"packages":["curl"]}'
```

If `sudo` requires a password:

```bash
uv run ansible-playbook playbooks/security_update.yml \
  --ask-become-pass \
  --extra-vars "packages=['curl','openssl']"
```

If you store secrets in Ansible Vault (for example encrypted variables in your inventory):

```bash
uv run ansible-playbook playbooks/security_update.yml \
  --ask-vault-pass \
  --extra-vars "packages=['curl','openssl']"
```

## How it works

1. The playbook requires at least one package in `packages` (via `--extra-vars`). If none are given, it stops with a clear error.
2. It gathers installed package facts with `package_facts`.
3. For each package in your list, it runs `apt` with `state: latest` **only if that package is already installed** on the host.
4. It prints the result so you can see what changed.

Example: if you pass `packages=['curl','icinga2']` and a host has `curl` but not `icinga2`, only `curl` is updated on that host.

## Safety notes

- **Test first.** Run against a staging or test host before production.
- **Use `--limit`** to patch one host at a time when rolling out critical fixes.
- This playbook updates packages you name explicitly. It does not replace a full patch-management process or vulnerability scanning workflow.
- Review package changelogs and service impact before updating production systems. Some updates may require service restarts.

## Example: emergency CVE response

When a security advisory names affected packages:

```bash
# 1. Test on one host
uv run ansible-playbook playbooks/security_update.yml \
  --limit staging.example.com \
  --extra-vars '{"packages":["curl"]}'

# 2. Roll out to production hosts
uv run ansible-playbook playbooks/security_update.yml \
  --limit webservers \
  --extra-vars '{"packages":["curl"]}'
```

## License

Use and adapt this project freely for your own infrastructure.
