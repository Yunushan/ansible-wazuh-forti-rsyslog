# Ansible: Wazuh + Fortinet Syslog via rsyslog

This repository is an **Ansible project** that configures a Linux **Wazuh manager/server** to receive **FortiGate / Fortinet** syslog via **rsyslog**, write it to a configurable file path, and rotate the logs with a configurable retention period (e.g., 1 / 7 / 14 days).

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
- Adds local Wazuh rules so high/critical Fortinet events appear in alerts (configurable)

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

If the playbook warns that the Wazuh service was not detected, restart it manually:

```bash
systemctl restart wazuh-manager
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
| `forti_log_group` | `wazuh` | Group for the Fortinet log path |
| `forti_log_file_mode` | `0644` | File mode for the Fortinet log file |

### Log retention (adjustable)

This project configures **logrotate**.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_logrotate_frequency` | `daily` | Rotation frequency (`daily`, `weekly`, etc.) |
| `forti_log_retention_days` | `1` | Used when frequency is `daily` |
| `forti_logrotate_rotate` | derived | Number of rotated files to keep |
| `forti_logrotate_copytruncate` | `true` | Use copytruncate (set `false` to avoid losing backlog on rotation) |

**Examples:**

- Keep **1 day**: `forti_log_retention_days: 1`
- Keep **7 days**: `forti_log_retention_days: 7`
- Keep **14 days**: `forti_log_retention_days: 14`

Wazuh archives rotation (separate retention):

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_archives_logrotate_manage` | `true` | Manage logrotate for `/var/ossec/logs/archives/archives.json` |
| `forti_wazuh_archives_logrotate_conf_path` | `/etc/logrotate.d/wazuh-archives` | Logrotate config path |
| `forti_wazuh_archives_log_path` | `/var/ossec/logs/archives/archives.json` | Archives file to rotate |
| `forti_wazuh_archives_logrotate_frequency` | `{{ forti_logrotate_frequency }}` | Rotation frequency |
| `forti_wazuh_archives_log_retention_days` | `7` | Used when frequency is `daily` |
| `forti_wazuh_archives_logrotate_rotate` | derived | Number of rotated files to keep |

**Note:** If your Wazuh package already ships `/etc/logrotate.d/wazuh` that manages archives, set `forti_wazuh_archives_logrotate_manage: false` to avoid duplicate logrotate entries.

### Performance tuning / ingestion strategy

| Variable | Default | Meaning |
|---|---:|---|
| `forti_tuning_mode` | `"2"` | `1=source_filter` (drop traffic logs), `2=split_traffic` (separate traffic file + optional Filebeat), `3=wazuh_light` (disable archives), `auto=detect` |
| `forti_syslog_traffic_patterns` | `["type=traffic"]` | Patterns used to match high-volume traffic logs |
| `forti_traffic_log_filename` | `fortigate-traffic.log` | Traffic-only log file name (mode 2) |
| `forti_traffic_log_path` | `{{ forti_log_dir }}/{{ forti_traffic_log_filename }}` | Full traffic log path (mode 2) |
| `forti_traffic_filebeat_manage` | `null` | Auto-enable Filebeat input for traffic logs in mode 2 (set `true`/`false` to override) |
| `forti_traffic_filebeat_enabled` | `true` | Enable traffic input in Filebeat |
| `forti_traffic_filebeat_inputs_file` | `/etc/filebeat/inputs.d/fortinet-traffic.yml` | Traffic input file path |
| `forti_traffic_filebeat_inputs_reload` | `false` | Reload inputs dynamically |
| `forti_traffic_filebeat_restart` | `true` | Restart Filebeat after changes |

**Notes:**
- Mode 1 drops traffic logs at rsyslog. Wazuh never sees those events.
- Mode 2 routes traffic logs to a separate file and (optionally) Filebeat. Wazuh sees only non-traffic logs.
- Mode 3 sets `<logall>` and `<logall_json>` to `no`, which disables archives (alerts still work, archives index will be empty).
- Auto mode picks `split_traffic` when `forti_traffic_filebeat_manage` is set, `wazuh_light` when both `forti_wazuh_logall` and `forti_wazuh_logall_json` are `no`, otherwise `source_filter`.

### Backfill (optional)

Use this to re-ingest a missing time window from a rotated Fortinet log.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_backfill_enable` | `null` | Auto-enable when all backfill values are set (set `true`/`false` to override) |
| `forti_backfill_source_path` | `""` | Source log (e.g., `/var/log/fortinet/fortigate.log-20260108`) |
| `forti_backfill_output_path` | `""` | Output backfill file path |
| `forti_backfill_start` | `""` | Start timestamp (`YYYY-MM-DD HH:MM:SS`, local time in log) |
| `forti_backfill_end` | `""` | End timestamp (exclusive) |
| `forti_backfill_force` | `false` | Overwrite backfill file if it already exists |
| `forti_backfill_manage_localfile` | `true` | Add localfile entry so Wazuh ingests the backfill |
| `forti_backfill_cleanup_localfile` | `false` | Remove the backfill localfile entry |
| `forti_backfill_cleanup_file` | `false` | Remove the backfill file |

