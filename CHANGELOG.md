# Changelog

## [Unreleased]

### Changed

- **Winlogbeat output fields (Windows):** reduce the default `fields` block to only `log_type` + `journal_forwarder`, and add `add_host_metadata` processor with NetInfo so host metadata is carried dynamically. Static `agent_type`/`host` fields are no longer rendered by default.

## [v1.1.0] — 2026-06-15

### Changed

- **Winlogbeat distribution**: Switch Windows downloads to Elastic's OSS-only Winlogbeat artifacts (`winlogbeat-oss-<version>-windows-x86_64.zip`) for Apache 2.0 licensing and update the default to the latest tested OSS release, `9.4.2`, while keeping the same extracted directory layout and service-script installer workflow. ([#10])

### Documentation

- **Graylog search**: Correct the Windows Events example query to use the lab-verified `agent_type:winlogbeat` discriminator instead of `log_type:winlogbeat`, because Windows events from `win2201` populate `agent_type` while `log_type` is not present in Graylog.

## [v1.0.1] — 2026-06-12

### Bugfixes

- **Winlogbeat config**: Guard `winlogbeat_fields` in template against undefined variable. The commented-out default caused `AnsibleUndefinedVariable` when deploying without explicitly setting `winlogbeat_fields`. ([#7])
- **Winlogbeat upgrade**: Remove redundant `backup/` folder. During upgrade, the old version folder was both kept in place AND copied to `backup/<version>/`. Now the backup step is removed — the old version folder itself is preserved and rotation is handled solely by `winlogbeat_retention_versions`. Cleanup rotates versioned install directories and removes any legacy `backup` directory left by older role versions. ([#9])
- **Uninstall handler**: Prevent `reload audit rules` handler failure during uninstall. Removing the audit package removes `augenrules`, causing the handler to fail with `No such file or directory`. Added `ignore_errors: true` to the handler. ([#2])
- **Dynamic Graylog hostname**: Use Fluent Bit built-in `${HOSTNAME}` macro instead of static `{{ ansible_hostname }}` in the audit and security_file `modify` filters. This ensures the Graylog `source` field dynamically follows the actual hostname. ([#5])

### Documentation

- **README**: Complete overhaul — fixed task layout (added `rhelAll-8/` and `uninstall.yml` per OS folder), documented all variables (including previously missing Winlogbeat paths, Linux DB paths), added Templates and Handlers sections, added Linux uninstall examples, expanded Graylog search patterns with cross-OS notes and portable queries. ([#3])
- **CHANGELOG**: Created this file. ([#8])

### Maintenance

- **Template revert**: Reverted unintended modification of the template dispatcher control file `tasks/main.yml`. ([#4])

[#2]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/2
[#3]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/3
[#4]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/4
[#5]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/5
[#7]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/7
[#9]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/9
[#10]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/10
[v1.0.0]: https://github.com/joe-speedboat/ansible.log_forwarder/releases/tag/v1.0.0
[v1.0.1]: https://github.com/joe-speedboat/ansible.log_forwarder/releases/tag/v1.0.1
[v1.1.0]: https://github.com/joe-speedboat/ansible.log_forwarder/releases/tag/v1.1.0
