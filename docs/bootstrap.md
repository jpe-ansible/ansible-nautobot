# SSoT Bootstrap data

`global_settings.yml` and `develop.yml` at the repository root declare the shared Nautobot taxonomy and the outbound
saved GraphQL query. The SSoT Bootstrap integration reads them from a Git repository datasource.

## Usage

Create one Nautobot Git Repository named `Nautobot Bootstrap` that points at this repository, with the provided data
type `config contexts`. The `sync_in` role creates it, syncs it and runs the Bootstrap job. The setup guide requires the
repository name to contain `Bootstrap` and the provided data type to include `config contexts`
([Bootstrap setup](https://docs.nautobot.com/projects/ssot/en/latest/admin/integrations/bootstrap_setup/)).

The two YAML files sit at the repository root because the integration reads only the root. In its Git mode the adapter
changes into the repository directory and loads every `.yml` and `.yaml` file found by `os.listdir("./")`, naming each
by its file stem; it then selects the environment file named after the branch (`develop`) and the `global_settings`
file ([adapter source](https://github.com/nautobot/nautobot-app-ssot/blob/develop/nautobot_ssot/integrations/bootstrap/diffsync/adapters/bootstrap.py)).
Keep every other YAML file at the repository root valid for `yaml.safe_load`.

`develop.yml` supplies the default branch for Git Repository objects that name none. No staging or production file is
needed.

## Declared objects

See [global_settings.yml](../global_settings.yml) for the declared objects and custom fields, including their content
types.

The `location` object is `homelab` of type `Site`; the Proxmox stage places node Devices in the Location named by
`sync_in_pve_location_name`.

The integration identifies custom fields by `label` and takes no explicit `key`. Nautobot derives the key from the label
when none is given, so each label in `global_settings.yml` yields its key.

## Boundaries

Site-specific Locations, VLANs, Prefixes, IP addresses, Devices, Device Types, Racks, Virtual Machines, Clusters,
Modules, and Inventory Items are not declared here. The `sync_in` role writes them with collection modules from the
live Proxmox API and from the caller's site data. Roles, Statuses and Tags are created by those flows on demand, so
this data declares none of them and the `role` model stays out of `bootstrap_models_to_sync`.

The integration deletes locally defined GraphQL queries that its data does not list. Define every saved query in
`global_settings.yml`.
The Bootstrap plan does not list deletions performed by the integration.

The saved query follows the export field map, and its root and nested properties validate against the deployed GraphQL
schema. The query omits Virtual Machines and IP address `type`, which the strict export contract does not allow.
No Scheduled Job is included; the caller's scheduler runs the collection flows.
