# AGENT.md

This file is for coding agents and maintainers working on `joe-speedboat.winlogbeat_forwarder`.

## Role purpose

This is a Windows-only Ansible role that installs and manages Winlogbeat for forwarding Windows Event Logs to a Graylog Beats input.

The role should stay focused on Windows Event Log forwarding. Do not add Linux journald, auditd, syslog, Fluent Bit, or GELF logic here. Linux forwarding belongs in a separate Linux/journal-forwarder role.

## Supported target and controller assumptions

Target systems:

- Windows Server 2022 or newer.
- WinRM access is already configured.
- The target can reach the Graylog Beats input, normally TCP port `5044`.

Controller requirements:

- Ansible 2.14 or newer.
- `ansible.windows` collection.
- `community.windows` collection.

Verify the exact controller environment that will run the role, not only a local development venv:

```bash
ansible-doc ansible.windows.win_template >/dev/null && echo "ansible.windows.win_template OK"
ansible-galaxy collection list | grep -E '^(ansible\.windows|community\.windows)\s'
```

If `ansible.windows.win_template` is missing or broken, the target can receive raw Jinja text instead of rendered YAML, causing Winlogbeat config validation to fail.

## Important files

```text
defaults/main.yml              Default variables, event-log groups, paths, version.
templates/winlogbeat.yml.j2    Rendered Winlogbeat configuration.
tasks/main.yml                 Role task dispatcher.
tasks/Windows/00_validate.yml  Windows preflight and event-log coverage reporting.
tasks/Windows/10_install.yml   Download, extract, junction, service install/upgrade.
tasks/Windows/20_configure.yml Render and validate winlogbeat.yml.
tasks/Windows/30_service.yml   Service state and start mode.
tasks/uninstall.yml            Public uninstall entry point.
tasks/Windows/uninstall.yml    Windows uninstall implementation.
handlers/main.yml              Service restart handler.
README.md                      User-facing documentation.
CHANGELOG.md                   Release notes.
```

## Development rules

- Keep public examples generic. Use `graylog.example.com`, `windows.example.com`, and placeholder credentials. Do not commit real inventories, real internal FQDNs, private IPs, tokens, passwords, or customer identifiers.
- Keep comments and docs plain, technical, and ASCII-only unless a non-ASCII character is required.
- Keep role defaults safe to rerun. A second converge should normally report `changed=0`.
- Validate config before service start/restart. `tasks/Windows/20_configure.yml` runs `winlogbeat.exe test config`; do not remove that guard.
- Prefer Elastic OSS Winlogbeat ZIP artifacts for Apache 2.0 licensing.
- Keep artifact basename and extracted folder separate: the OSS ZIP is named `winlogbeat-oss-<version>-windows-x86_64.zip`, but extracts to `winlogbeat-<version>-windows-x86_64`.
- Use `ssl.enabled: false` for non-TLS Logstash output. Do not use `ssl.verification_mode: none` as a substitute for disabling TLS.
- Use a Windows junction named `current` for version switching.
- Install/uninstall the service through the bundled PowerShell scripts, not an invented `winlogbeat.exe install` command.
- Check service existence with `win_service_info` before stopping/removing a service.
- Do not flush restart handlers early. Let the normal service task set `start_mode: auto` before any restart handler runs.

## Variables and behavior

Key variables:

```yaml
winlogbeat_graylog_host: graylog.example.com
winlogbeat_graylog_port: 5044
winlogbeat_version: '9.4.2'
winlogbeat_event_log_groups:
  - baseline
  - powershell
  - task_scheduler
winlogbeat_validate_event_channels: true
winlogbeat_report_unforwarded_event_channels: true
```

`winlogbeat_event_logs` is a complete manual override. If it is set, `winlogbeat_event_log_groups` are ignored.

`winlogbeat_event_log_groups` is the preferred interface for normal use. It selects curated groups from `winlogbeat_event_log_group_definitions`.

Default groups should stay small and broadly useful:

```yaml
winlogbeat_event_log_groups:
  - baseline
  - powershell
  - task_scheduler
```

Optional groups can be enabled per host class:

```text
applocker
rds
user_profile
fslogix
citrix
code_integrity
laps
storage
remote_management
defender_firewall
windows_update
network
smb
device_lifecycle
diagnostics
identity
server_manager
file_services
hyper_v
printing
remote_assistance
security_hardening
setup_provisioning
time_service
volume_shadow_copy
openssh
exchange
```

## Event-log coverage reporting

The role reports two coverage signals during validation:

1. Configured channels that are missing on the target.
2. Target channels that contain records but are not selected for Winlogbeat forwarding.

The second report is meant to help operators discover logs that exist on a host but are not currently forwarded to Graylog. It prints Event Viewer paths with record counts, for example:

```text
Windows Event Log channels with records on windows.example.com but not selected for Winlogbeat forwarding:
configured_count=28
unforwarded_count=12
[
  "Microsoft-Windows-WinRM/Operational (records=234)",
  "Microsoft-Windows-Windows Defender/Operational (records=98)"
]
```

Channels with zero records are intentionally ignored by the unforwarded report. A zero-record channel exists in Event Viewer, but it is not evidence of missed data yet.

## Adding useful default event-log groups from Ansible output

A practical workflow for improving the default group catalog is to paste the role's Ansible output into an agent and ask it to inspect the coverage report.

