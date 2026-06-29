# Changelog

## [Unreleased]

### Changed

- Add more optional Windows event-log groups from additional production coverage reports, including certificates/identity, diagnostics, file services, Hyper-V, OpenSSH, printing, remote assistance, security hardening, setup/provisioning, time service, volume shadow copy, Exchange, and expanded RDS/SMB/storage/network coverage.

## [v1.0.1] - 2026-06-29

### Added

- Add `AGENT.md` maintainer guidance for agents working on the role, including validation workflow and the pattern for deriving useful optional event-log groups from pasted Ansible coverage output.

### Changed

- Add optional Windows-native event-log groups for remote management, Defender/firewall, Windows Update, network, SMB, device lifecycle, diagnostics, identity, server manager, RDS, and storage coverage.
- Add deployment-time reporting for Windows Event Log channels that contain records on the target but are not selected for Winlogbeat forwarding.
- Remove leftover Linux-role references from the Windows-only role and render `winlogbeat_forwarder: "true"` as the role marker field.
