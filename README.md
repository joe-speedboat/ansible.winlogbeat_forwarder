# Ansible Role: `joe-speedboat.log_forwarder`

OS-independent Ansible role for forwarding Windows Event Logs and Linux logs to **Graylog**.

| Target OS | Collector | Graylog input | Default port | Log sources |
|---|---|---|---|---|
| Windows Server | Winlogbeat 7.x | Beats | 5044 | Application, System, Security |
| Ubuntu 24/26 | Fluent Bit | GELF TCP | 12201 | journald, auditd, auth/secure, nginx, apache2 |
| Rocky/RHEL/Alma 8–10 | Fluent Bit | GELF TCP | 12201 | journald, auditd, secure, nginx, httpd |

Based on the `joe-speedboat/ansible.template` OS-independent task dispatcher — OS selection via Ansible facts.

---

## Requirements

### Control Node

- Ansible ≥ 2.14
- Windows targets: collection `ansible.windows`
- Linux targets: SSH + root / privilege escalation

### Graylog

Create the matching inputs before deploying the role:

| Log source | Graylog input | Port |
|---|---|---|
| Windows Events | Beats | 5044 |
| Linux Logs | GELF TCP | 12201 |

**Important for the Beats input**: 
  In the Graylog web UI, when creating or editing the Beats input,   
  [X] **Do not add Beats type as prefix**  
  Otherwise Graylog prefixes all field names with `winlogbeat_` (e.g. `winlogbeat_agent_hostname` instead of `agent_hostname`).

### Target Nodes

**Windows**
- Server 2022 / 2025 (tested)
- WinRM configured, outbound access to Graylog port 5044

**Linux**
- systemd-based
- Package manager access (official Fluent Bit repository via `packages.fluentbit.io`)
- Outbound access to Graylog port 12201

---

## Quick Start

```yaml
- name: Forward Windows Event Logs to Graylog
  hosts: windows
  gather_facts: true
  roles:
    - role: joe-speedboat.log_forwarder
      vars:
        winlogbeat_graylog_host: graylog.example.com
        winlogbeat_graylog_port: 5044

- name: Forward Linux logs to Graylog
  hosts: linux
  gather_facts: true
  roles:
    - role: joe-speedboat.log_forwarder
      vars:
        log_forwarder_graylog_gelf_host: graylog.example.com
        log_forwarder_graylog_gelf_port: 12201
```

---

### Windows / Winlogbeat — Graylog fields

`fields_under_root: true` under `output.logstash:` injects custom fields at root level in Graylog (no `winlogbeat_` prefix).

| Field | Value | Description |
|---|---|---|
| `log_type` | `winlogbeat` | Identifies Windows events in Graylog |
| `agent_type` | `{{ winlogbeat_node_name }}` | Collector node name |
| `host` | `{{ winlogbeat_node_name }}` | Hostname in Graylog `source` |
| `host_ip` | `{{ ansible_default_ipv4.address }}` | Host IP |

---

## Variables

The `log_forwarder_` prefix is used OS-wide; `winlogbeat_` variables apply to Windows only.

### Windows / Winlogbeat — Graylog Connection

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_graylog_host` | `graylog.example.com` | Graylog Beats input host |
| `winlogbeat_graylog_port` | `5044` | Graylog Beats input port |
| `winlogbeat_version` | `7.17.28` | Winlogbeat version (Graylog only supports 7.x) |

### Windows / Winlogbeat — Event Logs & Service

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_event_logs` | Application, System, Security | Event log channels (all events) |
| `winlogbeat_node_name` | `{{ inventory_hostname }}` | Collector node identifier |
| `winlogbeat_service_state` | `started` | Desired service state |
| `winlogbeat_service_enabled` | `true` | Auto-start on boot |
| `winlogbeat_retention_versions` | `3` | Old versions retained during upgrades |
| `winlogbeat_tags` | `[]` | Additional tags |
| `winlogbeat_fields` | — | Additional fields (e.g. `env`, `datacenter`) |

### Linux / Fluent Bit — Graylog Connection

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_graylog_gelf_host` | `graylog.example.com` | Graylog GELF TCP host |
| `log_forwarder_graylog_gelf_port` | `12201` | Graylog GELF TCP port |
| `log_forwarder_graylog_gelf_mode` | `tcp` | GELF mode |
| `log_forwarder_fluent_bit_workers` | `2` | Fluent Bit output worker count |

### Linux / Fluent Bit — Uninstall Options

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_remove_repo` | `true` | Remove Fluent Bit repository on uninstall |
| `log_forwarder_audit_remove` | `true` | Remove audit packages on uninstall |

