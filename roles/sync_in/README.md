# sync_in

Synchronizes data into Nautobot in three ordered stages. All role inputs use the `sync_in_` prefix, and
`meta/argument_specs.yml` lists each one.

| Stage | Source | Writes |
| --- | --- | --- |
| `bootstrap` | The Bootstrap data repository | A Git Repository record, a repository sync, and a run of the Bootstrap job |
| `proxmox` | The live Proxmox VE API | Nodes, guests, interfaces, IP addresses and disks |
| `curated` | Caller-supplied rows | Racks, hardware, modules, circuits, VLANs, prefixes and IP reservations |

`sync_in_stages` selects the stages to run. An unknown stage fails
the run with `sync_in_stages holds an unknown stage; known stages: bootstrap, proxmox, curated`.

`sync_in_curated_components` selects curated families. `dcim` selects racks,
non-node devices, and their BMC interfaces. `hardware` selects hardware devices, modules, module bays, spare modules,
WAN providers, and circuits. VLANs, prefixes, and IP reservations run when both families are selected. If either
family is omitted, the role skips VLANs, prefixes, and IP reservations.

`sync_in_pve_components` selects Proxmox families. `nodes` selects
Proxmox node Clusters and Devices. `virtualization` selects guest Clusters, VirtualMachines, VMInterfaces, IP
associations, and primary IPs. `drives` selects discovered drive InventoryItems. Each family reads, applies, prunes,
and checks its own source records.

- `name` identifies the live Proxmox node.
- `nautobot_device_name` sets the Nautobot Device name when supplied.
- `commissioned` defaults to true when omitted.
- `management_dns_name` selects the management reservation and defaults to the Device name.

When `sync_in_plan_only` is `true`, the role reads and transforms the selected stages, sets the
`sync_in_plan` fact, and performs no Nautobot writes. Plan-only runs require `sync_in_read_token` and do not require
`sync_in_write_token`. The same plan is produced when a plan-only run uses Ansible check or diff mode.

`sync_in_plan` is a dictionary keyed by family. Each family has `create`, `update`, and `delete` lists. Every row
contains `object_kind` and a stable `identifier`; update rows also contain `changed_fields`, mapping each changed
field name to its `old` and `new` values. Read the update list with bracket syntax, for example
`sync_in_plan.virtualization['update']`, because `update` is also a dictionary method.

The virtualization plan also contains `adoptions`, one per exact-name match. Each row names the existing VM, its
Nautobot ID, the live VMID and cluster, and the fields that would change. Duplicate names and matches in a different
cluster fail the stage with the VM name. An existing IP address without `sync_source: proxmox` can be associated with a
guest interface, but the stage does not update, mark, or prune that address.
The drives plan also contains `expected_counts`, keyed by Proxmox node. After a write and prune, the stage reads back
owned InventoryItems from Nautobot and fails if any node's persisted count differs. Plan-only runs report the expected
counts and skip this readback.

```yaml
sync_in_plan:
  virtualization:
    create: []
    update:
      - object_kind: virtual_machine
        identifier: guest-name (VMID 101)
        changed_fields:
          memory:
            old: 2048
            new: 4096
    delete: []
```

## Usage

```yaml
- name: Synchronize Nautobot
  hosts: localhost
  gather_facts: false
  roles:
    - role: jpe_ansible.nautobot.sync_in
      vars:
        sync_in_nautobot_url: "{{ nautobot_url }}"
        sync_in_read_token: "{{ nautobot_read_token }}"
        sync_in_write_token: "{{ nautobot_write_token }}"
        sync_in_bootstrap_repo_url: "{{ bootstrap_repo_url }}"
```

The caller reads the tokens from its secret store at run time. The role has no default for a token or a URL.

Run the stages without applying the planned changes:

```yaml
- name: Plan Nautobot synchronization
  hosts: localhost
  gather_facts: false
  roles:
    - role: jpe_ansible.nautobot.sync_in
      vars:
        sync_in_stages:
          - proxmox
          - curated
        sync_in_plan_only: true
        sync_in_nautobot_url: "{{ nautobot_url }}"
        sync_in_read_token: "{{ nautobot_read_token }}"
        sync_in_pve_api_url: "{{ pve_api_url }}"
        sync_in_pve_token_id: "{{ pve_read_token_id }}"
        sync_in_pve_token_secret: "{{ pve_read_token_secret }}"
        sync_in_pve_ca_path: "{{ pve_ca_path }}"
```

## Bootstrap stage

