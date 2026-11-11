<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Text A Day (optional)

Text A Day publishes scheduled literary text, quotes, and vocabulary to Matrix
rooms. It also provides management commands through encrypted administrator DMs.

## Configure an administrator

Set at least one real Matrix user in `vars.yml`:

```yaml
matrix_bot_text_a_day_public_admin_user_ids:
  - "@alice:example.com"
```

This setting has two roles:

- Listed users receive power level 100 in public Text A Day rooms.
- Listed human users receive an invite to the shared space and an encrypted DM
  from the bot. They can manage it from that DM.

The bot account itself is included by default so it retains ownership of public
rooms. It is not sent an onboarding DM.

Deploy or redeploy after changing this setting. Bot MXID uses `textaday` by
default: `@textaday:example.com`. Replace `example.com` with your Matrix server
name.

## First use

After deployment, accept the shared-space invite and the `Text-a-day` DM from
`@textaday:example.com`. This DM is the administrator management room. Send
this in the DM:

```text
help
```

Use `status` to inspect configured streams and `run <stream-id>` to trigger one
immediately. Public stream IDs are normally:

- `text-a-day`
- `quote-of-the-day`
- `word-of-the-day`

The shared public space is displayed as `Text a day`. Its stable default alias is
`#space-textaday-daily-streams:example.com`. Stream aliases are normally
`#text-a-day:example.com`, `#quote-of-the-day:example.com`, and
`#word-of-the-day:example.com`.

Configured public stream rooms are created or reconciled during bot startup and
added to this space. Existing rooms with these aliases are reused.

Stream rooms use end-to-end encryption and default to `shared` history
visibility. Users who join a stream can read messages sent before they joined;
users who have not joined cannot read stream history. Room administrators can
make history stricter with Matrix room settings: `invited` exposes history from
invitation, while `joined` exposes history from join. Text A Day preserves these
stricter settings during later restarts.

Historical E2EE messages also require room keys. Fresh deployments retain the
bot's crypto state and share historical keys with users joining later. Messages
sent before this behavior was deployed may remain undecryptable.

To invite yourself or another user to a public stream and the shared space,
send this command in your administrator DM:

```text
join text-a-day @alice:example.com
join quote-of-the-day @alice:example.com
join word-of-the-day @alice:example.com
```

The named `textaday-management` room remains supported as a shared management
room, but creating it is not required for configured administrators.

## Synapse rate limits

If Synapse returns `M_LIMIT_EXCEEDED` for the bot, exempt only the bot account
from rate limiting with Synapse's Admin API:

```bash
curl --header "Authorization: Bearer ADMIN_ACCESS_TOKEN" \
  --request POST \
  https://example.com/_synapse/admin/v1/users/@textaday:example.com/override_ratelimit
```

This changes Synapse behavior immediately and does not require redeploying Text
A Day. Avoid disabling rate limits globally unless all users on the homeserver
are trusted.

## Configuration references

- `roles/custom/matrix-bot-text-a-day/defaults/main.yml` contains role defaults.
- `text-a-day/docs/text-a-day-local.md` documents local, non-Matrix commands.
- Logs are available through `journalctl -fu matrix-bot-text-a-day`.

## AI-compatible API

Text A Day uses OpenAI-compatible chat completions for summaries, Word sentence
enrichment, and optional quote judging. Configure API credentials, model lists,
and endpoint URL lists in `vars.yml`:

```yaml
matrix_bot_text_a_day_ai_api_key: YOUR_AI_API_KEY
matrix_bot_text_a_day_ai_models:
  - YOUR_PRIMARY_MODEL_ID
  - YOUR_FALLBACK_MODEL_ID
matrix_bot_text_a_day_ai_urls:
  - https://your-ai-service.example/v1
```

The URLs are required when AI features are enabled. They can point to any compatible
hosted service or custom deployment. Multiple models or URLs are tried sequentially as fallbacks.
Quote judging can override these settings with `matrix_bot_text_a_day_quote_ai_api_key`,
`matrix_bot_text_a_day_quote_ai_models`, and `matrix_bot_text_a_day_quote_ai_urls`.
