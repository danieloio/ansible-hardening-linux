# Ansible Hardening Linux

[![MIT License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-core%202.16-blue.svg)](https://www.ansible.com/)

> Project 7 of my ASIR portfolio. Ansible automation for security hardening of two already-existing Linux servers (a Zabbix Server and a Docker host), managing users, SSH, firewall, and fail2ban.

**Versión en español:** [README.md](README.md)

## Architecture

```
   Physical machine
        │ NAT
        ▼
  SRV-ANSIBLE-01 (control node)
        │ LAN Segment 1 — 192.168.100.0/24
        ├── SRV-ZABBIX  (.20) — Zabbix Server 6.4
        └── SRV-DOCKER  (.40) — Nginx, MariaDB, phpMyAdmin, Portainer
```

`SRV-ANSIBLE-01` has a dual interface (NAT + LAN), like a bastion host. Both managed servers already existed from earlier portfolio projects — the goal here wasn't to build infrastructure, but to harden it.

## Stack

Ansible Core 2.16 · Ubuntu Server 22.04/24.04 · VMware Workstation · ED25519 SSH keys · UFW · Fail2ban

## Repo structure

```
ansible-hardening-linux/
├── ansible.cfg
├── site.yml
├── inventory/hosts.ini
├── group_vars/all.yml
├── host_vars/{srv-zabbix,srv-docker}.yml
├── roles/
│   ├── common/        → system update, base packages, timezone
│   ├── users/          → ansible_adm user, SSH key, sudo
│   ├── ssh_hardening/  → custom SSH port, no root, no password
│   ├── firewall/       → UFW with per-host policies
│   └── fail2ban/       → SSH brute-force protection
└── screenshots/
```

> `group_vars/` and `host_vars/` sit next to `site.yml` at the root, not in a subfolder — see [point 5](#5-variables-werent-loading-inside-the-playbook) in problems found.

---

## How it was built, step by step

### 1. Setting up the control node

`SRV-ANSIBLE-01` with two network interfaces: NAT (access from the physical machine) and LAN Segment 1 (to reach the managed servers).

![Dual network interface on SRV-ANSIBLE-01](screenshots/01-srv-ansible-ip-a-dual-nic.png)

First connectivity check against both managed nodes:

```bash
ansible -i inventory/hosts.ini linux_servers -m ping
```

![Successful ping against both servers](screenshots/02-ansible-ping-linux-servers.png)

### 2. Project structure

Roles following the standard Ansible convention (`defaults/`, `tasks/`, `handlers/`, `templates/`, `files/`):

![Project directory tree](screenshots/03-tree-estructura-proyecto-1.png)
![Project directory tree, continued](screenshots/04-tree-estructura-proyecto-2.png)

### 3. `common` role

Updates packages, installs base tools (curl, vim, htop, net-tools, unzip), and configures the timezone.

![First run of the common role](screenshots/05-playbook-common-primera-ejecucion.png)

### 4. `users` role

Creates `ansible_adm`: dedicated Ansible user, SSH key-only authentication (ED25519), passwordless sudo.

```bash
ssh -i ~/.ssh/ansible_adm_id_ed25519 ansible_adm@<host> "whoami && sudo whoami"
```

![users role deploying ansible_adm on both hosts](screenshots/06-playbook-users-ansible-adm-creado.png)

### 5. `ssh_hardening` role

SSH port changed to 44331, `PermitRootLogin no`, `PasswordAuthentication no`. This was the stage with the most real problems — see the section below.

![Diagnosing the become issue during SSH hardening](screenshots/08-playbook-ssh-hardening-becomepass.png)

### 6. `firewall` role

Before writing any rule, the actually-used ports were verified with `ss -tlnp` — no rules based on assumptions:

![Real ports in use on SRV-ZABBIX](screenshots/09-ss-tlnp-puertos-srv-zabbix.png)

For SRV-DOCKER, ports were confirmed against its real `docker-compose.yml`:

![Reference docker-compose.yml on SRV-DOCKER](screenshots/12-docker-compose-srv-docker-referencia.png)

Active rules after applying the role:

![UFW status on SRV-DOCKER](screenshots/10-ufw-status-srv-docker.png)

Play Recap after `ssh_hardening` + `firewall`:

![Play Recap of ssh_hardening and firewall](screenshots/07-playbook-ssh-hardening-y-firewall-recap.png)

### 7. `fail2ban` role

SSH jail pointing at port 44331 (not 22, which fail2ban assumes by default):

![fail2ban-client status sshd on both hosts](screenshots/11-fail2ban-status-sshd-ambos-hosts.png)

---

## How to run it

```bash
git clone https://github.com/danieloio/ansible-hardening-linux.git
cd ansible-hardening-linux
ansible all -m ping
ansible-playbook site.yml
ansible-playbook site.yml --limit srv-zabbix   # against a single host
```

Adapt `inventory/hosts.ini`, `group_vars/all.yml`, and `host_vars/*.yml` to your own infrastructure.

## Problems found

**1. Inherited NOPASSWD in SRV-ZABBIX's sudoers.** The base image (LinuxVMImages.com) shipped with `%sudo ALL=(ALL:ALL) NOPASSWD:ALL`. Fixed with `visudo`.

**2. `become` timeout due to latency.** After fixing point 1, it still failed with `Timeout (12s) waiting for privilege escalation prompt`. Timing an equivalent SSH+sudo connection with `time` showed the session took ~15s — above the 12s default timeout. Fix: `become_timeout = 30` in `ansible.cfg`.

**3. Duplicate SSH port.** After changing the port via an override in `sshd_config.d/`, the server ended up listening on both 22 *and* 44331. OpenSSH treats multiple `Port` directives as additive, not overriding. Fix: a `lineinfile` task that neutralizes the implicit `Port 22` in the main `sshd_config`.

**4. Systemd *socket activation* ignoring the port.** Even after fixing point 3, SRV-DOCKER still listened only on 22. Cause: `ssh.socket` opens the port itself with a hardcoded `ListenStream=0.0.0.0:22`, before `sshd` even reads its config. Fix: disable `ssh.socket` and let `ssh.service` start independently.

**5. Variables weren't loading inside the playbook.** `firewall_common_allowed_ports` came back `undefined` in the playbook, but resolved fine with `ansible-inventory` and ad-hoc commands. Cause: the playbook lived in `playbooks/site.yml`, while `group_vars/`/`host_vars/` lived at the root — Ansible looks for those folders next to the inventory *and* next to the running playbook. Fix: moved `site.yml` to the repo root.

## Security decisions

- **NOPASSWD for `ansible_adm` + SSH key-only auth (no login password).** Not the same risk as point 1: here the only entry vector is possessing the private key, not a guessable password.
- **SSH port change = noise mitigation, not strong security.** Reduces automated bot scanning, doesn't stop a targeted attack.
- **Firewall rules based on evidence (`ss -tlnp`), not assumptions.** MariaDB and the local-only Zabbix Agent needed no rules since they weren't reachable from the network.

## Possible extensions

- Ansible Vault for sensitive variables
- Restrictive outbound rules in the `firewall` role
- `unattended-upgrades` as a standalone role
- Molecule testing

## Author

**Daniel Moisés Loyo Vásquez** — ASIR Technician · [GitHub](https://github.com/danieloio)