Use the `Report Windows Event Log channels not forwarded to Graylog` output as source data. The agent should:

1. Extract the channel paths from the `unforwarded_count` list.
2. Ignore channels from software that is not installed on the test host but exists in production-only groups, such as Citrix, FSLogix, or LAPS, unless the task explicitly asks to improve those groups.
3. Ignore zero-record channels. They should not appear in this report anyway.
4. Prefer adding Windows-native, operationally useful groups over noisy consumer/shell/telemetry channels.
5. Add new channels to focused optional groups, not to the small default `winlogbeat_event_log_groups` list.
6. Keep group names role-oriented and stable, for example `remote_management`, `defender_firewall`, `windows_update`, `network`, `smb`, `device_lifecycle`, `diagnostics`, `identity`, and `server_manager`.
7. Re-run the role with the expanded group list and compare `unforwarded_count` before/after.
8. Run a second converge and require `changed=0`.
9. Verify the service and rendered config after changes.
10. Update README and CHANGELOG when changing defaults or group definitions.

Good candidates from typical Windows Server output:

```yaml
remote_management:
  - name: Microsoft-Windows-WinRM/Operational
  - name: Microsoft-Windows-WMI-Activity/Operational

defender_firewall:
  - name: Microsoft-Windows-Windows Defender/Operational
  - name: Microsoft-Windows-Windows Firewall With Advanced Security/Firewall
  - name: Microsoft-Windows-Windows Firewall With Advanced Security/FirewallDiagnostics

windows_update:
  - name: Microsoft-Windows-WindowsUpdateClient/Operational

network:
  - name: Microsoft-Windows-Dhcp-Client/Admin
  - name: Microsoft-Windows-NCSI/Operational
  - name: Microsoft-Windows-NetworkProfile/Operational
  - name: Microsoft-Windows-Wcmsvc/Operational
  - name: Microsoft-Windows-WinINet-Config/ProxyConfigChanged

smb:
  - name: Microsoft-Windows-SmbClient/Connectivity
  - name: Microsoft-Windows-SMBServer/Operational
```

Be conservative with noisy channels. Do not add channels just because they have records. Many AppX, Shell, Store, telemetry, licensing, and desktop-consumer internals may be high-volume but low-value for infrastructure monitoring. Add them only if there is a clear operational requirement.

## Validation workflow

Use a separate lab harness rather than running directly from the role checkout.

Example harness layout:

```text
/tmp/winlogbeat-forwarder-lab/
  deploy.yml
  roles/
    joe-speedboat.winlogbeat_forwarder -> /path/to/ansible.winlogbeat_forwarder
    ansible.winlogbeat_forwarder       -> /path/to/ansible.winlogbeat_forwarder
```

Example playbook:

```yaml
---
- name: Deploy Winlogbeat to Graylog
  hosts: windows
  gather_facts: true
  roles:
    - role: joe-speedboat.winlogbeat_forwarder
      vars:
        winlogbeat_graylog_host: graylog.example.com
        winlogbeat_graylog_port: 5044
```

Minimum validation before a PR:

```bash
ANSIBLE_ROLES_PATH=/tmp/winlogbeat-forwarder-lab/roles \
ansible-playbook -i inventory.ini deploy.yml --syntax-check

ANSIBLE_ROLES_PATH=/tmp/winlogbeat-forwarder-lab/roles \
ansible-playbook -i inventory.ini deploy.yml --diff

ANSIBLE_ROLES_PATH=/tmp/winlogbeat-forwarder-lab/roles \
ansible-playbook -i inventory.ini deploy.yml --diff
```

The second real converge should report `changed=0`.

Independent Windows checks:

```powershell
Get-Service winlogbeat
& 'C:\Program Files\Winlogbeat\current\winlogbeat.exe' test config -c 'C:\Program Files\Winlogbeat\current\winlogbeat.yml' -e
Get-Content 'C:\Program Files\Winlogbeat\current\winlogbeat.yml'
```

For Graylog ingest validation, generate a unique Application event and search Graylog for the exact marker. Check at least:

```text
source
message
winlog_channel
winlog_provider_name
winlog_event_id
agent_type
```

Do not rely only on custom fields such as `log_type`; Graylog Beats input behavior can vary by input settings.

## Uninstall and reinstall checks

When changing install, service, path, version, or uninstall logic, test the full lifecycle:

1. Install.
2. Idempotency run.
3. Uninstall through `tasks_from: uninstall.yml`.
4. Verify service, install directory, data directory, and download directory are absent as intended.
5. Reinstall.
6. Final idempotency run.
7. Confirm service is running and config validates.

## PR checklist

Before opening or updating a PR:

- `git diff --check` has no output.
- Syntax check passes.
- Real Windows deploy succeeds if task logic/defaults changed.
- Second converge shows `changed=0`.
- `winlogbeat.exe test config` returns `Config OK`.
- Service is running and automatic when install state is expected.
- README and CHANGELOG match the actual behavior.
- Public docs/examples contain no real internal hosts, private IPs, secrets, or credentials.
- If changing Graylog search examples, verify the query against real Graylog output first.

## Release notes

For a release:

1. Move completed entries from `[Unreleased]` into a versioned section such as `[v1.0.1] - YYYY-MM-DD`.
2. Leave `[Unreleased]` present for future changes.
3. Validate syntax after documentation-only changes too.
4. Open a PR rather than tagging directly unless explicitly instructed and authorized to push upstream tags.
