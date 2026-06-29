# Changelog

## [Unreleased]

### Changed

- Add deployment-time reporting for Windows Event Log channels that contain records on the target but are not selected for Winlogbeat forwarding.
- Remove leftover Linux-role references from the Windows-only role and render `winlogbeat_forwarder: "true"` as the role marker field.
