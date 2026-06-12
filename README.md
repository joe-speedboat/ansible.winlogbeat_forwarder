# ansible.log_forwarder / `joe-speedboat.log_forwarder`

OS-independent Ansible role for forwarding Windows Event Logs and Linux logs to **Graylog**.

| Target OS | Collector | Graylog input | Default port | Log sources |
|---|---|---|---|---|
| Windows Server | Winlogbeat 7.x | Beats | 5044 | Application, System, Security |
| Ubuntu 24/26 | Fluent Bit | GELF TCP | 12201 | journald, auditd, auth.log, nginx, apache2 |
| Rocky/RHEL/Alma 8–10 | Fluent Bit | GELF TCP | 12201 | journald, auditd, secure, nginx, httpd |

Based on the `joe-speedboat/ansible.template` OS-independent task dispatcher — OS selection via Ansible facts.

---

## Requirements

### Control Node

- Ansible ≥ 2.14
- Windows targets: collection `ansible.windows`
- Linux targets: SSH + root / privilege escalation
- Rocky 8 targets: Python 3.9 must be installed (`dnf module enable -y python39 && dnf install -y python39`)

### Graylog

Create the matching inputs before deploying the role:

| Log source | Graylog input | Port |
|---|---|---|
| Windows Events | Beats | 5044 |
| Linux Logs | GELF TCP | 12201 |

**Important for the Beats input**: In the Graylog web UI, when creating or editing the Beats input, enable **«Do not add Beats type as prefix»**. Otherwise Graylog prefixes all field names with `winlogbeat_` (e.g. `winlogbeat_agent_hostname` instead of `agent_hostname`).

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

### Install Linux log forwarding

```yaml
- name: Forward Linux logs to Graylog
  hosts: linux
  gather_facts: true
  roles:
    - role: joe-speedboat.log_forwarder
      vars:
        log_forwarder_graylog_gelf_host: graylog.example.com
        log_forwarder_graylog_gelf_port: 12201
```

### Install Windows event forwarding

```yaml
- name: Forward Windows Event Logs to Graylog
  hosts: windows
  gather_facts: true
  roles:
    - role: joe-speedboat.log_forwarder
      vars:
        winlogbeat_graylog_host: graylog.example.com
        winlogbeat_graylog_port: 5044
```

### Uninstall Linux log forwarding

```yaml
- name: Uninstall Linux log forwarding
  hosts: linux
  gather_facts: true
  tasks:
    - ansible.builtin.include_role:
        name: joe-speedboat.log_forwarder
        tasks_from: uninstall.yml
```

### Uninstall Windows event forwarding

```yaml
- name: Uninstall Windows event forwarding
  hosts: windows
  gather_facts: true
  tasks:
    - ansible.builtin.include_role:
        name: joe-speedboat.log_forwarder
        tasks_from: uninstall.yml
```

---

## Variables

The `log_forwarder_` prefix is used OS-wide; `winlogbeat_` variables apply to Windows only.

### Windows / Winlogbeat — Graylog Connection

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_graylog_host` | `graylog.example.com` | Graylog Beats input host |
| `winlogbeat_graylog_port` | `5044` | Graylog Beats input port |
| `winlogbeat_version` | `7.17.28` | Winlogbeat version (Graylog only supports 7.x) |
| `winlogbeat_download_url` | computed | Download URL for Winlogbeat ZIP |
| `winlogbeat_checksum` | dict (7.17.28, 7.17.29) | SHA512 checksums per version |

### Windows / Winlogbeat — Paths

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_install_root` | `C:\Program Files\Winlogbeat` | Base install directory |
| `winlogbeat_data_dir` | `C:\ProgramData\Winlogbeat` | Data directory |
| `winlogbeat_log_dir` | `C:\ProgramData\Winlogbeat\Logs` | Log directory |
| `winlogbeat_download_dir` | `C:\Windows\Temp\winlogbeat` | Temporary download directory |

