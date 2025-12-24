# Ansible: Wazuh + Fortinet Syslog via rsyslog

This repository is an **Ansible project** that configures a Linux **Wazuh manager/server** to receive **FortiGate / Fortinet** syslog via **rsyslog**, write it to a configurable file path, and rotate the logs with a configurable retention period (e.g., 14 / 30 / 60 days).

It focuses on **configuration** (rsyslog + Wazuh ingestion). It does **not** install Wazuh itself.

---

## What this does

- Installs and enables **rsyslog** and **logrotate**
- Creates a configurable log directory and log file for Fortinet syslog
- Configures **rsyslog** to listen on:
  - **UDP** (optional)
  - **TCP** (optional)
  - ports are configurable
- Writes received Fortinet syslog to a configurable log file path
- Optionally checks rsyslog service, listener ports, and log path (fail or warn)
- Configures **logrotate** to keep logs for a configurable duration (daily rotation by default)
- Adds a `<localfile>` entry to **Wazuh** so the Wazuh manager ingests the Fortinet log file

---

## Project layout

```
.
├─ ansible.cfg
├─ inventory/
│  ├─ hosts.ini
│  └─ group_vars/
│     └─ wazuh.yml
├─ playbooks/
│  └─ site.yml
└─ roles/
   └─ forti_rsyslog/
      ├─ defaults/main.yml
      ├─ tasks/main.yml
      ├─ handlers/main.yml
      ├─ templates/
      │  ├─ fortinet-rsyslog.conf.j2
      │  └─ fortinet-logrotate.conf.j2
      └─ meta/main.yml
```

---

## Requirements

- Ansible 2.14+ recommended
- A Linux host that is already a **Wazuh manager/server**
- Root privileges (the playbook uses `become: true`)
- Network access from FortiGate to the syslog listener port(s)
- Firewall rules must allow the chosen UDP/TCP port(s) (this role **does not** manage firewalls)

---

## Quick start

1. Put your Wazuh server in `inventory/hosts.ini`
2. Adjust variables in `inventory/group_vars/wazuh.yml`
3. Run:

```bash
ansible-playbook playbooks/site.yml
```

---

## Authentication (no prompts)

To avoid entering passwords interactively, use SSH keys and (optionally) passwordless sudo.

Example (set in `inventory/group_vars/wazuh.yml` or `inventory/hosts.ini`):

```yaml
ansible_user: ansible
ansible_ssh_private_key_file: /root/.ssh/id_ed25519
ansible_become: true
ansible_become_method: sudo
```

If sudo still requires a password, store it with Ansible Vault:

```yaml
ansible_become_pass: "{{ vault_become_pass }}"
```

---

## Configuration variables

All defaults live in `roles/forti_rsyslog/defaults/main.yml`. Override them in your inventory (recommended) or with `-e`.

### Syslog listener

| Variable | Default | Meaning |
|---|---:|---|
| `forti_syslog_enable_udp` | `true` | Enable UDP listener |
| `forti_syslog_enable_tcp` | `false` | Enable TCP listener |
| `forti_syslog_udp_port` | `514` | UDP listen port |
| `forti_syslog_tcp_port` | `5514` | TCP listen port |
| `forti_syslog_listen_address` | `""` | Bind address; empty = all interfaces |
| `forti_syslog_allowed_senders` | `[]` | Optional list of allowed sender IPs (drop others) |

### Log paths (adjustable)

| Variable | Default | Meaning |
|---|---:|---|
| `forti_log_dir` | `/var/log/fortinet` | Directory to store Fortinet syslog file |
| `forti_log_filename` | `fortigate.log` | Log file name |
| `forti_log_path` | `{{ forti_log_dir }}/{{ forti_log_filename }}` | Full log file path |
| `forti_log_owner` | `syslog` | Owner for the Fortinet log path |
| `forti_log_group` | `syslog` | Group for the Fortinet log path |

### Log retention (adjustable)

This project configures **logrotate**.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_logrotate_frequency` | `daily` | Rotation frequency (`daily`, `weekly`, etc.) |
| `forti_log_retention_days` | `14` | Used when frequency is `daily` |
| `forti_logrotate_rotate` | derived | Number of rotated files to keep |

**Examples:**

- Keep **14 days**: `forti_log_retention_days: 14`
- Keep **30 days**: `forti_log_retention_days: 30`
- Keep **60 days**: `forti_log_retention_days: 60`

### Health checks

| Variable | Default | Meaning |
|---|---:|---|
| `forti_rsyslog_healthcheck` | `true` | Run rsyslog health checks after configuration |
| `forti_rsyslog_healthcheck_fail` | `true` | Fail the play on health check issues (set `false` to warn only) |
| `forti_rsyslog_healthcheck_delay` | `2` | Seconds to wait after handler flush before checking listeners |
| `forti_rsyslog_healthcheck_retries` | `5` | Listener check retries (helps avoid transient false negatives) |
| `forti_rsyslog_healthcheck_retry_delay` | `1` | Seconds to wait between listener retries |

---

## Example inventory configuration

Edit `inventory/group_vars/wazuh.yml`:

```yaml
# Listen on UDP only, port 514
forti_syslog_enable_udp: true
forti_syslog_enable_tcp: false
forti_syslog_udp_port: 514

# Store logs somewhere else
forti_log_dir: /data/syslog/fortinet
forti_log_filename: fortigate.log

# Keep 30 days of logs (daily rotation)
forti_log_retention_days: 30

# (Optional) restrict who can send logs
forti_syslog_allowed_senders:
  - "192.0.2.10"
  - "192.0.2.11"

# Wazuh config path/service name (change if your install differs)
forti_wazuh_ossec_conf_path: /var/ossec/etc/ossec.conf
forti_wazuh_manager_service_name: wazuh-manager
```

Run:

```bash
ansible-playbook playbooks/site.yml
```

---

## Notes & troubleshooting

- **Port 514:** This is now the default. If you have conflicts, use a high port like 5514 instead.
- **SELinux:** On SELinux-enforcing systems, you may need additional policy adjustments if you log to a non-standard path.
- **Wazuh config path:** If your `ossec.conf` is not at `/var/ossec/etc/ossec.conf`, set `forti_wazuh_ossec_conf_path` accordingly.
- **FortiGate configuration:** Configure FortiGate to send syslog to this server on the chosen protocol and port.

---

## License

MIT (see `LICENSE`).