**Note:** Backfill auto-runs when all values are set. Set `forti_backfill_enable: false` to suppress. Backfill is not idempotent for Wazuh; re-running with the same file can duplicate events. Enable it once, verify ingest, then set cleanup flags or remove the backfill config.

### Health checks

| Variable | Default | Meaning |
|---|---:|---|
| `forti_rsyslog_healthcheck` | `true` | Run rsyslog health checks after configuration |
| `forti_rsyslog_healthcheck_fail` | `true` | Fail the play on health check issues (set `false` to warn only) |
| `forti_rsyslog_healthcheck_delay` | `2` | Seconds to wait after handler flush before checking listeners |
| `forti_rsyslog_healthcheck_retries` | `5` | Listener check retries (helps avoid transient false negatives) |
| `forti_rsyslog_healthcheck_retry_delay` | `1` | Seconds to wait between listener retries |

### Wazuh integration

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_manage_logall` | `true` | Manage `<logall>` in `ossec.conf` |
| `forti_wazuh_logall` | `yes` | Value for `<logall>` |
| `forti_wazuh_manage_logall_json` | `true` | Manage `<logall_json>` in `ossec.conf` |
| `forti_wazuh_logall_json` | `yes` | Value for `<logall_json>` |

### Wazuh manager tuning (optional)

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_tuning_manage` | `null` | Auto-enable when tuning lists are set (set `true`/`false` to override) |
| `forti_wazuh_disable_modules` | `[]` | Module/wodle names to disable (e.g., `vulnerability-detector`, `syscollector`) |
| `forti_wazuh_ruleset_manage` | `null` | Auto-enable when ruleset lists are set (set `true`/`false` to override) |
| `forti_wazuh_ruleset_rule_excludes` | `[]` | Rule IDs to exclude from evaluation |
| `forti_wazuh_ruleset_decoder_excludes` | `[]` | Decoder names to exclude |
| `forti_wazuh_ruleset_exclude_forti` | `true` | Auto-exclude Fortinet rules files to suppress the full Forti ruleset alerts |
| `forti_wazuh_ruleset_exclude_forti_patterns` | `["*fortigate*","*fortinet*"]` | Filename patterns used to find Fortinet rules |
| `forti_wazuh_ruleset_dirs` | `[/var/ossec/ruleset/rules, /var/ossec/etc/rules]` | Ruleset directories to scan for auto excludes |
| `forti_wazuh_root` | `/var/ossec` | Wazuh root used to build relative exclude paths |
| `forti_wazuh_fortinet_alerts_manage` | `null` | Auto-enable high/critical Fortinet alerts when Fortinet rules are excluded (set `true`/`false` to override) |
| `forti_wazuh_fortinet_alerts_use_fortinet_level` | `true` | Use Fortinet `level` instead of `crlevel` and map to Wazuh levels |
| `forti_wazuh_fortinet_alerts_level_map` | defined | Mapping for Fortinet `level` strings (see defaults) |
| `forti_wazuh_fortinet_alerts_levels` | `[12,13,14,15]` | Fortinet `level=` values that should generate Wazuh alerts |
| `forti_wazuh_fortinet_alerts_map` | `[{match: high, level: 12}, {match: critical, level: 15}]` | Mapping for string severities (overrides levels list when non-empty) |
| `forti_wazuh_fortinet_alerts_match_field` | `crlevel` | Fortinet field name to match (e.g., `level`, `severity`) |
| `forti_wazuh_fortinet_alerts_rules_path` | `/var/ossec/etc/rules/forti-alerts.xml` | Local rules file path for Fortinet alerts |
| `forti_wazuh_fortinet_alerts_rule_id_base` | `100300` | Starting rule ID for generated Fortinet alert rules |

