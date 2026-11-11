<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Mautrix Android SMS/MMS bridging (optional)

<sup>Refer the common guide for configuring mautrix bridges: [Setting up a Generic Mautrix Bridge](configuring-playbook-bridge-mautrix-bridges.md)</sup>

The playbook can install and configure [mautrix-android-sms-mms](https://github.com/microchipster/mautrix-android-sms-mms) for you.

It is a bridge between Matrix and Android **SMS/MMS** — group chats, attachments/images and tapback reactions included — without Google services. It works with DeGoogled devices (GrapheneOS, LineageOS) and never touches RCS.

The project provides two components:

- `mautrix-android-sms-mms` — the bridgev2 appservice (Matrix ↔ agent), installed by this role on the Matrix server.
- `android-sms-mms-adb-agent` — an agent that talks to the phone over ADB and exposes a small authenticated HTTP API the bridge consumes. It runs on the host the phone is plugged into. See [Setting up the Android SMS/MMS agent](configuring-playbook-android-sms-mms-agent.md).

See the project's [documentation](https://github.com/microchipster/mautrix-android-sms-mms) to learn what it does and why it might be useful to you.

## Prerequisites

To use this bridge, you need:

- an Android phone with USB debugging enabled, plugged into (or reachable from) a host running the [Android SMS/MMS agent](configuring-playbook-android-sms-mms-agent.md);
- the agent's HTTP API URL and bearer token;
- (optional) [Appservice Double Puppet](configuring-playbook-appservice-double-puppet.md) if you want to set up [Double Puppeting](https://docs.mau.fi/bridges/general/double-puppeting.html) automatically.

### Enable Appservice Double Puppet (optional)

If you want to set up [Double Puppeting](https://docs.mau.fi/bridges/general/double-puppeting.html) (hint: you most likely do) for this bridge automatically, you need to have enabled [Appservice Double Puppet](configuring-playbook-appservice-double-puppet.md) service for this playbook.

See [this section](configuring-playbook-bridge-mautrix-bridges.md#set-up-double-puppeting-optional) on the [common guide for configuring mautrix bridges](configuring-playbook-bridge-mautrix-bridges.md) for details about setting up Double Puppeting.

## Adjusting the playbook configuration

To enable the bridge, add the following configuration to your `inventory/host_vars/matrix.example.com/vars.yml` file:

```yaml
matrix_bridge_mautrix_android_sms_mms_enabled: true

# The agent's HTTP API address. Pre-fills the bridge's login flow.
# The default is http://127.0.0.1:8093, which only works when the bridge container
# can reach that address (see "Running the agent on the same machine" below).
matrix_bridge_mautrix_android_sms_mms_default_agent_url: http://agent.example.com:8093

# The agent's bearer token. Pre-fills the bridge's login flow.
matrix_bridge_mautrix_android_sms_mms_default_agent_token: YOUR_AGENT_TOKEN
```

### Enable encryption

The bridge requires end-to-bridge encryption for its management rooms — it fails closed when the Matrix leg is not encrypted. Unless you have already enabled it globally, add this to your `vars.yml` file:

```yaml
matrix_bridges_encryption_enabled: true
```

### Running the agent on the same machine

When the agent runs on the same machine as the bridge, the bridge's container cannot reach the host's `127.0.0.1`. Either:

- expose the agent on the host's LAN address (set `matrix_android_sms_mms_agent_listen: 0.0.0.0:8093` in the agent's configuration) and point `matrix_bridge_mautrix_android_sms_mms_default_agent_url` at it; or
- add a host gateway to the bridge container and keep the agent on `127.0.0.1`:

```yaml
matrix_bridge_mautrix_android_sms_mms_container_extra_arguments:
  - '--add-host=host.docker.internal:host-gateway'
matrix_bridge_mautrix_android_sms_mms_default_agent_url: http://host.docker.internal:8093
```

The agent's bearer token is always required when it listens on a non-loopback address (the agent refuses to start otherwise).

### Extending the configuration

There are some additional things you may wish to configure about the bridge.

See [this section](configuring-playbook-bridge-mautrix-bridges.md#extending-the-configuration) on the [common guide for configuring mautrix bridges](configuring-playbook-bridge-mautrix-bridges.md) for details about variables that you can customize and the bridge's default configuration, including [bridge permissions](configuring-playbook-bridge-mautrix-bridges.md#configure-bridge-permissions-optional), [encryption support](configuring-playbook-bridge-mautrix-bridges.md#enable-encryption-optional), [relay mode](configuring-playbook-bridge-mautrix-bridges.md#enable-relay-mode-optional), [bot's username](configuring-playbook-bridge-mautrix-bridges.md#set-the-bots-username-optional), etc.

Bridge-specific connector options (see `connector/config.go` in the project) are available as `matrix_bridge_mautrix_android_sms_mms_*` variables in [`roles/custom/matrix-bridge-mautrix-android-sms-mms/defaults/main.yml`](../roles/custom/matrix-bridge-mautrix-android-sms-mms/defaults/main.yml):

- `matrix_bridge_mautrix_android_sms_mms_default_agent_url` — the agent URL pre-filled in the login flow (default: `http://127.0.0.1:8093`)
- `matrix_bridge_mautrix_android_sms_mms_default_agent_token` — the agent bearer token pre-filled in the login flow (default: empty)
- `matrix_bridge_mautrix_android_sms_mms_send_reactions_as_text` — send Matrix reactions as plain text messages instead of dropping them (default: `false`)
- `matrix_bridge_mautrix_android_sms_mms_history_default_days` — the automatic catch-up/backfill window in days (default: `30`)
- `matrix_bridge_mautrix_android_sms_mms_max_attachment_size_mb` — caps outgoing attachment size, in MB (default: `50`)
- `matrix_bridge_mautrix_android_sms_mms_device_name` — overrides the device display name used for the login's remote name / room topic (default: empty)
- `matrix_bridge_mautrix_android_sms_mms_pairing_enabled` — serve and expose the public app pairing API under `/_smsmms/` (default: `true`)
- `matrix_bridge_mautrix_android_sms_mms_pairing_public_url` — externally reachable base URL embedded into the pairing payload (default: `https://matrix.example.com`)
- `matrix_bridge_mautrix_android_sms_mms_pairing_listen` — the in-container address the pairing API listens on (default: `:29322`)
- `matrix_bridge_mautrix_android_sms_mms_cdn_domain` — public hostname of the encrypted file service (default: `cdn.example.com`; empty disables the service)
- `matrix_bridge_mautrix_android_sms_mms_cdn_enabled` — explicit override of the automatic provisioning-based enablement (unset by default, in which case the service is enabled exactly when `cdn_domain` is non-empty)

The bridge's command prefix is `!sms` (see `matrix_bridge_mautrix_android_sms_mms_command_prefix`).

The bridge does not publish container images yet, so `matrix_bridge_mautrix_android_sms_mms_container_image_self_build` defaults to `true` and the image is built from the project's git repository. See [Self-building](self-building.md) for details.

## Installing

After configuring the playbook, run it with [playbook tags](playbook-tags.md) as below:

<!-- NOTE: let this conservative command run (instead of install-all) to make it clear that failure of the command means something is clearly broken. -->
```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

The shortcut commands with the [`just` program](just.md) are also available: `just install-all` or `just setup-all`

`just install-all` is useful for maintaining your setup quickly ([2x-5x faster](../CHANGELOG.md#2x-5x-performance-improvements-in-playbook-runtime) than `just setup-all`) when its components remain unchanged. If you adjust your `vars.yml` to remove other components, you'd need to run `just setup-all`, or these components will still remain installed. Note these shortcuts run the `ensure-matrix-users-created` tag too.

## Usage

To use the bridge, you need to start a chat with `@android-sms-mmsbot:example.com` (where `example.com` is your base domain, not the `matrix.` domain).

To log in to a phone, run `login` in the management room. The bridge asks for the agent URL and bearer token (pre-filled from the `matrix_bridge_mautrix_android_sms_mms_default_agent_*` variables), and creates one login per phone. You can add multiple phones (one login each).

After logging in, the bridge creates its per-user space and management room structure. Run `sync 30d` (or `sync 7d`, `sync full`) to backfill history. Each phone thread becomes a stable Matrix room; adding or removing group participants does not fracture the room.

Note that Matrix rooms you create on your own are not automatically bridged to SMS/MMS.

## Contact names from a CardDAV address book

The bridge can import contact names (and photos) from a CardDAV address book, so auto-named rooms use your own contacts' names. In the management room, run:

```
carddav-login <you@example.com> <password>
```

You can also pass a full address book URL (`carddav-login <addressbook URL> <username> <password>`) if you know it. With just an email address, the bridge probes a list of well-known CardDAV addresses (mailcow/SOGo, Nextcloud, Radicale, Mail-in-a-Box, Fastmail, Google and others) and uses the first one that responds. The command message (which contains your password) is deleted from the room automatically, and credentials are only stored if the connection test succeeds.

Then preview the import and apply it:

```
carddav-sync --no-apply
carddav-sync
```

Stored CardDAV credentials and imported contact names are encrypted with AES-256-GCM. The encryption key is kept in the bridge's memory only. Unlock it with

```
carddav-unlock <a secret you choose>
```

in the management room (ideally an encrypted room; the command message is removed from the room afterwards). The secret always derives the same key, so after a bridge restart you can simply run `carddav-unlock` with the same secret again — nothing is lost and nothing needs to be re-fetched. Because the key is never written to disk, a copy of the bridge database or its backups cannot reveal your contacts.

`carddav-status` shows the current state (`Encryption: locked` / `Encryption: unlocked` / legacy key file). `carddav-lock` forgets the in-memory key immediately, and `carddav-reset yes` removes the stored credentials and imported names entirely. If the secret is ever lost, run `carddav-reset yes`, then `carddav-login` and `carddav-sync` again — the address book still lives on your CardDAV server and can always be re-fetched.

While the bridge is locked (for example, right after a restart until you unlock it), background contact syncing pauses.

Installations created before this feature used a key file at `/data/carddav.key`. Running `carddav-unlock` migrates them to the in-memory key; after that the file can be deleted.

Note that contact names used in room names and avatars are Matrix room state that your homeserver serves to room members — encryption cannot hide those from the server, and while the bridge is running the key is necessarily in its memory.

Run `carddav-status` to check the configuration and `carddav-logout` to remove the stored credentials.

## Pairing the Android app

Besides (or instead of) the ADB agent, you can pair the Fossify Messages bridge app on the phone. The app then polls the bridge directly over HTTPS, so the phone does not need to be reachable from the server and no USB connection is required.

Get the signed APK from the [Messages Bridge releases page](https://github.com/microchipster/fossify-messages-bridge/releases) and install it on the phone:

```sh
adb install -r MessagesBridge-<version>.apk
```

Then open **Messages Bridge**, grant the SMS permissions it asks for and make it the default SMS app (required for sending/receiving MMS and for read-state sync; plain SMS works without it). It installs side by side with the regular Fossify Messages app.

The bridge serves the pairing API under `/_smsmms/v1/` on container port `29322` by default, and the playbook reverse-proxies that path to the bridge, so the app reaches it at `https://matrix.example.com/_smsmms/v1/...`. When you use your own reverse proxy instead of the playbook-managed Traefik, make sure requests for `/_smsmms/` are forwarded to the bridge's pairing port (see below).

To pair a phone:

1. In the management room, run `!sms login` and choose **App pairing (QR)** (or run `login app-pair`).
2. Scan the QR code with the app (Settings → Bridge pairing), or enter the homeserver URL, the bridge URL and the pairing code shown in the room manually. The code is single-use and expires after 10 minutes.
3. Once the app completes the pairing, the bridge posts a confirmation message into the room.

Paired devices are stored as logins with the `app` transport. Use `!sms devices` to list them and `!sms unpair <device_id|all>` to revoke one or all of them.

The `/_smsmms/` endpoints are publicly reachable, like the rest of the Matrix endpoints. Pairing codes are single-use, expire after 10 minutes and are rate-limited; after pairing, the app authenticates with a per-device, revocable bearer token.

When no bridge URL is given, the app discovers it from the `chat.mdad.bridge.sms_mms` key that the playbook adds to `/.well-known/matrix/client` while the pairing API is enabled. To point the app at a different public URL (e.g. when the bridge is hosted elsewhere), override it in your `vars.yml` file:

```yaml
matrix_bridge_mautrix_android_sms_mms_pairing_public_url: https://bridge.example.com
```

To stop serving and exposing the pairing API, set `matrix_bridge_mautrix_android_sms_mms_pairing_enabled: false`.

If your reverse proxy is not the playbook-managed Traefik, bind the container's pairing port to the host's loopback interface and forward `/_smsmms/` to it:

```yaml
matrix_bridge_mautrix_android_sms_mms_container_http_host_bind_ports:
  - "127.0.0.1:29322:29322"
```

## Serving large files as links (CDN)

Carriers impose tight limits on MMS attachments. Instead of pushing an oversized file through MMS, the app can encrypt it on the phone and upload the ciphertext to the bridge, which serves it as a link such as `https://cdn.example.com/f/<id>#<key>`. The decryption key lives only in the URL fragment (`#…`) and never reaches the bridge.

The feature follows the playbook's provisioning model: it is enabled automatically while a non-empty CDN hostname is configured. The default hostname is `cdn.example.com` (`cdn.` followed by your base domain), so the service is on out of the box. To use it:

1. Create an `A`/`AAAA` DNS record for the CDN hostname (`cdn.example.com` by default).
2. Make sure your certificate setup covers the name. A wildcard certificate for the base domain covers `cdn.example.com`; otherwise Traefik's `certResolver` must be able to issue a certificate for the exact hostname.

To turn the service off (and stop the reverse proxy from exposing it), blank the domain in your `vars.yml` file:

```yaml
matrix_bridge_mautrix_android_sms_mms_cdn_domain: ''
```

To serve it from a different hostname, override the same variable. This is a fully custom CDN domain: it is used as-is for the reverse-proxy route and for the links the bridge advertises, without being derived from (or needing to be a subdomain of) your homeserver name:

```yaml
matrix_bridge_mautrix_android_sms_mms_cdn_domain: files.example.net
```

For backwards compatibility, the older `matrix_bridge_mautrix_android_sms_mms_cdn_enabled` variable is still honored: when set explicitly, it overrides the automatic domain-driven behavior.

The reverse proxy serves the CDN hostname on the same container port as the pairing API (`matrix_bridge_mautrix_android_sms_mms_pairing_port`, `29322` by default), so no extra port, volume or systemd service is needed. Ciphertext lives under `/data/cdn` inside the container, on the existing bridge data volume.

The behavior can be tuned with these variables:

- `matrix_bridge_mautrix_android_sms_mms_cdn_domain` — public hostname the reverse proxy serves the service on (default: `cdn.example.com`; set to empty to disable the service)
- `matrix_bridge_mautrix_android_sms_mms_cdn_data_dir` — ciphertext and metadata directory inside the container (default: `/data/cdn`)
- `matrix_bridge_mautrix_android_sms_mms_cdn_max_file_bytes` — maximum size of one encrypted upload, in bytes (default: `536870912`, 512 MiB)
- `matrix_bridge_mautrix_android_sms_mms_cdn_per_user_cap_bytes` — stored ciphertext per user, in bytes; the oldest files are pruned first once the cap is exceeded (default: `10737418240`, 10 GiB)
- `matrix_bridge_mautrix_android_sms_mms_cdn_default_expiry` — link lifetime used when an upload does not request one (default: `30d`)
- `matrix_bridge_mautrix_android_sms_mms_cdn_max_expiry` — longest lifetime an upload may request, hard-capped at one year (default: `365d`)
- `matrix_bridge_mautrix_android_sms_mms_cdn_gc_interval` — how often expired files are cleaned up (default: `1h`)
- `matrix_bridge_mautrix_android_sms_mms_cdn_upload_token` — optional static bearer token for trusted clients that cannot hold a paired-device token (for example the jmpchat bridge sending its outbound attachments through this service); set together with `cdn_upload_token_owner` (default: empty, disabled)
- `matrix_bridge_mautrix_android_sms_mms_cdn_upload_token_owner` — Matrix user id that owns uploads made with `cdn_upload_token`

The app only offers URL uploads when the bridge advertises the service (`cdn_base_url`) in its pairing responses, which happens exactly when the feature is enabled. Files can be managed from the management room with `!sms cdn list`, `!sms cdn revoke <id>`, `!sms cdn stats` and `!sms cdn gc` (`stats` and `gc` are admin-only).

When you use your own reverse proxy instead of the playbook-managed Traefik, forward the CDN hostname to the bridge's pairing port too.

The [project's `docs/cdn-file-service.md`](https://github.com/microchipster/mautrix-android-sms-mms/src/branch/main/docs/cdn-file-service.md) documents the security model and the app-side contract (AES-256-GCM, keys never stored server-side, quota and expiry behavior).

## Anti-spam (optional)

The bridge can run a local anti-spam gate on incoming messages — no external service ever sees numbers or content. It ships disabled. Evaluate it in dry-run mode first: decisions are reported in the management room, but nothing is labeled or dropped.

Add this configuration (adjust to taste):

```yaml
matrix_bridge_mautrix_android_sms_mms_network_configuration_extension_yaml: |
  anti_spam:
    enabled: true
    dry_run: true
    # unknown_sender_policy: bridge  # bridge (default) = deliver + label + notice; label = silent label; hold = drop messages from unknown senders
    # threshold: 10
    # allow_labels: [family, service]
```

Then check the results from the bridge management room:

- `spam status` — effective configuration and counters
- `spam recent [N]` — the most recent decisions (message ids only)

Once the signal looks right, set `dry_run: false` and pick an `unknown_sender_policy`. The bridge ships full documentation in its `docs/anti-spam.md` (patterns with weights, URL analysis lists, duplicate detection, first-contact rate limits, per-contact overrides) and the design rationale in `docs/anti-spam-research.md`.

## Troubleshooting

As with all other services, you can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu matrix-mautrix-android-sms-mms`.

If logins fail, verify that the bridge can reach the agent's `/v1/health` endpoint and that the token is correct. See the [agent's troubleshooting section](configuring-playbook-android-sms-mms-agent.md#troubleshooting).

### Increase logging verbosity

The default logging level for this component is `warn`. If you want to increase the verbosity, add the following configuration to your `vars.yml` file and re-run the playbook:

```yaml
# Valid values: fatal, error, warn, info, debug, trace
matrix_bridge_mautrix_android_sms_mms_logging_level: 'debug'
```
