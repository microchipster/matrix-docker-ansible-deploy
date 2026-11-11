<!--
SPDX-FileCopyrightText: 2026 MDAD project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Local image preloading

Some components are [self-built](self-building.md): their container images are built from a git checkout on the server itself. Building on the server is undesirable when the server is weak (e.g. a small VPS or a Raspberry Pi) or when it runs production workloads that must not be disturbed by heavy builds.

Instead of building on the server, the playbook can build the images on the machine running Ansible (the controller) and copy them to the server. The server then has each image present by the time the component's role runs and skips building it.

No container registry is involved: images are copied directly to the server over the Ansible connection.

## Components with preloading enabled

The playbook registers its self-building components for preloading, and each one opts in through its own toggle. All toggles default to `false`, so preloading is not enabled for any component out of the box:

| Component | Preloading toggle |
| --- | --- |
| [Text A Day](configuring-playbook-bot-text-a-day.md) | `matrix_bot_text_a_day_container_image_preload` |
| [mautrix-jmpchat](configuring-playbook-bridge-mautrix-jmpchat.md) | `matrix_bridge_mautrix_jmpchat_container_image_preload` |
| [Matrix Invite Portal](configuring-playbook-invite-portal.md) | `matrix_invite_portal_container_image_preload` |

Your own components can be added through `matrix_playbook_image_preload_components_custom` (see below).

Preloading is disabled entirely by setting `matrix_playbook_image_preload_enabled` to `false`, which makes every component build on the server as before.

## Requirements

- The controller must have `git`, Docker (or a compatible CLI configured via `matrix_playbook_image_preload_docker_command`, e.g. `rtk docker`) and network access to the components' git remotes
- Builds on the controller run with host networking (`--network=host`), so they do not depend on a usable default Docker bridge on the controller
- When the controller architecture differs from the server architecture, the image is built with `--platform` for the server's architecture, which requires emulation on the controller (on Linux: `docker run --privileged --rm tonistiigi/binfmt --install all`)

## Configuration

| Variable | Default | Description |
| --- | --- | --- |
| `matrix_playbook_image_preload_enabled` | `true` | Controls whether local image preloading runs at all |
| `matrix_playbook_image_preload_docker_command` | `docker` | Docker command to use on the controller |
| `matrix_playbook_image_preload_controller_checkout_root` | `~/.cache/matrix-docker-ansible-deploy/image-preload` (respects `$XDG_CACHE_HOME`) | Directory on the controller where the components' repositories are checked out |
| `matrix_playbook_image_preload_transfer_compressor` | `auto` | Compression for the transfer: `auto` (zstd when both ends have it, gzip otherwise), `zstd`, `gzip` or `none` |
| `matrix_playbook_image_preload_components_auto` | playbook-defined | Components preloaded by the playbook's own wiring |
| `matrix_playbook_image_preload_components_custom` | `[]` | Your own components to preload |
| `matrix_playbook_image_preload_host_temp_dir` | `/matrix/image-preload` | Directory on the server used while loading image archives |

## Preloading custom components

To preload the image of a component maintained outside this playbook, add an entry to `matrix_playbook_image_preload_components_custom` in your `vars.yml` file:

```yaml
matrix_playbook_image_preload_components_custom:
  - name: my-component
    enabled: "{{ my_component_enabled and my_component_container_image_self_build }}"
    repo_url: "{{ my_component_container_image_self_build_repo }}"
    version: "{{ my_component_container_image_self_build_branch }}"
    image: "{{ my_component_container_image }}"
    host_src_files_path: "{{ my_component_container_src_files_path }}"
    service: my-component
    service_name: matrix-my-component.service
```

The entry is processed when the component's own role would run, matching the same tags (`setup-all`, `install-all`, `setup-<service>`, `install-<service>`), so `just install-service my-component` preloads only that component.

## How it works

For every enabled component, the playbook:

1. checks out the component's repository (at the configured `version`) into the controller's checkout directory, updating an existing checkout if needed
2. builds the image on the controller if it is missing or was built from a different commit (a label inside the image records the commit)
3. ensures the server's repository checkout is at the exact same commit
4. compares the local and server image layers and copies the image only when they differ
5. restarts the component's service, if it is already installed

When nothing changed, a playbook run does no building, no copying and no restarting.

## Troubleshooting

- Images are always built from commits that exist on the component's git remote: the controller clones the repository itself, and the server checks out the same commit
- A cross-architecture build with `--platform` is slower and requires emulation on the controller (see the requirements above)
- Images are staged in `matrix_playbook_image_preload_host_temp_dir` on the server; make sure there is enough free disk space there
- The controller's checkout directory is a cache and is safe to delete at any time
