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

## Configuration variables

All defaults live in `roles/forti_rsyslog/defaults/main.yml`. Override them in your inventory (recommended) or with `-e`.

### Syslog listener

| Variable | Default | Meaning |
|---|---:|---|
| `forti_syslog_enable_udp` | `true` | Enable UDP listener |
| `forti_syslog_enable_tcp` | `true` | Enable TCP listener |
| `forti_syslog_udp_port` | `5514` | UDP listen port |
| `forti_syslog_tcp_port` | `5514` | TCP listen port |
| `forti_syslog_listen_address` | `""` | Bind address; empty = all interfaces |
| `forti_syslog_allowed_senders` | `[]` | Optional list of allowed sender IPs (drop others) |

### Log paths (adjustable)

| Variable | Default | Meaning |
|---|---:|---|
| `forti_log_dir` | `/var/log/fortinet` | Directory to store Fortinet syslog file |
| `forti_log_filename` | `fortigate.log` | Log file name |
| `forti_log_path` | `{{ forti_log_dir }}/{{ forti_log_filename }}` | Full log file path |

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

---

## Example inventory configuration

Edit `inventory/group_vars/wazuh.yml`:

```yaml
# Listen on UDP only, port 5514
forti_syslog_enable_udp: true
forti_syslog_enable_tcp: false
forti_syslog_udp_port: 5514

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

- **Port 514:** Using 514 is possible, but 5514 is the default to avoid conflicts with other syslog receivers.
- **SELinux:** On SELinux-enforcing systems, you may need additional policy adjustments if you log to a non-standard path.
- **Wazuh config path:** If your `ossec.conf` is not at `/var/ossec/etc/ossec.conf`, set `forti_wazuh_ossec_conf_path` accordingly.
- **FortiGate configuration:** Configure FortiGate to send syslog to this server on the chosen protocol and port.

---

## License

MIT (see `LICENSE`).
