<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 MDAD project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Mautrix jmp.chat bridging (optional)

<sup>Refer the common guide for configuring mautrix bridges: [Setting up a Generic Mautrix Bridge](configuring-playbook-bridge-mautrix-bridges.md)</sup>

The playbook can install and configure [mautrix-jmpchat](https://github.com/microchipster/mautrix-jmpchat) for you.

It is a bridge between Matrix and [JMP](https://jmp.chat/) phone numbers, which work over XMPP through the [Cheogram](https://cheogram.com/) network. The bridge connects your JMP/Cheogram XMPP account to Matrix, so that phone-number chats, contacts and group chats can be used from Matrix rooms.

See the project's [documentation](https://github.com/microchipster/mautrix-jmpchat) to learn what it does and why it might be useful to you.

## Prerequisites

To use this bridge, you need a [JMP](https://jmp.chat/) phone number (or another Cheogram-hosted XMPP account). The bridge logs in to the XMPP side of your account using your JID and password.

### Enable Appservice Double Puppet (optional)

If you want to set up [Double Puppeting](https://docs.mau.fi/bridges/general/double-puppeting.html) (hint: you most likely do) for this bridge automatically, you need to have enabled [Appservice Double Puppet](configuring-playbook-appservice-double-puppet.md) service for this playbook.

See [this section](configuring-playbook-bridge-mautrix-bridges.md#set-up-double-puppeting-optional) on the [common guide for configuring mautrix bridges](configuring-playbook-bridge-mautrix-bridges.md) for details about setting up Double Puppeting.

## Adjusting the playbook configuration

To enable the bridge, add the following configuration to your `inventory/host_vars/matrix.example.com/vars.yml` file:

```yaml
matrix_bridge_mautrix_jmpchat_enabled: true
```

### Extending the configuration

There are some additional things you may wish to configure about the bridge.

See [this section](configuring-playbook-bridge-mautrix-bridges.md#extending-the-configuration) on the [common guide for configuring mautrix bridges](configuring-playbook-bridge-mautrix-bridges.md) for details about variables that you can customize and the bridge's default configuration, including [bridge permissions](configuring-playbook-bridge-mautrix-bridges.md#configure-bridge-permissions-optional), [encryption support](configuring-playbook-bridge-mautrix-bridges.md#enable-encryption-optional), [relay mode](configuring-playbook-bridge-mautrix-bridges.md#enable-relay-mode-optional), [bot's username](configuring-playbook-bridge-mautrix-bridges.md#set-the-bots-username-optional), etc.

### Calling (optional)

The bridge can register one of your JMP numbers as a SIP client and bridge calls between the phone network and MatrixRTC ([Element Call](configuring-playbook-element-call.md)), so incoming and outgoing calls ring in Matrix.

Calling support always ships in the standard bridge image and is enabled bridge-wide by default — set `calls.enabled: false` below to disable the SIP runtime entirely.

**Opting in, per account:** run `sip-login` in the bridge bot's management room (your direct chat with the bot, not a chat room) and answer the prompts with the SIP credentials of the number (see the SIP credentials bullet below). That is the whole opt-in — from then on, calls to and from that JMP number ring in the number's own chat room. While the SIP account is registered, the JMP app on your phone no longer receives the calls (the bridge answers them instead); `sip-logout` gives them back to the app. `sip-status` shows whether the account is set up and currently registered.

Calling needs some extra infrastructure:

- No special build or build argument: the bridge image always includes the calling support (there is no reduced variant).
- SIP credentials for your JMP number: message `reset sip account` to the `cheogram.com` bot from the Jabber (XMPP) account of that number. The bot replies with the SIP username, password, server and transport. There is no separate SIP provider to sign up for — the bridge registers to JMP's SIP service with these credentials.
- A working MatrixRTC stack: [LiveKit](configuring-playbook-livekit-server.md) and the [LiveKit JWT service](configuring-playbook-livekit-jwt-service.md), as described in [MatrixRTC](configuring-playbook-matrix-rtc.md). The LiveKit JWT service must accept bridge ghost users (that is, `LIVEKIT_FULL_ACCESS_HOMESERVERS` must include your homeserver).
- [Appservice Double Puppet](#enable-appservice-double-puppet-optional) is recommended, so read receipts can stop ringing on your other devices.
- Calls ring in the phone number's own room, encrypted or not. In an encrypted room the bridge joins the call as its own call device (matching the `SIPCALL` device in its call membership), distributes SFrame call keys to the room's other devices over encrypted to-device messages and encrypts its media with them. This needs Synapse to forward to-device messages to appservices, which the playbook enables automatically when the bridge is enabled.
- If you prefer the legacy behavior, set `room_mode: call-room` below to keep using a dedicated unencrypted *call room*, or turn call encryption off with `e2ee: off` (encrypted rooms then fall back to the call room).
- Every incoming call also posts an urgent room message next to the ring ("📞 Incoming call from … — open this room to answer.") that mentions you. Clients that don't render call notifications still get a push and an unread there, and the message stays marked read once you answer.
- Calls that never connect (missed, busy, rejected, failed) are reported to the management room, so a call never ends silently.
- If call notification sounds are too quiet on a client, a per-user homeserver push rule giving that message a distinct sound is the next lever — the bridge cannot set push rules for your account.

Each account opts in from its own management room after the playbook run; to tune calling (or to disable it entirely), adjust this configuration in your `inventory/host_vars/matrix.example.com/vars.yml` file:

```yaml
matrix_bridge_mautrix_jmpchat_network_configuration_extension_yaml: |
  calls:
    # enabled: false   # uncomment to disable the SIP runtime entirely (accounts cannot opt in then)
    # room_mode: portal   # portal (default): calls ring in the number's own room, encrypted or not
    #                     # call-room: legacy — use the dedicated unencrypted call room instead
    # e2ee: auto          # auto (default): distribute SFrame call keys and encrypt media in encrypted rooms
    #                     # off: never encrypt; encrypted rooms use the call room (or refuse to ring)
    # pickle_key: secret  # optional: protects the bridge's call-device encryption keys in the database
    #                     # (empty derives one from the appservice token)
    # On an incoming call, if the portal room is encrypted and has no call room yet,
    # create one automatically instead of refusing to ring (only relevant with room_mode: call-room,
    # or with e2ee: off for encrypted rooms).
    # auto_create_call_room: true
    # livekit_service_url: https://livekit-jwt.example.com  # optional: skip well-known discovery
    # external_ip: 203.0.113.10                             # optional: advertise this IP in SIP/SDP
    # stun_server: stun.example.org                         # optional
    # ring_timeout_seconds: 25  # how long an incoming call rings in Matrix; keep it below ~30 seconds,
    #                           # because the phone network stops waiting for later answers
```

If you want to receive SIP/RTP traffic directly (instead of relying on the bridge's NAT traversal), publish the SIP and RTP ports:

```yaml
matrix_bridge_mautrix_jmpchat_container_extra_arguments: [
  '--publish=5060-5069:5060-5069/udp',
  '--publish=5060-5069:5060-5069/tcp',
  '--publish=40000-40100:40000-40100/udp',
]
```

After re-running the playbook, each account registers its SIP credentials in its own management room:

- `sip-login` — walks you through the SIP credentials (registrar, username, password, your phone number). Run it in your management room (the DM with the bridge bot) — answers typed in a chat room would be sent to the contact.
- `sip-status` — shows whether calls are enabled and the registration state.
- `call <number>` / `hangup` / `dtmf <digits>` — place calls and manage them from a room.
- `call-room <number>` — create the dedicated unencrypted call room for a number (legacy; used when `room_mode` is `call-room`, or when `e2ee: off` and the number's room is encrypted). Run `call-room` to list the linked call rooms, or `call-room --remove <number>` to unlink one.

### Sending attachments through a file service (optional)

By default, media you send from Matrix is uploaded to Cheogram's own XMPP file service (XEP-0363) and the recipient's Jabber client downloads it from there. If you run the [bridge file service](configuring-playbook-bridge-mautrix-android-sms-mms.md) (the sms-mms CDN), you can send outgoing attachments through it instead: the bridge encrypts the file locally, uploads only ciphertext and puts a link in the message. The decryption key stays in the link's URL fragment, so the file service never sees it.

Configure the bridge (the token pair lives on the host that runs the file service, see the sms-mms page):

```yaml
matrix_bridge_mautrix_jmpchat_network_configuration_extension_yaml: |
  cdn:
    base_url: https://cdn.example.com
    upload_token: SECRET
```

The bridge downloads the Matrix media, encrypts it with a fresh AES-256-GCM key, uploads it to `<base_url>/f` with the token and sends the resulting `https://…/f/<id>#<key>` link as the attachment (OOB data plus a plain link in the body, so both XMPP clients and Matrix rooms can render it). If the upload fails, the bridge falls back to Cheogram's file service automatically.

Receiving attachments from the XMPP side is unaffected: incoming files are downloaded from Cheogram's service and re-uploaded to your Matrix media repository as usual.

## Installing

After configuring the playbook, run it with [playbook tags](playbook-tags.md) as below:

<!-- NOTE: let this conservative command run (instead of install-all) to make it clear that failure of the command means something is clearly broken. -->
```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

The shortcut commands with the [`just` program](just.md) are also available: `just install-all` or `just setup-all`

`just install-all` is useful for maintaining your setup quickly ([2x-5x faster](../CHANGELOG.md#2x-5x-performance-improvements-in-playbook-runtime) than `just setup-all`) when its components remain unchanged. If you adjust your `vars.yml` to remove other components, you'd need to run `just setup-all`, or these components will still remain installed. Note these shortcuts run the `ensure-matrix-users-created` tag too.

## Usage

To use the bridge, you need to start a chat with `@jmpchatbot:example.com` (where `example.com` is your base domain, not the `matrix.` domain).

To log in to your JMP/Cheogram account, run `login <jid>` in the management room. The username is your XMPP JID (for example, `user@cheogram.com`) and the password is your JMP/Cheogram account password. The bridge verifies the credentials by making a real XMPP connection during login.

After logging in, the bridge connects to your account and creates its per-user space and management room structure. You can then run commands like `sync`, `create` and `list-contacts` to discover your chats or create new ones, and talk in the portal rooms that the bridge creates.

Note that Matrix rooms you create on your own are not automatically bridged to JMP.

## Contact names from a CardDAV address book

The bridge can import contact names (and photos) from a CardDAV address book, adding the contacts to your Jabber roster and applying their names (and photos) to auto-named rooms. In the management room, run:

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

## Troubleshooting

As with all other services, you can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu matrix-mautrix-jmpchat`.

### Increase logging verbosity

The default logging level for this component is `warn`. If you want to increase the verbosity, add the following configuration to your `vars.yml` file and re-run the playbook:

```yaml
# Valid values: fatal, error, warn, info, debug, trace
matrix_bridge_mautrix_jmpchat_logging_level: 'debug'
```
