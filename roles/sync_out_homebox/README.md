# sync_out_homebox

Feeds Nautobot devices, modules and inventory items to Homebox as one CSV import. The feed is one way: Nautobot owns
every column the CSV carries, and the role never reads Homebox data back except the asset IDs.

The role reads with the `networktocode.nautobot.graphql_info` module, renders one Jinja template into a CSV and posts it
to Homebox's `/api/v1/entities/import` endpoint as a multipart upload. It runs no script and needs no Python beyond
`pynautobot`, which the module requires.

Role inputs are listed in [meta/argument_specs.yml](meta/argument_specs.yml) and in the table below. The Nautobot read
token and the Homebox API key come from the caller's secrets store at run time; the role has no default for either.

## Behavior

One GraphQL query returns `devices`, `modules` and `inventory_items`, with their custom fields, tags and location
trees. The role turns each record into one CSV row:

| Record | Import ref | Parent | Location |
| --- | --- | --- | --- |
| Device | Asset tag, else `<prefix><uuid>` | none | Location path, else the default location |
| Module in a bay | Asset tag, else `<prefix><uuid>` | The host device | The host device's location |
| Module on a shelf | Asset tag, else `<prefix><uuid>` | none | Its own location path, else the default location |
| Inventory item | Asset tag, else `<prefix><uuid>` | Its device | The device's location |

A module nested in another module's bay attaches to the host device. A module or inventory item whose asset tag is
already in use by an earlier row falls back to its UUID ref, and the run log counts these renames.

A location path joins the Nautobot location tree with ` / `, up to `sync_out_homebox_location_depth` levels.

### What Nautobot owns and what Homebox keeps

| Nautobot owns (overwritten on every import) | Homebox keeps |
| --- | --- |
| Name, manufacturer, model number, serial number | Attachments and photos |
| Location, parent, archived state | Maintenance entries |
| Purchase date, price and seller, receipt reference, asset tag | Items that have no Nautobot record |
| Tags and the custom fields named in `sync_out_homebox_custom_fields` | |

Homebox replaces an item's tags and custom fields with the CSV values on every import, so a tag added by hand in
Homebox to a fed item does not survive the next run. `sync_out_homebox_extra_tags` adds the same tags to every fed row.

### Which modules are fed

Devices and inventory items are always fed. A module is fed or skipped by these rules, first match wins:

1. Its asset tag starts with one of `sync_out_homebox_always_include_asset_tag_prefixes`, or it carries
   `sync_out_homebox_include_tag`: **fed**.
2. Its asset tag matches one of `sync_out_homebox_exclude_asset_tag_patterns` (regular expressions): **skipped**.
3. Its price is known and below `sync_out_homebox_min_price_cents`: **skipped**.
4. Otherwise: **fed**.

An unknown price never skips a module. A `sync_out_homebox_min_price_cents` of 0 turns the price floor off, and with
the defaults nothing is skipped. The run log reports how many modules each rule skipped.

### Asset IDs

Homebox assigns a new asset ID to every imported row that has none. When `sync_out_homebox_apply` and
`sync_out_homebox_preserve_asset_ids` are both true, the role first reads Homebox's `/api/v1/entities/export`, maps each
import ref to its current asset ID and writes that ID into the CSV, so a re-import keeps the IDs stable.

### Output

`sync_out_homebox_artifact_path`, when set, receives the CSV (mode `0600`). The upload runs only when
`sync_out_homebox_apply` is true and the play is not in check mode; otherwise the role renders and stops. To feed a
second Homebox, call the role again with another `sync_out_homebox_url` and `sync_out_homebox_api_key`.

## Usage

```yaml
- name: Feed Nautobot hardware to Homebox
  hosts: localhost
  gather_facts: false
  roles:
    - role: jpe_ansible.nautobot.sync_out_homebox
      vars:
        sync_out_homebox_nautobot_url: "{{ nautobot_url }}"
        sync_out_homebox_read_token: "{{ nautobot_read_token }}"
        sync_out_homebox_url: "{{ homebox_url }}"
        sync_out_homebox_api_key: "{{ homebox_api_key }}"
        sync_out_homebox_apply: true
        sync_out_homebox_min_price_cents: 500
        sync_out_homebox_always_include_asset_tag_prefixes: [CPU, GPU, SSD]
        sync_out_homebox_exclude_asset_tag_patterns: ['^NET-CLIP']
```

## Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `sync_out_homebox_nautobot_url` | required | Nautobot base URL; must be https, or http on a loopback host. |
| `sync_out_homebox_read_token` | required | Nautobot token with read access to devices, modules and inventory items. |
| `sync_out_homebox_url` | unset | Homebox base URL, required when `apply` is true. |
| `sync_out_homebox_api_key` | unset | Homebox API key, required when `apply` is true. |
| `sync_out_homebox_apply` | `false` | Upload the CSV. When false, the role only renders it. |
| `sync_out_homebox_artifact_path` | `""` | File that receives the rendered CSV. |
| `sync_out_homebox_preserve_asset_ids` | `true` | Read the current asset IDs before an upload and keep them. |
| `sync_out_homebox_always_include_asset_tag_prefixes` | `[]` | Module asset tag prefixes that are always fed. |
| `sync_out_homebox_include_tag` | `""` | Nautobot tag that always feeds a module. |
| `sync_out_homebox_exclude_asset_tag_patterns` | `[]` | Regular expressions that skip a module by asset tag. |
| `sync_out_homebox_min_price_cents` | `0` | Skip a module with a known price below this; 0 is off. |
| `sync_out_homebox_price_field` | `purchase_price_cents` | Custom field key that holds the price, in cents. |
| `sync_out_homebox_custom_fields` | four fields | Nautobot custom field key, Homebox column and optional divisor. |
| `sync_out_homebox_asset_tag_column` | `HB.field.Asset tag` | Column that carries the asset tag; empty turns it off. |
| `sync_out_homebox_extra_tags` | `[]` | Tags added to every fed row. |
| `sync_out_homebox_archived_statuses` | `[Decommissioning, Offline]` | Nautobot statuses that archive an item. |
| `sync_out_homebox_location_depth` | `4` | Levels of the location tree in a location path. |
| `sync_out_homebox_default_location` | `Unassigned` | Location for a record with none. |
| `sync_out_homebox_ref_prefix` | `nautobot-` | Prefix of the import ref built from a UUID. |

## Tests

The Molecule scenario renders a saved GraphQL response through the same tasks and templates and checks the header, the
row set, each scope rule, the precedence between them, CSV quoting and the asset ID map. It sends nothing to Homebox.

## Known limits

The GraphQL field names in the query are checked against the Nautobot schema by an introspection run, not by the
scenario. Only the first `sync_out_homebox_location_depth` levels of a location tree reach the CSV.
