<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 MDAD project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Matrix Invite Portal (optional)

The playbook can install and configure a closed-registration invite portal for you.

The portal is meant for non-technical users. It keeps registration closed on the homeserver, shows a short invite page, and creates accounts server-side by talking directly to Synapse's admin API on the private Docker network.

This is a replacement for the old `matrix-registration` CGI backdoor flow. It is not a public registration endpoint.

The portal keeps the user experience simple and unchanged:

- You send one invite link to one person, or to a whole group.
- Each person opens the same link, picks a username and password, and gets a checklist at the end.
- The portal uses a portal-only admin token for invite lifecycle management.
- The portal uses Synapse's `registration_shared_secret` for account creation.
- A leaked portal token can manage invite links, but it cannot administer Synapse.
- A leaked registration shared secret can create users, but it still cannot administer Synapse.

## Where the secrets come from

There are two secrets to think about:

- `matrix_invite_portal_admin_token` is the portal's own admin token. You generate it yourself and store it in `vars.yml`. It is not read from Synapse.
- `matrix_synapse_registration_shared_secret` is the Synapse registration secret. You also generate it yourself if you want to set it explicitly. In this playbook, it can inherit from `matrix_synapse_macaroon_secret_key`, which is why many setups already have a value for it.

If you are starting from scratch, generate both with a strong random string generator like `pwgen -s 64 1` and paste the results into `vars.yml`. If you already have Synapse configured, keep the existing `matrix_synapse_registration_shared_secret` value and point the portal at it.

## Configure it

Add the following to your `inventory/host_vars/matrix.example.com/vars.yml` file:

```yaml
matrix_invite_portal_enabled: true

# Portal-only secret used for invite creation, listing, rotation, and revocation.
# Generate this yourself, for example with `pwgen -s 64 1`.
matrix_invite_portal_admin_token: "paste-a-portal-only-secret-here"

# Synapse shared secret used by the portal when it creates accounts.
# If you already have Synapse configured, reuse the existing value.
# Otherwise generate a new one, for example with `pwgen -s 64 1`.
matrix_synapse_registration_shared_secret: "paste-a-separate-registration-secret-here"

# Optional: make the default invite link usable by many people.
# Set this to the number of people you expect, or 0 for no limit.
matrix_invite_portal_default_use_limit: 20

# Optional: make invite links long-lived or short-lived.
matrix_invite_portal_default_invite_ttl_hours: 24
```

If you want to override the portal's displayed homeserver URL or onboarding help page, you can still set:

```yaml
matrix_invite_portal_homeserver_url: "https://matrix.example.com"
matrix_invite_portal_info_website_url: "https://example.com/help/matrix"
```

The portal uses the Matrix homeserver hostname by default and serves the portal under `/invite`, so you do not need a new DNS name.

Run the playbook to install the service.

The role's defaults live in `roles/galaxy/matrix-invite-portal/defaults/main.yml` after you run `just roles`. If you need to pin the portal source checkout for its Docker build, set `matrix_invite_portal_container_image_self_build_repo_version` to a branch, tag, or commit SHA. It defaults to `main`.

## Invite lifecycle

The portal exposes a public-facing admin API for invite management. It is reachable over HTTPS from outside the container network, but every request must carry the portal-only token in the `Authorization` header.

Create an invite for a group, where the same link can be used by up to 20 people:

```sh
curl -sS -X POST https://matrix.example.com/admin/invites \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"created_by":"ops","note":"team onboarding","expires_in_hours":24,"use_limit":20}'
```

Create an invite with no use limit if you want one permalink for a broader campaign:

```sh
curl -sS -X POST https://matrix.example.com/admin/invites \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"created_by":"ops","note":"open campaign","expires_in_hours":72,"use_limit":0}'
```

The response includes an `invite_id`, an `invite_url`, and a `token_preview`.

- `invite_url` is the permalink you send to people.
- `invite_id` is the handle you use later to rotate or revoke that permalink.
- `token_preview` is only a short preview for inventory and audit purposes.

List existing invites:

```sh
curl -sS https://matrix.example.com/admin/invites \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" | jq .
```

There is no server-side "valid only" filter yet, but you can filter the returned list locally:

```sh
curl -sS https://matrix.example.com/admin/invites \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" | \
  jq '.[] | select(.revoked_at == null and .expires_at > now and (.use_limit == 0 or .use_count < .use_limit)) | {invite_id, token_preview, use_count, use_limit, expires_at}'
```

That shows the invites that are still usable right now.

Rotate an invite if you want to replace the permalink while keeping the invite record:

```sh
curl -sS -X POST https://matrix.example.com/admin/invites/INVITE_ID/rotate \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason":"link rotated"}'
```

Revoke one invite if you no longer want that permalink to work:

```sh
curl -sS -X POST https://matrix.example.com/admin/invites/INVITE_ID/revoke \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason":"no longer needed"}'
```

Revoke all currently active invites if you want to invalidate every active permalink at once:

```sh
curl -sS -X POST https://matrix.example.com/admin/invites/revoke-all \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason":"rotate everything"}'
```

## Password reset links

The portal also exposes one-time password reset links for existing users.

The reset creation endpoint is part of the same public-facing, bearer-token-gated portal admin API. You can curl it from off-network as long as you present `matrix_invite_portal_admin_token`. The portal then talks to Synapse over the private Docker network when the user submits the new password.

The reset flow uses the portal admin token to create links, and it uses a separate Synapse admin access token when the user actually submits the new password. That keeps invite management and password administration compartmentalized.

Important: `matrix_invite_portal_synapse_admin_access_token` must be an access token for a Synapse server-admin account. A token returned by `/_matrix/client/.../login` is fine only if the logged-in account is actually marked as a Synapse admin.

