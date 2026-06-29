# Changelog

## [Unreleased]

No unreleased changes.

## [v1.0.1] - 2026-06-29

### Changed

- Add optional Windows-native event-log groups for remote management, Defender/firewall, Windows Update, network, SMB, device lifecycle, diagnostics, identity, server manager, RDS, and storage coverage.
- Add deployment-time reporting for Windows Event Log channels that contain records on the target but are not selected for Winlogbeat forwarding.
- Remove leftover Linux-role references from the Windows-only role and render `winlogbeat_forwarder: "true"` as the role marker field.
