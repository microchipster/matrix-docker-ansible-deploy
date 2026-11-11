<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up the Android SMS/MMS agent (optional)

The playbook can install and configure the [android-sms-mms-adb-agent](https://github.com/microchipster/mautrix-android-sms-mms) for you.

It is the phone host side of the [mautrix-android-sms-mms bridge](configuring-playbook-bridge-mautrix-android-sms-mms.md): it reads SMS/MMS from a USB-connected Android device over [adb](https://developer.android.com/tools/adb) (non-root or root; [GrapheneOS](https://grapheneos.org/) and [LineageOS](https://lineageos.org/) work without root) and exposes them as a small authenticated HTTP API that the bridge consumes.

The agent runs as a `matrix-android-sms-mms-agent.service` systemd service (not as a container), because it needs adb — and therefore USB — access to the device. It is built from the same repository as the bridge.

See the project's [documentation](https://github.com/microchipster/mautrix-android-sms-mms) (in particular `docs/AGENT_API.md`) to learn what it does and why it might be useful to you.

## Prerequisites

- An Android phone with **USB debugging** enabled and authorized for the host the agent runs on.
- The `adb` binary installed on the agent host (for example, the `adb` or `android-tools-adb` package).
- Access to the phone's USB device. The service runs as `root` by default, which works without additional setup. If you run it as an unprivileged user, set up [udev rules for the device](https://developer.android.com/studio/run/device#setting-up) (typically by adding the user to the `plugdev` group) and keep the phone authorized.
- Docker on the agent host — it is used to build the agent binary from source. The playbook installs Docker on the hosts it manages.
- A bearer token for the agent's HTTP API. There is no default — you must define one.

## Adjusting the playbook configuration

To enable the agent, add the following configuration to your `inventory/host_vars/matrix.example.com/vars.yml` file:

```yaml
matrix_android_sms_mms_agent_enabled: true

# The bearer token required by all agent routes except /v1/health.
# Generate one (for example, with `pwgen -s 64 1`) and keep it secret.
# The bridge uses it when logging in, and can pre-fill it automatically
# (see the bridge's documentation).
matrix_android_sms_mms_agent_token: YOUR_AGENT_TOKEN

# The phone's own number(s), so they are excluded from thread recipients.
matrix_android_sms_mms_agent_own_numbers:
  - '+12025550123'
```

### Extending the configuration

There are some additional things you may wish to configure about the agent. See [`roles/custom/matrix-android-sms-mms-agent/defaults/main.yml`](../roles/custom/matrix-android-sms-mms-agent/defaults/main.yml) for the full list.

Common options:

- `matrix_android_sms_mms_agent_listen` — the HTTP bind address (default: `127.0.0.1:8093`). Set it to `0.0.0.0:8093` if the bridge runs in a container on the same machine or on a different machine.
- `matrix_android_sms_mms_agent_serial` — selects the adb device (adb's `-s` argument). Leave empty to use the only attached device.
- `matrix_android_sms_mms_agent_adb_path` — the `adb` binary to use (default: `adb`).
- `matrix_android_sms_mms_agent_sub_ids` — limits which SIM subscriptions are bridged. An empty list means all of them.
- `matrix_android_sms_mms_agent_poll_interval_ms` / `matrix_android_sms_mms_agent_sweep_overlap_ms` — the polling interval (default: `3000`) and sweep overlap (default: `600000`, i.e. 10 minutes).
- `matrix_android_sms_mms_agent_history_limit` — caps history pages returned per request (default: `200`).
- `matrix_android_sms_mms_agent_send_enabled`, `matrix_android_sms_mms_agent_send_isms_tx_code`, `matrix_android_sms_mms_agent_send_allow_sms_split`, `matrix_android_sms_mms_agent_send_caller_package`, `matrix_android_sms_mms_agent_send_resolve_timeout_ms` — outgoing message settings. `allow_sms_split` defaults to `false`, because the shell backend cannot do real multipart: long messages fail loudly instead of silently fracturing.

The agent's data (watermarks and dedupe state) is stored under `matrix_android_sms_mms_agent_data_dir` (default: `/matrix/android-sms-mms-agent/data`).

## Installing

After configuring the playbook, run it with [playbook tags](playbook-tags.md) as below:

<!-- NOTE: let this conservative command run (instead of install-all) to make it clear that failure of the command means something is clearly broken. -->
```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

The shortcut commands with the [`just` program](just.md) are also available: `just install-all` or `just setup-all`

## Usage

Check that the agent is running and sees the phone:

```sh
systemctl status matrix-android-sms-mms-agent
curl -fsS http://127.0.0.1:8093/v1/health
```

The health response includes the device model, serial, whether root is available and the capabilities (threads, history, media, read state, and SMS/MMS sending).

Then configure a login on the bridge (see the [bridge's usage section](configuring-playbook-bridge-mautrix-android-sms-mms.md#usage)). The bridge asks for the agent URL and bearer token (pre-filled when both components are enabled in the same playbook run).

### Running on a separate host

The agent must run on the machine the phone is plugged into. When that is not the Matrix server, the agent role still installs fine (it is self-contained), but you need to make its HTTP API reachable from the bridge — for example by setting `matrix_android_sms_mms_agent_listen: 0.0.0.0:8093` and restricting access to the bridge's address via a firewall, or by tunneling the connection.

## Troubleshooting

As with all other services, you can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the machine with SSH and running `journalctl -fu matrix-android-sms-mms-agent`.

Common problems:

- **The service exits with `Failed to query device over adb`** — the phone is not connected, USB debugging is not enabled, or the host was never authorized for this phone. Run `adb devices` as the user the service runs as (root by default) and check that the device is listed as `device` (not `unauthorized`).
- **Permission errors accessing the device** — when running the service as an unprivileged user, make sure udev rules grant that user access to the phone (usually through the `plugdev` group) and that the service's `matrix_android_sms_mms_agent_user_name` / `matrix_android_sms_mms_agent_group_name` match.
- **The bridge cannot reach the agent** — verify `matrix_android_sms_mms_agent_listen` and any firewall rules. The agent refuses to listen on a non-loopback address without a token.
- **Rebuilding** — the binary is rebuilt when the configured `matrix_android_sms_mms_agent_repo_version` changes or when the binary is missing. Run the playbook again (`just install-all`) after changing `matrix_android_sms_mms_agent_repo_version`.
