# preflight

Checks published guest memory and disk-pool intent against Nautobot before a converge. All role inputs use the
`preflight_` prefix. The role reads inventory with a GraphQL POST and evaluates it with Jinja; the evaluation task
does not access the network.

## Prerequisites

The SSoT Bootstrap YAML defines the custom fields `physical_ram_mb` on Device and `current_node` and `live_disks`
on VirtualMachine. `live_disks` is a JSON list of objects with `interface` and `datastore` keys. The caller supplies
`preflight_desired_guests` as a mapping from guest name to `node`, `memory_mb`, and `disks`; each disk has an
`interface` and `datastore_id`. The caller supplies `preflight_nautobot_url` and the read-only
`preflight_read_token` when fetch is enabled. The token has no role default and is sent only in the authorization
header.

The Nautobot GraphQL guide states: “Additionally, by default, all custom fields in GraphQL will be prefixed with
`cf_`.” [GraphQL guide](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/).
The query uses `cf_physical_ram_mb`, `cf_current_node`, and `cf_live_disks` from those Bootstrap keys; the fields
are queryable after Bootstrap creates them and the Nautobot web service restarts.

The query roots follow the model names. Nautobot's schema generator names each list root from the model's plural
verbose name (`list_name = str_to_var_name(model._meta.verbose_name_plural)` in
[`nautobot/core/graphql/schema.py`](https://github.com/nautobot/nautobot/blob/develop/nautobot/core/graphql/schema.py)),
which gives `devices` and `virtual_machines`. A list query returns every row the token may view
(`queryset.restrict(user, "view")` in
[`generators.py`](https://github.com/nautobot/nautobot/blob/develop/nautobot/core/graphql/generators.py)),
and applies `limit` and `offset` only when the query passes them. A token without view permission on a type
therefore gets an empty list, not an error; the role treats an empty device list as incomplete data.

The [Virtual Machine user guide][vm-guide] labels allocated memory in GB, while the tagged model source labels
`VirtualMachine.memory` as “Memory (MB)” ([tagged model source][vm-model]).
This role follows the model field and treats its integer MB value, custom `physical_ram_mb`, and caller-provided
`memory_mb` as MiB under the shared `_mb` input convention. The evaluation task normalizes all three values to
`memory_mib` once before comparing them.

[vm-guide]: https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/virtualization/virtualmachine/
[vm-model]: https://github.com/nautobot/nautobot/blob/v3.2.0/nautobot/virtualization/models.py

A caller can compose this role in one pre-converge playbook with its own live-cluster checks, so every check runs
before a converge. This role checks the caller's published desired state against Nautobot inventory, with memory
reported in MiB.

## Usage

- **Playbook:** call `jpe_ansible.nautobot.preflight` with the published desired guest mapping.
- **Scheduler:** run the preflight playbook as a stage before converge stages.
- **CI:** set `preflight_skip_fetch: true` and provide fixture values for the normalized Nautobot records and
  desired guest mapping.

The RAM check sums live guests by `current_node` and desired guests by `node`, then compares each sum with the
matching Device's physical capacity. The desired placement sum includes live guests outside the desired guest map,
and each live guest in that map is counted at its desired node. The storage-pool check compares each desired disk's
`datastore_id` with the live `datastore` for the same guest and interface. Missing inventory fields, GraphQL errors,
and an empty device list fail with a `Pre-flight data incomplete:` message. A Device must carry a valid
`cf_physical_ram_mb` only when a desired guest's `node` or a live guest's `current_node` names it; other Devices
need no RAM value. Two Nautobot virtual machines with one name, or two live disks of one guest with one interface,
fail with `Pre-flight data incomplete: duplicate guest <name>` or
`Pre-flight data incomplete: duplicate live disk <guest>/<interface>`.