### Windows / Winlogbeat — Event Logs & Service

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_event_logs` | Application, System, Security | Event log channels (all events) |
| `winlogbeat_node_name` | `{{ inventory_hostname }}` | Collector node identifier |
| `winlogbeat_service_name` | `winlogbeat` | Windows service name |
| `winlogbeat_service_state` | `started` | Desired service state |
| `winlogbeat_service_enabled` | `true` | Auto-start on boot |
| `winlogbeat_retention_versions` | `3` | Old versions retained during upgrades |
| `winlogbeat_tags` | `[]` | Additional tags |
| `winlogbeat_fields` | — | Additional fields (e.g. `env`, `datacenter`) |

### Windows / Winlogbeat — Graylog Fields

`fields_under_root: true` under `output.logstash:` injects custom fields at root level in Graylog (no `winlogbeat_` prefix).

| Field | Value | Description |
|---|---|---|
| `log_type` | `winlogbeat` | Identifies Windows events in Graylog |
| `agent_type` | `{{ winlogbeat_node_name }}` | Collector node name |
| `host` | `{{ winlogbeat_node_name }}` | Hostname in Graylog `source` |
| `host_ip` | `{{ ansible_default_ipv4.address }}` | Host IP |

### Linux / Fluent Bit — Graylog Connection

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_graylog_gelf_host` | `graylog.example.com` | Graylog GELF TCP host |
| `log_forwarder_graylog_gelf_port` | `12201` | Graylog GELF TCP port |
| `log_forwarder_graylog_gelf_mode` | `tcp` | GELF mode (tcp or udp) |
| `log_forwarder_fluent_bit_workers` | `2` | Fluent Bit output worker count |

