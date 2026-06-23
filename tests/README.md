# Test harness

This role uses the joe-speedboat OS-independent task dispatcher. Run tests from a separate harness with symlinks instead of `role: ../../`, so `tasks/main.yml` resolves `role_path|basename` correctly.

```bash
mkdir -p /tmp/log-forwarder-harness/roles
ln -sfn "$PWD" /tmp/log-forwarder-harness/roles/joe-speedboat.log_forwarder
ln -sfn "$PWD" /tmp/log-forwarder-harness/roles/ansible.winlogbeat_forwarder
# If your checkout directory has a different basename, add that too:
ln -sfn "$PWD" "/tmp/log-forwarder-harness/roles/$(basename "$PWD")"

ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --syntax-check
ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --diff
ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --diff
```

The `ansible.winlogbeat_forwarder` or checkout-basename symlink covers runs where Ansible resolves `role_path|basename` to the repository directory instead of the Galaxy role name.

Always verify beyond a green play recap on Windows targets:

```powershell
Get-Service winlogbeat
& "C:\Program Files\Winlogbeat\current\winlogbeat.exe" test config -c "C:\Program Files\Winlogbeat\current\winlogbeat.yml" -e
Get-Content "C:\Program Files\Winlogbeat\current\winlogbeat.yml" -TotalCount 40
```

Generate a marker event and verify it arrives in Graylog before claiming end-to-end success.