You can verify the configured token with the portal's admin health check:

```sh
curl -sS https://matrix.example.com/admin/synapse-health \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN"
```

The endpoint returns `200 OK` and `ok` when the token can successfully call Synapse's admin `server_version` endpoint.

The generated reset link is served from the same portal hostname as the invite flow, under `/reset/<token>`.

Add these to your inventory if you want password reset links:

```yaml
matrix_invite_portal_synapse_admin_access_token: "paste-a-synapse-admin-token-here"
matrix_invite_portal_default_password_reset_ttl_hours: 24
```

Create a reset link for a specific Matrix ID:

```sh
curl -sS -X POST https://matrix.example.com/admin/password-resets \
  -H "Authorization: Bearer YOUR_PORTAL_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"@alice:matrix.example.com","created_by":"ops","note":"password support request","expires_in_hours":24}'
```

The response includes a `reset_url`, a `reset_id`, and the target `user_id`.

- `reset_url` is the one-time link you send to the user.
- `reset_id` is the handle you keep for inventory and audit purposes.
- `user_id` is the Matrix account that the link is locked to.

When the user opens the link, they see a short intro page, then a new password form with confirmation and an optional checkbox to log out existing devices. After the reset, they get the same kind of checklist and help links that the invite flow shows.

## What the portal does behind the scenes

The portal creates users by calling Synapse's shared-secret registration API on the private Docker network. You normally do not call this yourself, but this is the exact API flow.

Fetch a nonce:

```sh
curl -sS http://matrix-synapse:8008/_synapse/admin/v1/register
```

The response looks like this:

```json
{"nonce":"thisisanonce"}
```

Create the HMAC and submit the registration request:

```sh
shared_secret='YOUR_SYNAPSE_REGISTRATION_SHARED_SECRET'
nonce='thisisanonce'
username='alice'
password='a-strong-password'
admin='notadmin'

mac=$(printf '%s\0%s\0%s\0%s' "$nonce" "$username" "$password" "$admin" | \
  openssl sha1 -hmac "$shared_secret" | awk '{print $2}')

curl -sS -X POST http://matrix-synapse:8008/_synapse/admin/v1/register \
  -H "Content-Type: application/json" \
  -d "{\"nonce\":\"$nonce\",\"username\":\"$username\",\"password\":\"$password\",\"admin\":false,\"mac\":\"$mac\"}"
```

If you need an administrator account for some other purpose, you can set `admin='admin'` in that raw call. The portal itself does not do that.

## Revoke the shared secret

There is no curl command for revoking `registration_shared_secret`. It is a configuration secret, so revocation means rotating it in the playbook inventory and reapplying the playbook.

For example:

```yaml
matrix_synapse_registration_shared_secret: "a-new-random-secret"
```

Then rerun the playbook with the usual setup command. After that, the old secret stops working for future registrations, while already-issued invite links continue to work until you revoke or expire them separately.

## How to use it

1. Enable the service in your host variables.
2. Run the playbook to install the service.
3. Create an invite with the portal admin API and send the returned `invite_url` to the people you want to onboard.
4. Keep the returned `invite_id` so you can revoke or rotate that permalink later.

By default, the playbook exposes the portal on the homeserver hostname and keeps the invite permalink under `/invite`, so the returned `invite_url` looks like `https://matrix.example.com/invite/<secret>`.

If you want to serve the portal from a different hostname, override `matrix_invite_portal_container_labels_traefik_hostname`.

If you need the admin API to return links that point somewhere else, override the derived base URL explicitly:

```yaml
matrix_invite_portal_public_base_url: https://join.example.com
```

The portal talks to Synapse directly on the private container network, using `http://matrix-synapse:8008/_synapse/admin` by default. If your Synapse container network or hostname differs, override `matrix_invite_portal_synapse_admin_api_base_url`.

The onboarding flow now starts with a short read-this-first page that names the account provider host inline, then shows an explicit `Registering account on <homeserver>` banner on the form page, and ends with a compact checklist after signup. The checklist is intentionally tight enough for a phone screen and includes the username reminder, password reminder, recovery key warning, account-provider host, help link, and client links. The form keeps password mismatch inline on the same view, instead of sending people to a separate error page. After signup, the portal shows the summary immediately and rewrites the browser URL to the GET summary page, so refreshing does not resubmit the form. The completed summary is retained server-side and mirrored into browser-local storage, not cookies, for up to 48 hours, so reopening the invite in the same browser still shows the finished checklist even after the tab is closed or the browser is restarted. The form is blank when you start fresh or click `Try another name`, and the `Try another name` button clears that browser-local memory and opens a clean form. The summary also has a small `Clear data` action if the user wants to wipe the saved summary from both the portal and the browser and go back to the intro:

- iOS, Android, and F-Droid badges for the app stores.
- `Element Web` with a derived URL in the form of `element.<homeserver>`.
- `Element Desktop` linking to the Element desktop client page.
- `Other clients` linking to the generic Matrix client list.
- `matrix_invite_portal_info_website_url` for the help and guide page.

You can and should log into multiple clients. Any Matrix client works.

The final checklist also shows the homeserver host it expects people to sign in to. It is presented as the required account provider. By default, that comes from `matrix_homeserver_url`, but you can override it if you need to show a different value:

```yaml
matrix_invite_portal_homeserver_url: "https://matrix.example.com"
```

## After installation

You can use the usual playbook commands after enabling the service:

```sh
just setup-all
# or
just install-service matrix-invite-portal
```

As with all other services, you can find the logs in systemd-journald by logging in to the server with SSH and running `journalctl -fu matrix-invite-portal`.