### Linux / Fluent Bit — Log Sources

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_linux_install` | `true` | Enable Linux log forwarding |
| `log_forwarder_journal_tag` | `journald` | Tag for journald input |
| `log_forwarder_read_from_tail` | `true` | Start journald reading from tail |
| `log_forwarder_storage_path` | `/var/lib/fluent-bit` | Fluent Bit state directory |
| `log_forwarder_systemd_db` | `{{ log_forwarder_storage_path }}/flb_systemd.db` | journald cursor DB path |
| `log_forwarder_audit_enabled` | `true` | Enable auditd + audit log forwarding |
| `log_forwarder_audit_tag` | `audit` | Tag for audit input |
| `log_forwarder_audit_db` | `{{ log_forwarder_storage_path }}/flb_audit.db` | audit cursor DB path |
| `log_forwarder_audit_rules_file` | `/etc/audit/rules.d/logbeat-package.rules` | Package-change audit rules file |
| `log_forwarder_security_files_enabled` | `true` | Forward auth.log/secure + web server logs |
| `log_forwarder_security_files_tag` | `security_files` | Tag for security file input |
| `log_forwarder_security_files_db` | `{{ log_forwarder_storage_path }}/flb_security_files.db` | Security file cursor DB path |

Default `log_forwarder_security_log_paths`:

```yaml
log_forwarder_security_log_paths:
  - /var/log/auth.log
  - /var/log/secure
  - /var/log/nginx/*.log
  - /var/log/apache2/*.log
  - /var/log/httpd/*.log
```

### Linux / Fluent Bit — Uninstall Options

| Variable | Default | Description |
|---|---|---|
| `log_forwarder_remove_repo` | `true` | Remove Fluent Bit repository on uninstall |
| `log_forwarder_audit_remove` | `true` | Remove audit packages on uninstall |

Set to `false` to keep packages/repositories configured after uninstall.

---

## Audit Rules

The role deploys package-change audit rules for each OS family:

### Ubuntu

Monitors `dpkg`, `apt`, `apt-get`, `apt-cache` and `python3` execution:

```text
-w /usr/bin/dpkg -p x -k package_change
-w /usr/bin/apt -p x -k package_change
-w /usr/bin/apt-get -p x -k package_change
-w /usr/bin/apt-cache -p x -k package_change
-w /usr/bin/python3 -p x -k package_change
```

### Rocky / RHEL

The same rules are generated dynamically in the Fluent Bit config template. Rocky 8 uses `command`-based tasks instead of native `yum_repository`/`dnf` modules, because RHEL 8's system Python 3.6 is incompatible with Ansible 2.20+ module wrappers.

---

## Task Layout

```
tasks/
├── main.yml              # Template dispatcher (entry point)
├── include-file.yml      # OS folder selection via Ansible facts
├── uninstall.yml         # Uninstall dispatcher (for include_role tasks_from)
│
├── Ubuntu/               # Fluent Bit — Debian family
│   ├── 00_validate.yml   #   Assert mandatory variables
│   ├── 10_install.yml    #   GPG key, apt repo, fluent-bit + auditd packages
│   ├── 20_configure.yml  #   Fluent Bit config, parsers, audit rules
│   ├── 30_service.yml    #   Enable/start auditd + fluent-bit
│   └── uninstall.yml     #   Stop, purge fluent-bit + auditd, remove config/repo
│
├── rhelAll/              # Fluent Bit — RHEL family (Rocky 9+)
│   ├── 00_validate.yml   #   Assert mandatory variables
│   ├── 10_install.yml    #   yum_repository, dnf install fluent-bit + audit
│   ├── 20_configure.yml  #   Fluent Bit config, parsers, audit rules
│   ├── 30_service.yml    #   Enable/start auditd + fluent-bit
│   └── uninstall.yml     #   Stop, dnf remove, delete config/repo
│
├── rhelAll-8/            # Fluent Bit — RHEL 8 (command-based tasks)
│   ├── 10_install.yml    #   command-based repo, rpm import, dnf install
│   └── uninstall.yml     #   command-based stop, dnf remove, file cleanup
│
└── Windows/              # Winlogbeat — Windows family
    ├── 00_validate.yml   #   Assert mandatory variables
    ├── 10_install.yml    #   Download ZIP, extract, junction, service install
    ├── 20_configure.yml  #   Deploy winlogbeat.yml config
    ├── 30_service.yml    #   Enforce service state
    └── uninstall.yml     #   Stop, uninstall-service, remove directories
```

The template dispatcher (`main.yml` + `include-file.yml`) discovers numbered task files (regex `^[0-9]{2}_.*.yml$`) across all OS folders at runtime and selects the first matching file by OS specificity.

---

## Templates

| Template | Destination | Purpose |
|---|---|---|
| `fluent-bit-logbeat.conf.j2` | `/etc/fluent-bit/fluent-bit.conf` | Fluent Bit main config: systemd/tail inputs, auth/sudo parsers, GELF output |
| `parsers-logbeat-forwarder.conf.j2` | `/etc/fluent-bit/parsers-logbeat-forwarder.conf` | Regex parsers: audit_raw, audit_package_fields, auth_sshd, auth_pam, sudo |
| `audit-package.rules.j2` | `/etc/audit/rules.d/logbeat-package.rules` | Audit rules for package-change monitoring (per OS family) |
| `winlogbeat.yml.j2` | `{{ winlogbeat_install_path }}\winlogbeat.yml` | Winlogbeat config: event log inputs, Beats output, field injection |

---

## Handlers

| Name | Action | Trigger |
|---|---|---|
| `restart winlogbeat` | Restarts Winlogbeat service | Config change on Windows |
| `stop winlogbeat` | Stops Winlogbeat service | Uninstall on Windows |
| `restart fluent-bit` | Restarts Fluent Bit service | Config change on Linux |
| `reload audit rules` | Runs `augenrules --load` | Audit rules change on Linux |

---

## Winlogbeat Upgrades

Winlogbeat is installed into versioned directories and exposed via a `current` junction:

```yaml
winlogbeat_version: '7.17.28'
```

Upgrade workflow:
1. Stop service → extract ZIP → backup previous installation
2. Repoint `current` junction → deploy config → start service
3. Clean up old versions beyond `winlogbeat_retention_versions`

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

## Runtime Checks

### Linux

```bash
systemctl is-active fluent-bit
systemctl is-enabled fluent-bit
systemctl is-active auditd
auditctl -l | grep package_change
fluent-bit -c /etc/fluent-bit/fluent-bit.conf --dry-run
```

### Windows PowerShell

```powershell
Get-Service winlogbeat
Get-Content 'C:\Program Files\Winlogbeat\current\winlogbeat.yml'
```

### Graylog Search

After enabling **«Do not add Beats type as prefix»** on the Graylog Beats input, all fields arrive without the `winlogbeat_` prefix (e.g. `agent_hostname`, not `winlogbeat_agent_hostname`).

| Source | Query |
|---|---|
| journald | `source:<host> AND log_type:journald` |
| Audit (Rocky) | `source:<host> AND log_type:auditd AND package_action:*` |
| Audit (Ubuntu) | `source:<host> AND log_type:auditd AND (package_action:install OR package_action:remove)` |
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