**Note:** By default, Fortinet rules are auto-excluded to keep noise down, but the role adds local rules that map Fortinet `level` strings into Wazuh alerts. Set `forti_wazuh_fortinet_alerts_use_fortinet_level: false` to map `crlevel=high` and `crlevel=critical` into Wazuh levels 12/15 instead. Adjust `forti_wazuh_fortinet_alerts_map` or `forti_wazuh_fortinet_alerts_level_map` to change the mapping, or set `forti_wazuh_fortinet_alerts_manage: false` to disable. Set `forti_wazuh_ruleset_exclude_forti: false` to use the full Fortinet ruleset instead. Disabling modules and excluding rules can reduce CPU/RAM usage, but may reduce visibility. Apply conservatively.

Example for Fortinet logs that use numeric `level=12-15` instead of `crlevel` strings:

```yaml
forti_wazuh_fortinet_alerts_match_field: level
forti_wazuh_fortinet_alerts_map: []
forti_wazuh_fortinet_alerts_levels: [12,13,14,15]
```

Example for Fortinet logs that use syslog severity strings (e.g., `level=notice`, `level=error`):

```yaml
forti_wazuh_fortinet_alerts_use_fortinet_level: true
# Optional: override defaults in forti_wazuh_fortinet_alerts_level_map
```

Example for Fortinet logs where you want `crlevel=high/critical` to drive alerts instead:

```yaml
forti_wazuh_fortinet_alerts_use_fortinet_level: false
forti_wazuh_fortinet_alerts_match_field: crlevel
forti_wazuh_fortinet_alerts_map:
  - match: high
    level: 12
  - match: critical
    level: 15
```

### Email alerts (SMTP via custom integration)

Wazuh's built-in email uses only `smtp_server` without auth. To support SMTP auth/ports/TLS, this role can install a **custom integration** that sends email via Python.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_email_integration_manage` | `false` | Enable the custom email integration |
| `forti_wazuh_email_integration_name` | `custom-forti-email` | Integration name (must start with `custom-`) |
| `forti_wazuh_email_integration_alert_levels` | `[12,15]` | Exact Wazuh rule levels that send email |
| `forti_wazuh_email_integration_level` | `null` | Minimum level filter (null = min of list) |
| `forti_wazuh_email_integration_timeout` | `10` | Integration timeout (seconds) |
| `forti_wazuh_email_integration_retries` | `0` | Integration retries |
| `forti_wazuh_email_subject_prefix` | `[Wazuh]` | Email subject prefix |
| `forti_wazuh_email_from` | `""` | From address |
| `forti_wazuh_email_to` | `[]` | Recipient list |
| `forti_wazuh_email_smtp_server` | `""` | SMTP server |
| `forti_wazuh_email_smtp_port` | `587` | SMTP port |
| `forti_wazuh_email_smtp_user` | `""` | SMTP username (optional) |
| `forti_wazuh_email_smtp_password` | `""` | SMTP password (optional) |
| `forti_wazuh_email_smtp_protocol` | `starttls` | `starttls`, `ssl`, or `none` |

Example:

```yaml
forti_wazuh_email_integration_manage: true
forti_wazuh_email_from: wazuh@example.com
forti_wazuh_email_to:
  - soc@example.com
forti_wazuh_email_smtp_server: smtp.example.com
forti_wazuh_email_smtp_port: 587
forti_wazuh_email_smtp_user: wazuh@example.com
forti_wazuh_email_smtp_password: "{{ vault_smtp_password }}"
forti_wazuh_email_smtp_protocol: starttls
forti_wazuh_email_integration_alert_levels: [12, 15]
```

**Note:** Email is triggered by **alerts**, not archives. Use `forti_wazuh_archives_severity_mode: keep` if you want rule levels retained in archives.
Requires `python3` on the Wazuh manager host.

