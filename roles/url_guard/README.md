# url_guard

Fails when a URL that carries a credential does not use https. Plain http is accepted only for a loopback host
(`127.0.0.1`, `localhost`, `[::1]`), which the test scenarios use. The accepted pattern is defined once, in
`defaults/main.yml`; the other roles of the collection include this role before their first credentialed call.

Role inputs use the [`url_guard_` prefix](meta/argument_specs.yml).

## Usage

```yaml
- name: Require https for the Nautobot URL
  ansible.builtin.include_role:
    name: jpe_ansible.nautobot.url_guard
  vars:
    url_guard_urls:
      - {name: sync_in_nautobot_url, url: "{{ sync_in_nautobot_url }}"}
```
