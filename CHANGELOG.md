# Changelog

## [v1.0.1] — 2026-06-12

### Bugfixes

- **Winlogbeat config**: Guard `winlogbeat_fields` in template against undefined variable. The commented-out default caused `AnsibleUndefinedVariable` when deploying without explicitly setting `winlogbeat_fields`. ([#7])
- **Uninstall handler**: Prevent `reload audit rules` handler failure during uninstall. Removing the audit package removes `augenrules`, causing the handler to fail with `No such file or directory`. Added `ignore_errors: true` to the handler. ([#2])
- **Dynamic Graylog hostname**: Use Fluent Bit built-in `${HOSTNAME}` macro instead of static `{{ ansible_hostname }}` in the audit and security_file `modify` filters. This ensures the Graylog `source` field dynamically follows the actual hostname. ([#5])

### Documentation

- **README**: Complete overhaul — fixed task layout (added `rhelAll-8/` and `uninstall.yml` per OS folder), documented all variables (including previously missing Winlogbeat paths, Linux DB paths), added Templates and Handlers sections, added Linux uninstall examples, expanded Graylog search patterns with cross-OS notes and portable queries. ([#3])
- **CHANGELOG**: Created this file. ([this PR])

### Maintenance

- **Template revert**: Reverted unintended modification of the template dispatcher control file `tasks/main.yml`. ([#4])

[#2]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/2
[#3]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/3
[#4]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/4
[#5]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/5
[#7]: https://github.com/joe-speedboat/ansible.log_forwarder/pull/7
[v1.0.0]: https://github.com/joe-speedboat/ansible.log_forwarder/releases/tag/v1.0.0
[v1.0.1]: https://github.com/joe-speedboat/ansible.log_forwarder/releases/tag/v1.0.1