### Filebeat archives shipping (optional)

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_filebeat_manage` | `false` | Manage Filebeat Wazuh module archives settings |
| `forti_wazuh_filebeat_module_path` | `/etc/filebeat/modules.d/wazuh.yml` | Path to Filebeat Wazuh module config |
| `forti_wazuh_filebeat_archives_enabled` | `false` | Enable archives shipping in Filebeat |
| `forti_wazuh_filebeat_archives_paths` | `[/var/ossec/logs/archives/archives.json]` | Paths for archives in Filebeat |
| `forti_wazuh_archives_clear_severity` | `true` | Drop `rule.level` from archives events (set `false` to keep severity) |
| `forti_wazuh_archives_severity_mode` | `"drop"` | Override severity handling for archives: `drop`, `zero`, `keep` |
| `forti_wazuh_archives_include_severity` | `null` | Legacy override: `true` keeps `rule.level`, `false` drops it |
| `forti_wazuh_archives_pipeline_manage` | `false` | Manage OpenSearch ingest pipeline to enforce archives severity handling |
| `forti_wazuh_opensearch_url` | `""` | OpenSearch URL (e.g., `https://127.0.0.1:9200`) |
| `forti_wazuh_opensearch_auth_method` | `basic` | Auth method: `basic`, `token`, or `none` |
| `forti_wazuh_opensearch_username` | `""` | OpenSearch username (basic auth) |
| `forti_wazuh_opensearch_password` | `""` | OpenSearch password (basic auth) |
| `forti_wazuh_opensearch_token` | `""` | OpenSearch bearer token (token auth) |
| `forti_wazuh_opensearch_validate_certs` | `true` | Validate HTTPS certificates |
| `forti_wazuh_archives_pipeline_name` | `""` | Ingest pipeline name (auto-detect when empty) |
| `forti_wazuh_archives_pipeline_match` | `wazuh-archives` | Regex used for auto-detect when name is empty |
| `forti_wazuh_filebeat_config_path` | `/etc/filebeat/filebeat.yml` | Filebeat main config path (used for inputs fallback) |
| `forti_wazuh_filebeat_inputs_manage` | `true` | Manage inputs fallback when module config is missing |
| `forti_wazuh_filebeat_inputs_dir` | `/etc/filebeat/inputs.d` | Directory for Filebeat inputs |
| `forti_wazuh_filebeat_inputs_file` | `/etc/filebeat/inputs.d/wazuh-archives.yml` | Archives input file path |
| `forti_wazuh_filebeat_inputs_glob` | `/etc/filebeat/inputs.d/*.yml` | Inputs glob for filebeat config |
| `forti_wazuh_filebeat_inputs_reload` | `false` | Reload inputs dynamically |
| `forti_wazuh_filebeat_input_type` | `log` | Input type used for archives |
| `forti_wazuh_filebeat_restart` | `true` | Restart Filebeat after changes |
| `forti_filebeat_service_name` | `filebeat` | Filebeat service name |

**Note:** If the Wazuh Filebeat module file is missing, the role falls back to creating an input file under `inputs.d` and enables `filebeat.config.inputs` in `filebeat.yml`. If the Wazuh module is defined inline in `filebeat.yml`, the role removes the archives input file to avoid duplicate indexing. By default, `forti_wazuh_archives_clear_severity: true` drops `rule.level` from archives events; use `forti_wazuh_archives_severity_mode` to `keep` or `zero` it.  
Wazuh's **archives ingest pipeline** can re-create `rule.level` after Filebeat processors run. To enforce the chosen severity behavior in OpenSearch, enable `forti_wazuh_archives_pipeline_manage` and provide OpenSearch credentials.

### Filebeat performance tuning (optional)

