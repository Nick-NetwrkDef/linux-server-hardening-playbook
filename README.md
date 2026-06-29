# linux-server-hardening-playbook

> A practical, opinionated hardening reference and automation layer for Linux servers — built on the shoulders of [imthenachoman/How-To-Secure-A-Linux-Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server).

---

## What This Is

This repository is a structured, practitioner-focused hardening resource for Linux servers. It consolidates security best practices, configuration references, and Ansible automation into a single place — making it easier to go from a fresh Linux install to a hardened, production-ready server without hunting across dozens of sources.

It is intended for system administrators, MSPs, homelab operators, and security-minded developers who want a clear, repeatable process for securing Linux infrastructure.

---

## Origins

This project is directly inspired by and derived from **[How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server)** by [@imthenachoman](https://github.com/imthenachoman) — one of the most comprehensive, community-maintained Linux hardening guides available. That guide covers an enormous range of topics, from SSH hardening and firewall configuration to intrusion detection, 2FA/MFA, and kernel-level controls.

The original guide is written as a narrative how-to, walking readers through each step manually. This repository builds on that foundation with the following goals:

- Organize the material into a format suitable for MSP/MSSP workflows and repeatable deployments
- Automate hardening steps via Ansible playbooks where manual steps are tedious or error-prone
- Adapt recommendations for specific use-cases (professional services firms, cloud VMs, remote-access servers)
- Track and incorporate upstream updates and community contributions over time

Full credit and deep gratitude go to the original author and contributors of the source guide. This project would not exist without their work.

---

## What It Covers

### Automated (Ansible Playbook — `server_hardening.yml`)

The current playbook automates the following controls on **Debian/Ubuntu** targets:

| Section | What It Does |
|---|---|
| **SSH Group** | Creates `sshusers` group; adds specified users |
| **SSH Hardening** | Backs up `sshd_config`, then renders a hardened config from a template — disables root login, password auth, X11/TCP forwarding, applies Mozilla "modern" ciphers/MACs/KEX, and enforces session limits and `AllowGroups` |
| **Diffie-Hellman Cleanup** | Removes DH moduli shorter than 3072 bits from `/etc/ssh/moduli` |
| **sudo Restriction** | Creates `sudousers` group; grants password-required sudo (not `NOPASSWD`) via `/etc/sudoers` |
| **su Restriction** | Creates `suusers` group; restricts `/bin/su` via `dpkg-statoverride` |
| **NTP Sync** | Enables NTP via `timedatectl set-ntp true` and configures `systemd-timesyncd` |
| **Automatic Updates** | Installs and configures `unattended-upgrades`, `apt-listchanges`, and `apticron` for automated security patching and email alerts |

### Reference Documentation (Planned — `/docs`)

The following areas are covered in reference docs and/or planned for future playbook automation:

**SSH Hardening**
- Ed25519 public/private key setup
- Modern cipher suite, MACs, and KexAlgorithms (Mozilla OpenSSH guidelines)
- Optional 2FA/MFA via `libpam-google-authenticator`

**User and Privilege Controls**
- Strong password enforcement via `libpam-pwquality`
- Application sandboxing with FireJail

**Networking and Firewall**
- UFW with default-deny posture
- PSAD for iptables-level intrusion detection
- Fail2Ban and CrowdSec for application-level intrusion prevention

**System Hardening**
- `/proc` hardening with `hidepid=2`
- NTP configuration via `systemd-timesyncd` (Debian 13+) or `ntp` package
- Kernel sysctl hardening

**Auditing and Monitoring**
- File integrity monitoring with AIDE
- Anti-virus scanning with ClamAV
- Rootkit detection with rkhunter and chkrootkit
- Log analysis with logwatch
- Full security audits with Lynis
- Host intrusion detection with OSSEC

**Email Alerting**
- MSMTP and Exim4 configuration for system alert delivery via Gmail/SMTP

---

## Who This Is For

- **MSPs and MSSPs** deploying servers for clients and needing a repeatable hardening baseline
- **Homelabbers** who want to run a secure, internet-facing server at home
- **Developers and startups** standing up cloud VMs (AWS, DigitalOcean, Hetzner, etc.) and wanting more than default OS security
- **Security practitioners** looking for a readable, auditable reference to share with teams

---

## Relationship to the Original Guide

| Aspect | [Original Guide](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) | This Repository |
|---|---|---|
| Format | Narrative how-to | Reference + Ansible automation |
| Audience | Anyone learning Linux security | Practitioners with working Linux knowledge |
| Automation | Manual steps with copy-paste snippets | Ansible playbooks for repeatable deployment |
| Scope | Distribution-agnostic | Debian/Ubuntu (Ansible); distribution-agnostic (docs) |
| Maintenance | Community-maintained | Actively maintained for MSP/production use-cases |

This repository does **not** replace the original guide. If you are new to Linux security, start there first. Read through it fully before using anything here.

---

## Getting Started

> **Important**: Read the full original guide at [imthenachoman/How-To-Secure-A-Linux-Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) before applying anything here. Understand what each step does and how it affects your specific setup.

### Prerequisites

- Ansible installed on your control machine
- SSH access to the target server with a user that has `sudo` privileges
- SSH public key already deployed to the target (the playbook sets `PasswordAuthentication no`)
- Run from the repository root so the playbook can find `templates/sshd_config.j2` (the SSH config is rendered from this template)

### Configuring users

The playbook manages three groups and adds the users you specify to each:

| Variable | Group created | Purpose |
|---|---|---|
| `ssh_users` | `sshusers` | Allowed to log in over SSH (`AllowGroups sshusers`) |
| `sudo_users` | `sudousers` | Granted password-required `sudo` |
| `su_users` | `suusers` | Allowed to run `/bin/su` |

Set them in the `vars:` block at the top of `server_hardening.yml`, or pass them on the command line with `-e`. Both a **comma-separated string** and a **JSON list** are accepted — the playbook normalizes either into a list:

```bash
# comma-separated (simplest)
-e "ssh_users=alice,bob sudo_users=alice su_users=alice"

# JSON list (use single quotes around the whole object)
-e '{"ssh_users": ["alice","bob"], "sudo_users": ["alice"], "su_users": ["alice"]}'
```

> Note: `-e "ssh_users=[...]"` (a bare list literal in `key=value` form) does **not** work — Ansible treats it as a plain string. Use one of the two forms above.

### Running the Playbook

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/linux-server-hardening-playbook.git
   cd linux-server-hardening-playbook
   ```

2. Edit your inventory file or pass the target host inline:
   ```bash
   # inventory.ini example
   [servers]
   192.168.1.100 ansible_user=ubuntu
   ```

3. (Recommended) Dry-run first to preview changes without applying them:
   ```bash
   ansible-playbook server_hardening.yml -i inventory.ini \
     -e "ssh_users=ubuntu sudo_users=ubuntu su_users=ubuntu" --check
   ```

4. Run the hardening playbook, passing your user lists:
   ```bash
   ansible-playbook server_hardening.yml -i inventory.ini \
     -e "ssh_users=ubuntu sudo_users=ubuntu su_users=ubuntu"
   ```

5. After hardening, run a Lynis audit to score your configuration:
   ```bash
   sudo lynis audit system
   ```

The play is idempotent: SSH is only restarted when `sshd_config` actually changes (via a handler), and re-running it produces no further changes once the host is hardened. NTP sync (`systemd-timesyncd`) is configured against `pool.ntp.org`.

> **Warning**: The playbook sets `PasswordAuthentication no` in `sshd_config`. Ensure your SSH public key is already on the target server before running, or you will lose access. As a safety net, the previous `sshd_config` is backed up to `/etc/ssh/sshd_config-COPY-<timestamp>` and the new config is validated with `sshd -t` before it is applied.

---

## Repository Structure

Current layout:

```
linux-server-hardening-playbook/
├── server_hardening.yml          # Main Ansible hardening playbook
├── templates/
│   └── sshd_config.j2            # Hardened sshd_config rendered by the playbook
├── inventory.ini                 # Example inventory
├── LICENSE
└── README.md
```

Planned additions (not yet present in the repo):

```
├── CHECKLIST.md                  # Step-by-step hardening checklist
├── docs/                         # SSH / firewall / users / auditing / kernel references
└── scripts/                      # Standalone shell scripts
```

---

## Contributing

Contributions are welcome. If you find an error, have a suggestion, or want to add support for another distribution, please open an issue or submit a pull request.

When contributing, please:
- Reference the relevant section of the original guide where applicable
- Test changes on a fresh VM before submitting
- Keep all playbook tasks idempotent

---

## License

This project is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), consistent with the license of the original guide it is based on.

Original guide: © [imthenachoman](https://github.com/imthenachoman) — [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server), licensed under CC BY-SA 4.0.

---

## Acknowledgments

- **[@imthenachoman](https://github.com/imthenachoman)** — for writing and maintaining the foundational guide this project is built on
- **[@moltenbit](https://github.com/moltenbit)** — for the companion [Ansible playbooks](https://github.com/moltenbit/How-To-Secure-A-Linux-Server-With-Ansible)
- The broader open-source security community for the tools referenced throughout: UFW, Fail2Ban, CrowdSec, PSAD, Lynis, AIDE, rkhunter, ClamAV, OSSEC, and more
