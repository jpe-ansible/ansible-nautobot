# Live Nautobot scenario

This scenario runs the inbound-sync Proxmox and curated stages against a disposable Nautobot instance and a local HTTPS
Proxmox API mock. It uses generated service accounts for inbound sync, pre-flight, outbound sync and drift, while the
superuser is reserved for setup and verification. The Bootstrap stage is omitted because the stock image has no job
runner.

The Proxmox stage populates physical RAM, current-node and live-disk custom fields from the synthetic API responses. The
curated stage places a Device in a Rack, records its position, face and Device Type height, and ignores a curated row
with the Proxmox node role. It creates a Circuit Type through the REST API, reconciles a circuit that uses it, and
checks that a missing Circuit Type fails with the circuit and type named in the error.
It adopts an exact-name VirtualMachine without replacing its identity, retains manual addresses on its unowned
interfaces while adding live addresses, reports that adoption in the plan, and rejects
duplicate names and an unowned matching name in a different Cluster with reason-specific failures. It maps a Proxmox
node to a different Device name and checks drive placement and guest `current_node`. It moves an owned guest to another
Cluster in place, checks that its identity, interfaces and addresses remain attached, and verifies a zero-change repeat.
It associates an existing unowned guest address without changing its description or ownership marker, and confirms the
address is retained when its source guest is removed.
It binds management reservations to commissioned nodes, reports one missing reservation, skips an uncommissioned node,
and fails plan-only mode when all commissioned nodes lack reservations.

The pre-flight role reads the live inventory and accepts desired guest memory and disk pools derived from those records. It
also rejects desired RAM above a node's physical capacity, a changed disk pool, and a disk interface absent from the live
guest.
The outbound-sync role runs the saved GraphQL query from the shared Bootstrap data and validates its document against the
same export contract used by its fixture scenario. It checks that live guest addresses appear in the document.
The drift role sends one informational event for matching live guests and one critical event for a source guest absent
from Nautobot. The local HTTPS mock records each request body and echoes the observed authorization header for an in-memory
assertion.
