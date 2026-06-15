# Test harness

This role uses the joe-speedboat OS-independent task dispatcher. Run tests from a separate harness with symlinks instead of `role: ../../`, so `tasks/main.yml` resolves `role_path|basename` correctly.

```bash
mkdir -p /tmp/log-forwarder-harness/roles
ln -sfn "$PWD" /tmp/log-forwarder-harness/roles/joe-speedboat.log_forwarder
ln -sfn "$PWD" /tmp/log-forwarder-harness/roles/ansible.log_forwarder
# If your checkout directory has a different basename, add that too:
ln -sfn "$PWD" "/tmp/log-forwarder-harness/roles/$(basename "$PWD")"

ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --syntax-check
ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --diff
ANSIBLE_ROLES_PATH=/tmp/log-forwarder-harness/roles \
  ansible-playbook -i tests/inventory tests/deploy.yml --diff
```

The `ansible.log_forwarder` or checkout-basename symlink covers runs where Ansible resolves `role_path|basename` to the repository directory instead of the Galaxy role name.

For a full Linux matrix, use fresh lab VMs for Ubuntu 24/26 and Rocky 8/9/10. Rocky 8 needs Python 3.9 bootstrapped before normal modules when the control node uses modern Ansible:

```bash
ansible -i inventory.ini rocky8 -m raw -a 'dnf -y module enable python39 && dnf -y install python39'
```

If lab DNS is stale or missing for newly-created VMs, keep the inventory aliases as the actual hostnames but pin `ansible_host=<ip>` after verifying the IP by SSH. This was required during the 2026-06-15 fresh Linux matrix run.

Always verify beyond a green play recap:

```bash
ansible -i inventory.ini linux -m shell -a '
  systemctl is-enabled fluent-bit
  systemctl is-active fluent-bit
  systemctl is-active auditd
  fluent-bit -c /etc/fluent-bit/fluent-bit.conf --dry-run
  grep -n "Add     hostname" /etc/fluent-bit/fluent-bit.conf
  grep -n "Gelf_Host_Key" /etc/fluent-bit/fluent-bit.conf
  auditctl -l | grep package_change
'
```
