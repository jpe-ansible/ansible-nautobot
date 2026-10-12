# AI Agents Configuration

Ansible collection `jpe_ansible.nautobot` provides Nautobot server, inbound-sync, outbound-sync, pre-flight, and drift roles, plus SSoT Bootstrap YAML.

## Rules

- Do not add `.py`, `.sh`, or `.rb` files; roles have no `files/` directories.
- The roles read secrets from a secrets store at run time.