Set to `false` to keep packages/repositories configured after uninstall.

### Linux / Fluent Bit — Log Sources

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_linux_install` | `true` | Enable Linux log forwarding |
| `log_forwarder_journal_tag` | `journald` | Tag for journald input |
| `log_forwarder_read_from_tail` | `true` | Start journald reading from tail |
| `log_forwarder_storage_path` | `/var/lib/fluent-bit` | Fluent Bit state directory |
| `log_forwarder_audit_enabled` | `true` | Enable auditd + audit log forwarding |
| `log_forwarder_audit_tag` | `audit` | Tag for audit input |
| `log_forwarder_security_files_enabled` | `true` | Forward auth.log/secure + web server logs |
| `log_forwarder_security_log_paths` | see below | File paths/globs to tail |

Default `log_forwarder_security_log_paths`:

```yaml
log_forwarder_security_log_paths:
  - /var/log/auth.log
  - /var/log/secure
  - /var/log/nginx/*.log
  - /var/log/apache2/*.log
  - /var/log/httpd/*.log
```

The role also deploys audit rules for package manager execution (key `package_change`) targeting:
- **Ubuntu**: `dpkg`, `apt-get`, `apt-cache`, `/usr/bin/python3`
- **Rocky/RHEL**: `rpm`, `dnf`, `yum`, `/usr/libexec/platform-python`

---

## Task Layout

```
tasks/
├── main.yml              # Template dispatcher
├── include-file.yml      # OS folder selection via facts
├── uninstall.yml         # Windows uninstall
├── Windows/              # Winlogbeat
│   ├── 00_validate.yml
│   ├── 10_install.yml
│   ├── 20_configure.yml
│   └── 30_service.yml
├── Ubuntu/               # Fluent Bit (Debian family)
│   ├── 00_validate.yml
│   ├── 10_install.yml
│   ├── 20_configure.yml
│   └── 30_service.yml
└── rhelAll/              # Fluent Bit (RHEL family)
    ├── 00_validate.yml
    ├── 10_install.yml
    ├── 20_configure.yml
    └── 30_service.yml
```

---

## Winlogbeat Upgrades

Winlogbeat is installed into versioned directories and exposed via a `current` junction:

```yaml
winlogbeat_version: '7.17.29'
```

Upgrade workflow:
1. Stop service → extract ZIP → backup previous installation
2. Repoint `current` junction → deploy config → start service
3. Clean up old versions beyond `winlogbeat_retention_versions`

---

## Uninstall (Windows)

```yaml
- hosts: windows
  gather_facts: true
  tasks:
    - ansible.builtin.include_role:
        name: joe-speedboat.log_forwarder
        tasks_from: uninstall.yml
```

Linux uninstall is not implemented yet; remove Fluent Bit / audit rules via your host lifecycle tooling.

---

## Installation

```bash
git clone https://github.com/joe-speedboat/ansible.log_forwarder.git /etc/ansible/roles/joe-speedboat.log_forwarder
```

Or via Galaxy (once published):

```bash
ansible-galaxy role install joe-speedboat.log_forwarder
```

---

### Runtime Checks (Linux)

```bash
systemctl is-active fluent-bit
systemctl is-enabled fluent-bit
systemctl is-active auditd
auditctl -l | grep package_change
fluent-bit -c /etc/fluent-bit/fluent-bit.conf --dry-run
```

### Runtime Checks (Windows PowerShell)

```powershell
Get-Service winlogbeat
Get-Content 'C:\Program Files\Winlogbeat\current\winlogbeat.yml'
```

### Graylog Search

After enabling **«Do not add Beats type as prefix»** on the Graylog Beats input, all fields arrive without the `winlogbeat_` prefix (e.g. `agent_hostname`, not `winlogbeat_agent_hostname`).

| Source | Query |
|---|---|
| journald | `source:<host> AND log_type:journald` |
| Audit | `source:<host> AND log_type:auditd` |
| auth/secure | `source:<host> AND log_type:security_file` |
| Windows Events | `source:<host> AND winlog_channel:Security AND log_type:winlogbeat` |

---

## Tested Platforms

| Target | Collector | Graylog ingest verified |
|---|---|---|
| Windows Server 2022 | Winlogbeat 7.17 | ✅ |
| Ubuntu 24.04 LTS | Fluent Bit | ✅ |
| Ubuntu 26.04 LTS | Fluent Bit | ✅ |
| Rocky Linux 8.10 | Fluent Bit | ✅ |
| Rocky Linux 9.7 | Fluent Bit | ✅ |
| Rocky Linux 10.1 | Fluent Bit | ✅ |

---

## License

GPLv3
