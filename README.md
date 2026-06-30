# Ansible Role: `joe-speedboat.winlogbeat_forwarder`

Windows-only Ansible role for forwarding **Windows Event Logs** to **Graylog** with **Winlogbeat** over the Graylog Beats input.

## Requirements

### Control Node

- Ansible >= 2.14
- Windows collections: `ansible.windows` and `community.windows`

Install or refresh the collections on the same Ansible/Rundeck controller environment that runs the playbook:

```bash
ansible-galaxy collection install ansible.windows community.windows --force
```

Verify `win_template` is available:

```bash
ansible-doc ansible.windows.win_template >/dev/null && echo "ansible.windows.win_template OK"
ansible-galaxy collection list | grep -E '^(ansible\.windows|community\.windows)\s'
```

`ansible.windows.win_template` renders Jinja locally on the controller before copying the rendered file to Windows. If the controller collection is missing, stale, or broken, the target may receive raw `.j2` content such as `{% for ... %}` / `{{ variable }}` and Winlogbeat fails validation with a YAML error.

### Target Nodes

- Windows Server 2019 / 2022 / 2025
- WinRM configured
- Outbound access to the Graylog Beats input, default `5044/tcp`

### Graylog

Create a Beats input before deploying this role:

| Log source | Graylog input | Port |
|---|---|---|
| Windows Event Logs | Beats | 5044 |

For the Beats input, enable **Do not add Beats type as prefix** so fields are not renamed with a `winlogbeat_` prefix.

## Quick Start

```yaml
- name: Forward Windows Event Logs to Graylog
  hosts: windows
  gather_facts: true
  roles:
    - role: joe-speedboat.winlogbeat_forwarder
      vars:
        winlogbeat_graylog_host: graylog.example.com
        winlogbeat_graylog_port: 5044
```

## Uninstall

```yaml
- name: Uninstall Windows event forwarding
  hosts: windows
  gather_facts: true
  tasks:
    - ansible.builtin.include_role:
        name: joe-speedboat.winlogbeat_forwarder
        tasks_from: uninstall.yml
```

## Variables

### Graylog Connection

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_graylog_host` | `graylog.example.com` | Graylog Beats input host |
| `winlogbeat_graylog_port` | `5044` | Graylog Beats input port |

### Winlogbeat Version and Paths

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_version` | `9.4.2` | Winlogbeat OSS version tested against Graylog Beats input |
| `winlogbeat_install_root` | `C:\Program Files\Winlogbeat` | Base install directory |
| `winlogbeat_data_dir` | `C:\ProgramData\Winlogbeat` | Data directory |
| `winlogbeat_log_dir` | `C:\ProgramData\Winlogbeat\Logs` | Log directory |
| `winlogbeat_download_dir` | `C:\Windows\Temp\winlogbeat` | Download directory |

### Event Logs and Service

| Variable | Default | Description |
|---|---|---|
| `winlogbeat_event_log_groups` | all keys from `winlogbeat_event_log_group_definitions` | Curated event-log groups to collect |
| `winlogbeat_event_log_group_definitions` | see defaults | Built-in group-to-channel mapping |
| `winlogbeat_event_logs` | unset | Complete manual event-channel override; if set, groups are ignored |
| `winlogbeat_validate_event_channels` | `true` | Validate target event-channel availability during deployment |
| `winlogbeat_report_unforwarded_event_channels` | `true` | Report target channels that contain records but are not selected for forwarding |
| `winlogbeat_service_name` | `winlogbeat` | Windows service name |
| `winlogbeat_service_state` | `started` | Desired service state |
| `winlogbeat_service_enabled` | `true` | Auto-start on boot |

By default, `winlogbeat_event_log_groups` selects all built-in groups from `winlogbeat_event_log_group_definitions`. Override it when a host class should collect only selected categories:

```yaml
winlogbeat_event_log_groups:
  - baseline
  - powershell
  - task_scheduler
```

Built-in groups include `baseline`, `powershell`, `task_scheduler`, `applocker`, `rds`, `user_profile`, `fslogix`, `citrix`, `code_integrity`, `laps`, `storage`, `remote_management`, `defender_firewall`, `windows_update`, `network`, `smb`, `device_lifecycle`, `diagnostics`, `identity`, `server_manager`, `file_services`, `hyper_v`, `printing`, `remote_assistance`, `security_hardening`, `setup_provisioning`, `time_service`, `volume_shadow_copy`, `openssh`, and `exchange`.