The stage creates or updates the Git Repository record named by `sync_in_bootstrap_repo_name`, with the provided
content type `extras.configcontext`. It syncs the repository, enables the job named by `sync_in_bootstrap_job_name`
when it is disabled, runs the job against the Git load source with `dryrun` off, and polls each job result until it
reaches a terminal status. A status other than `SUCCESS` fails the run with the status in the message, for example
`Bootstrap job finished with status FAILURE`. A job that is missing fails with `Bootstrap job not found: <name>`.

The two reads retry transient failures twice, 15 seconds apart. The writes that queue a job do not retry, because a
replayed request can enqueue a second job; the next scheduled run repeats a failed stage. Check mode runs the reads
and reports the plan without writing.

The data repository is described in `docs/bootstrap.md`.
Set the `default` Git Repository branch in `develop.yml`; `sync_in_bootstrap_branch` defaults to that value and must
match it. The Bootstrap plan does not list deletions performed by the integration.

## Proxmox stage

The stage reads nodes, guests and each node's physical disk list from the Proxmox VE API with a read-only token, and
reads Nautobot with one GraphQL query. It writes one Cluster, one Device, and one management interface per commissioned
node, one virtual machine per guest, one interface per guest network entry, guest addresses, and one inventory item per
physical drive. Node Devices and Clusters, guests, owned guest addresses, and discovered drives carry
`sync_source: proxmox`; management interfaces and existing management reservations do not receive that marker. Shared
reference objects (statuses, roles, tags, manufacturers, cluster type, node device type) are created when absent and
never deleted.

The nodes family preserves the Device Type, Location, and Role of existing Devices. It creates a management-only
`mgmt` interface when absent and associates the existing IP address whose `dns_name` matches the mapped management
DNS name. The address is not created or updated, and a matching address becomes the Device primary IPv4 address.
Missing reservations appear in `sync_in_plan.nodes.unbound_nodes`; the stage fails when every commissioned node lacks
a matching reservation.

Bootstrap must define `sync_source` on Devices, Clusters, InventoryItems, VirtualMachines, VMInterfaces and IPAddresses;
`physical_ram_mb` on Devices; and `current_node`, `live_disks` and `vmid` on VirtualMachines. Node-map Device names are
used for guest `current_node`, drive InventoryItem ownership and expected drive counts. Cluster names continue to use
the Proxmox node names.

Guest interfaces and addresses come from the guest configuration, so stopped guests are covered. A QEMU interface
`netN` takes its address from `ipconfigN`; a container takes the interface name, MAC and address from its `netN`
entry. DHCP, manual and missing addresses give an interface with no address, templates are skipped, and a guest
address is written only when a Prefix in `sync_in_pve_namespace_name` contains it. An address that already carries
another `sync_source` is associated with the interface but not rewritten. The first accepted address of the lowest
interface becomes the guest's primary IPv4 address. The curated stage creates the Prefixes, so addresses appear on
the run after they exist.

Before any write the stage rejects an incomplete read: an offline node, an empty disk list, a missing guest field or a
missing part of the Nautobot snapshot fails with `Proxmox read incomplete: <part>` or `Nautobot read incomplete:
<objects>`. An owned exact-name VirtualMachine that moves to another Proxmox node is moved to the corresponding Cluster
in place before its remaining fields are synchronized. A matching VM owned by another flow still fails when its Cluster
differs. The stage deletes only objects it owns that the source no longer reports, in dependency order, and fails instead
when the plan exceeds `sync_in_prune_max_fraction` of the owned objects and exceeds `sync_in_prune_min_allowance`.
A Location named by
`sync_in_pve_location_name` must exist; the Bootstrap data defines one.

## Curated stage

The stage reads Nautobot with one GraphQL query, normalizes the rows in `sync_in_racks`, `sync_in_devices`,
`sync_in_hardware_devices`, `sync_in_hardware_modules`, `sync_in_wan_circuits`, `sync_in_vlans`, `sync_in_prefixes`
and `sync_in_reservations`, and writes them with collection modules. A module row with `quantity` above one expands
into owned units whose asset tags end in `#1`, `#2`, and so on. Module bay positions derive from the asset tag and are
stable between runs. A row for a drive that live inventory already reports writes nothing, and the stage removes only
surplus modules that it owns, identified by the `sync_source` custom field value `curated`. Every written object
carries `sync_source: curated`.

The stage creates a manufacturer or a Device role that a row names when Nautobot lacks it, and leaves existing ones
unchanged.

An empty list for an input writes nothing for that input.

## Known limits

- Python 3.13 and later verify certificates strictly, and a certificate authority without a key usage extension fails
  that check. The Proxmox cluster CA has no such extension, so the proxmox stage needs a controller Python older
  than 3.13 or a Proxmox endpoint certificate from an authority that passes strict verification.

The curated stage requires each circuit type to exist before it runs, because the collection's circuit type module
accepts no custom fields.