Use this to reduce archives lag by increasing Filebeat batching and queue size.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_filebeat_tuning_manage` | `false` | Enable Filebeat tuning block in `filebeat.yml` |
| `forti_wazuh_filebeat_tuning_output_worker` | `2` | Output workers for Elasticsearch |
| `forti_wazuh_filebeat_tuning_output_bulk_max_size` | `2048` | Bulk batch size for Elasticsearch output |
| `forti_wazuh_filebeat_tuning_output_compression_level` | `0` | Compression level (0 = off, less CPU) |
| `forti_wazuh_filebeat_tuning_queue_events` | `4096` | In-memory queue size (events) |
| `forti_wazuh_filebeat_tuning_queue_flush_min_events` | `2048` | Queue flush threshold |
| `forti_wazuh_filebeat_tuning_queue_flush_timeout` | `"1s"` | Queue flush timeout |

**Note:** These settings trade CPU and memory for throughput. Increase gradually and monitor Filebeat CPU/RAM.

### Dashboards index pattern (optional)

If you want the role to create the **Wazuh archives** index pattern in Wazuh/OpenSearch Dashboards, enable this block.

| Variable | Default | Meaning |
|---|---:|---|
| `forti_wazuh_dashboards_manage` | `false` | Create the archives index pattern via Dashboards API |
| `forti_wazuh_dashboards_url` | `""` | Base URL for Dashboards (e.g., `https://wazuh.example.com`) |
| `forti_wazuh_dashboards_auth_method` | `basic` | Auth method: `basic`, `token`, or `none` |
| `forti_wazuh_dashboards_username` | `""` | Dashboards username (basic auth) |
| `forti_wazuh_dashboards_password` | `""` | Dashboards password (basic auth) |
| `forti_wazuh_dashboards_token` | `""` | Bearer token (token auth) |
| `forti_wazuh_dashboards_version` | `2.x` | Dashboards version (used to select XSRF header) |
| `forti_wazuh_dashboards_validate_certs` | `true` | Validate HTTPS certificates |
| `forti_wazuh_dashboards_timezone` | `Etc/GMT-3` | Dashboards timezone (affects archives display) |
| `forti_wazuh_dashboards_tenant` | `""` | Dashboards tenant (e.g., `global` or `private`) |
| `forti_wazuh_dashboards_archives_pattern` | `wazuh-archives-*` | Index pattern title |
| `forti_wazuh_dashboards_archives_pattern_id` | `wazuh-archives-*` | Saved object ID |
| `forti_wazuh_dashboards_archives_time_field` | `timestamp` | Time field for the pattern |

**Note:** This creates the index pattern only. You still need Filebeat archives shipping enabled for data to appear. Archives are stored in UTC; the timezone setting controls how Dashboards displays them (e.g., `Etc/GMT-3` for GMT+3).

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
forti_logrotate_copytruncate: false

# (Optional) restrict who can send logs
forti_syslog_allowed_senders:
  - "192.0.2.10"
  - "192.0.2.11"

# (Optional) tune ingestion strategy (default is "2")
# forti_tuning_mode: "2"  # 1=source_filter, 2=split_traffic, 3=wazuh_light, auto=detect
# forti_syslog_traffic_patterns:
#   - "type=traffic"
# forti_traffic_log_filename: fortigate-traffic.log
# forti_traffic_filebeat_manage: null

# (Optional) backfill a missing window from a rotated log
# forti_backfill_enable: true  # optional; auto if all values are set
# forti_backfill_source_path: /var/log/fortinet/fortigate.log-20260108
# forti_backfill_output_path: /var/log/fortinet/fortigate.backfill-20260107-1500_20260108-0200.log
# forti_backfill_start: "2026-01-07 15:00:00"
# forti_backfill_end: "2026-01-08 02:00:00"
# forti_backfill_manage_localfile: true
# forti_backfill_cleanup_localfile: false
# forti_backfill_cleanup_file: false

# Wazuh config path/service name (change if your install differs)
forti_wazuh_ossec_conf_path: /var/ossec/etc/ossec.conf
forti_wazuh_manager_service_name: wazuh-manager

# (Optional) create Wazuh archives index pattern in Dashboards
# forti_wazuh_dashboards_manage: true
# forti_wazuh_dashboards_url: https://wazuh.example.com
# forti_wazuh_dashboards_auth_method: basic
# forti_wazuh_dashboards_username: admin
# forti_wazuh_dashboards_password: "{{ vault_dashboards_password }}"
# forti_wazuh_dashboards_version: "2.x"
# forti_wazuh_dashboards_validate_certs: false
# forti_wazuh_dashboards_timezone: Etc/GMT-3
# forti_wazuh_dashboards_tenant: global
# forti_wazuh_dashboards_archives_pattern: wazuh-archives-*
# forti_wazuh_dashboards_archives_time_field: timestamp
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
