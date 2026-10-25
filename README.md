# ansible-nautobot

Ansible collection `jpe_ansible.nautobot`: roles that run Nautobot as a system of record, load data into it from live
sources, publish data out of it, check changes against it, and report drift.

[![CI Gate][ci-badge]][ci-workflow]
[![License](https://img.shields.io/github/license/jpe-ansible/ansible-nautobot)](LICENSE)

## Installation

Add the collection to a `requirements.yml` and install it into a project-local collections path:

```yaml
collections:
  - name: https://github.com/jpe-ansible/ansible-nautobot.git
    type: git
    version: <release tag>
```

```sh
ANSIBLE_COLLECTIONS_PATH=./collections ansible-galaxy collection install -r requirements.yml
```

The collection declares its collection dependencies in `galaxy.yml`; `requirements.yml` pins the versions this
repository tests against. The controller also needs the Python libraries in `requirements.txt`, which
`meta/execution-environment.yml` hands to `ansible-builder`.
`requirements-test.txt` pins the Molecule toolchain that CI and local test runs install.

## Usage

Reference a role by its fully qualified name:

```yaml
roles:
  - role: jpe_ansible.nautobot.sync_in
```

| Role | Purpose |
| --- | --- |
| `server` | Install and configure Nautobot, its service accounts and their tokens. |
| `sync_in` | Load the Bootstrap data, the live Proxmox inventory and caller-supplied rows into Nautobot. |
| `sync_out` | Build the export document from a saved GraphQL query and publish it to object storage. |
| `sync_out_homebox` | Render Nautobot devices, modules and inventory items as a Homebox CSV import and upload it. |
| `preflight` | Fail a run when published guest memory or storage intent contradicts Nautobot. |
| `drift` | Compare Nautobot with its sources and send one event per run to a log collector. |

Each role's README lists its inputs and behavior. Every role takes its credentials from variables that the caller
reads from a secrets store at run time; no role has a default for a token.

## Bootstrap data

`global_settings.yml` and `develop.yml` at the repository root declare the shared Nautobot taxonomy and the saved
GraphQL query. The SSoT Bootstrap integration loads them from a Git repository record that the `sync_in` role
manages. See [docs/bootstrap.md](docs/bootstrap.md).

## Development

The repository ships a Nix development shell. Run `direnv allow` once, then from the repository root:

```sh
pre-commit run --all-files
```

Run `molecule test -s default` from `molecule/<role>/` for each role. Use a project-local
`ANSIBLE_COLLECTIONS_PATH` for every lint, commit and molecule run so installs stay out of shared collection
directories.

Run the live Nautobot and Proxmox API integration scenario from `molecule/live/`.

## Contributing

Open a pull request against `develop`. Keep commits atomic, and keep Python, shell and Ruby files out of the roles:
each role is Ansible tasks, templates and data. Every role has a Molecule scenario; add or update the scenario that
covers your change.

## License

Apache License 2.0. See [LICENSE](LICENSE).

[Documentation](https://docs.jacobpevans.com)

[ci-badge]: https://github.com/jpe-ansible/ansible-nautobot/actions/workflows/ci-gate.yml/badge.svg
[ci-workflow]: https://github.com/jpe-ansible/ansible-nautobot/actions/workflows/ci-gate.yml
