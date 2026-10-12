# sync_out

The SSoT Bootstrap YAML defines the saved query; this role only runs it by name.

Role inputs are listed in [meta/argument_specs.yml](meta/argument_specs.yml); optional `sync_out_contract_version`
selects the export schema.

Object storage credentials come from the caller's environment, not from role variables.

## Behavior

The saved query returns six collections: `vlans`, `prefixes`, `ip_addresses`, `devices`, `racks` and `interfaces`.
The transform checks that each collection is present, rejects a rack without a site, and requires `last_updated` on
every record. It renders the document from `templates/export.json.j2` and logs `Data as of <timestamp>` with the newest
`last_updated` value, so a stale export is visible in the run log. Publishing is skipped in check mode.

The role reads the saved query text with the token's `view` permission and posts it to the GraphQL endpoint, so the
token needs no `add` or `run` permission on saved queries.

The play order guarantees the export follows the inbound sync: run this role after `sync_in` in the same playbook.

## Usage

```yaml
- name: Publish the Nautobot export
  hosts: localhost
  gather_facts: false
  roles:
    - role: jpe_ansible.nautobot.sync_out
      vars:
        sync_out_nautobot_url: "{{ nautobot_url }}"
        sync_out_read_token: "{{ nautobot_read_token }}"
        sync_out_query_name: nautobot-export
        sync_out_object_storage_bucket: "{{ export_bucket }}"
```

## Contract test

The Molecule scenario renders a fixture response through `tasks/transform.yml` and validates the document against the
export schema at the pinned `sync_out_contract_version` of the contract repository. It also proves the strict schema
rejects the removed `virtual_machines` collection and the removed IP address `type` key, and that a rack without a
site fails with the stable message.

## Known limits

The GraphQL property names in the saved query (`last_updated`, `dns_name`, `mac_address`, `mgmt_only` and the
relation fields) are checked against the live schema when the output is compared with the previous exporter's output,
not by this role's tests.
