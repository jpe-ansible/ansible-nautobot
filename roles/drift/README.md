# drift

Compares Nautobot virtual machines and tags with a published source inventory, reads object counts, and sends one
structured event to a HEC input. Drift does not fail the role. Nautobot API errors and event delivery errors fail the role.

Role inputs use the [`drift_` prefix](meta/argument_specs.yml).

`drift_source_inventory` contains guest mappings under each key in `drift_source_groups`. Each guest
may provide a `hostname` and a list of `tags`; when `hostname` is absent, the mapping key is used as its name.

## Event policy

The role posts one JSON envelope to `/services/collector/event` with `sourcetype: nautobot:drift`. Its `event` contains
`severity`, the REST counts and drift-list sizes, `missing_in_nautobot`, `extra_in_nautobot`, `tag_drift`, and
`unmodelled_domains`.

| Severity | Condition |
| --- | --- |
| `critical` | A source guest is missing from Nautobot or a Nautobot VM is absent from the source. |
| `warning` | A shared guest is missing one or more source tags. |
| `info` | No guest or tag drift exists. |

Extra Nautobot tags are not reported. The REST counts cover VLANs, prefixes, IP addresses, devices, modules, module types,
circuits, providers, virtual machines, tags, installed modules, and spare modules. Ansible check mode skips event delivery.

## Usage

Run the role from a controller-side scheduled job after resolving the published inventory artifact. Pass its parsed
object as `drift_source_inventory`. Supply `drift_read_token` and `drift_hec_token` from the secrets store as separate
role variables. The role uses the read token only for Nautobot queries and the HEC token only for the event request; both
request tasks suppress secret output.

Read and event requests retry up to two more times, 15 seconds apart, on network errors and server errors. Client errors
(400, 401, 403, 404) fail at once because a retry cannot change them. A retried event post can duplicate one event.