## Checking Windows Event Log Coverage

The role checks event-log coverage in two directions:

1. Selected channels that are missing on the target.
2. Target channels that contain records on Windows but are not selected for Winlogbeat forwarding to Graylog.

The coverage report shows the configured channels first, including whether each selected path exists and how many records it currently contains. This helps identify configured paths that are empty, nearly empty, or absent on the target. It then shows populated target channels that are not currently collected. During deployment, the role prints:

```text
Configured Windows Event Log channels selected for Winlogbeat forwarding on <host>:
configured_count=<number>
[<channel path> (records=<count>), <channel path> (records=0), <channel path> (missing), ...]
Windows Event Log channels with records on <host> but not selected for Winlogbeat forwarding:
unforwarded_count=<number>
[<channel path> (records=<count>), <channel path> (records=<count>), ...]
```

Channels with zero records are included in the configured-channel list when selected, so you can see which configured paths are empty or rarely used. Zero-record channels are still ignored by the unforwarded report because they are not evidence of missed data yet.

The channel names are the Windows Event Log tree paths, for example:

```text
Application (records=123)
Security (records=456)
System (records=789)
Microsoft-Windows-TerminalServices-LocalSessionManager/Operational (records=12)
Microsoft-Windows-WinRM/Operational (records=34)
```

Use this output to decide whether a host class needs another group in `winlogbeat_event_log_groups`, or a direct `winlogbeat_event_logs` override.

Set this to disable the coverage report while keeping selected-channel validation enabled:

```yaml
winlogbeat_report_unforwarded_event_channels: false
```

The role only reports missing channels that are part of the **effective Winlogbeat configuration**. It does not compare against every possible built-in group.

If you override `winlogbeat_event_log_group_definitions`, Ansible replaces the complete group definition map. For example, this config only defines `Application` for `baseline`:

```yaml
vars:
  winlogbeat_event_log_group_definitions:
    baseline:
      - name: Application
```

With that input, the role can only check/report `Application`. It will not report `System`, `Security`, PowerShell, or Task Scheduler as missing because they are no longer selected.

To test whether specific channels are available on a host, use `winlogbeat_event_logs` as an explicit checklist:

```yaml
vars:
  winlogbeat_event_logs:
    - name: Application
    - name: System
    - name: Security
    - name: Microsoft-Windows-PowerShell/Operational
    - name: Windows PowerShell
    - name: Microsoft-Windows-TaskScheduler/Operational
    - name: Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational
    - name: Microsoft-FSLogix-Apps/Operational
```

During deployment, the role prints `Missing Winlogbeat event channels` only when at least one selected channel is absent. If no message appears, all selected channels were found.

Use `ignore_missing_channel: false` when a channel is mandatory for a host class:

```yaml
winlogbeat_version: '9.4.2'
```

The role uses Elastic's OSS-only ZIP artifacts (`winlogbeat-oss-<version>-windows-x86_64.zip`) for Apache 2.0 licensing. The OSS ZIP still extracts to `winlogbeat-<version>-windows-x86_64`, so the same service scripts and junction-based installer workflow are used.

Upgrade workflow:
1. Stop service -> extract ZIP -> repoint `current` junction
2. Deploy config -> install service -> start service
3. Clean up old versions beyond `winlogbeat_retention_versions` (default: keep 3)
4. Remove any legacy `backup` directory left by older role versions
5. The old version folder remains intact until it exceeds retention - no separate backup copy is created

---

## Installation

```bash
git clone https://github.com/joe-speedboat/ansible.winlogbeat_forwarder.git /etc/ansible/roles/joe-speedboat.winlogbeat_forwarder
```

Or via Galaxy (once published):

```bash
ansible-galaxy role install joe-speedboat.winlogbeat_forwarder
```

## Graylog Search

Useful validation queries:

```text
winlog_channel:Application AND agent_type:winlogbeat
winlog_channel:Security AND agent_type:winlogbeat
winlog_provider_name:<provider> AND agent_type:winlogbeat
winlogbeat_forwarder:true
```

## License

Apache License 2.0
