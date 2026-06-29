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

### Automated (Ansible Playbook — `playbooks/server_hardening.yml`)

The current playbook automates the following controls on **Debian/Ubuntu** targets:

| Section | What It Does |
|---|---|
| **SSH Group** | Creates `sshusers` group; adds specified users |
| **SSH Hardening** | Backs up and hardens `sshd_config` — disables root login, password auth, X11/TCP forwarding, enforces session limits and `AllowGroups` |
| **Diffie-Hellman Cleanup** | Removes DH moduli shorter than 3072 bits from `/etc/ssh/moduli` |
| **sudo Restriction** | Creates `sudousers` group; restricts `/etc/sudoers` to that group |
| **su Restriction** | Creates `suusers` group; restricts `/bin/su` via `dpkg-statoverride` |
| **NTP Verification** | Checks `timedatectl` status and reports sync state |
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

## Known Limitations and Planned Fixes

The following issues are known in the current `server_hardening.yml` playbook and are tracked for resolution:

- **`sshd_config` hardening uses `blockinfile`** — this does not remove pre-existing conflicting settings. A full file template or per-setting `lineinfile` approach is planned to prevent silent conflicts.
- **Missing cipher/MAC/KexAlgorithm hardening** — Mozilla's recommended OpenSSH cipher suite is not yet applied by the playbook. Manual configuration is required for now; see the [original guide's SSH section](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server#secure-etcsshsshd_config).
- **`unattended-upgrades` origins are Ubuntu-specific** — the `Origins-Pattern` block in the playbook uses Ubuntu labels and will not correctly match security updates on Debian. Conditional logic per `ansible_distribution` is planned.
- **NTP is verified but not configured** — the playbook checks `timedatectl` status but does not enable or configure `systemd-timesyncd`. A proper NTP configuration task is planned.
- **`dpkg-statoverride` exit code 2 is tolerated** — exit code 2 is a genuine argument error and should not be suppressed. This will be tightened to only tolerate exit code 1 (entry already exists).

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

3. Run the hardening playbook, passing your user lists:
   ```bash
   ansible-playbook playbooks/server_hardening.yml -i inventory.ini \
     -e "ssh_users=['ubuntu'] sudo_users=['ubuntu'] su_users=['ubuntu']"
   ```

4. After hardening, run a Lynis audit to score your configuration:
   ```bash
   sudo lynis audit system
   ```

> **Warning**: The playbook sets `PasswordAuthentication no` in `sshd_config`. Ensure your SSH public key is already on the target server before running, or you will lose access.

---

## Repository Structure

```
linux-server-hardening-playbook/
├── CHECKLIST.md                  # Step-by-step hardening checklist
├── docs/
│   ├── ssh.md                    # SSH hardening reference
│   ├── firewall.md               # UFW + PSAD + Fail2Ban reference
│   ├── users.md                  # User/privilege control reference
│   ├── auditing.md               # Monitoring and auditing tools
│   └── kernel.md                 # Sysctl and kernel hardening
├── playbooks/
│   └── server_hardening.yml      # Main Ansible hardening playbook
├── scripts/                      # Standalone shell scripts
└── README.md
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
