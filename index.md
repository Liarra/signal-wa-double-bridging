---
title: "How to unite Signal and WhatsApp groups using Matrix"
---

Written by Nina Buchina in collaboration with
[James Collier](https://james.thecolliers.xyz/).

> **Last reviewed: August 2026.** This guide assumes current Go/bridgev2
> releases of [mautrix-signal](https://github.com/mautrix/signal) and
> [mautrix-whatsapp](https://github.com/mautrix/whatsapp). In particular, the
> supported [`bridge` command](https://docs.mau.fi/bridges/general/relay-mode.html#bridging-existing-rooms)
> for attaching an existing remote chat requires bridge version 26.04 or newer.
> Older installations use different commands and
> database schemas; upgrade them instead of applying the SQL instructions from
> old versions of this article.

* TOC
{:toc}

## Introduction

Our friends and family are spread over different chat services. Signal users
cannot normally participate in WhatsApp groups, and WhatsApp users cannot
normally participate in Signal groups. This guide describes how to use a
self-hosted Matrix room as the common transport between one group on each
network.

[Matrix](https://matrix.org) is an open protocol for decentralized
communication. A Matrix room can be linked to a chat on another network using a
bridge. By linking the same Matrix room to both a Signal group and a WhatsApp
group, messages can travel between the two groups.

This is an unusual topology, not a formally integrated Signal/WhatsApp product.
Test it with disposable groups before trusting it with an established community.

## Intended audience and scope

This guide assumes that you are an experienced Linux user with some affinity for
self-hosting and system administration. It is not a general installation guide
for a Matrix homeserver or mautrix bridges; the upstream projects already
maintain those.

The guide covers:

* joining one Signal group and one WhatsApp group through a single Matrix room;
* using relay accounts for people who do not have accounts on both remote
  networks;
* attaching existing remote groups without editing the bridge database;
* recovering and diagnosing the arrangement with current bridge commands.

## How it works

Suppose we have one Signal group and one WhatsApp group. A Matrix room sits in
the middle, and each mautrix bridge links its remote group to that same room.

![A cartoon representation of bridged chat rooms. The Matrix room is in the middle connecting WhatsApp and Signal.](https://i.imgur.com/4q1urLH.png)

There is an unavoidable identity problem. A WhatsApp user who has no Signal
account cannot be the sender of a Signal message, and vice versa. Each remote
group therefore needs a service account that sends messages on behalf of people
who are absent from that network. Mautrix calls this
[**relay mode**](https://docs.mau.fi/bridges/general/relay-mode.html).

You need two remote relay identities:

* a Signal account that is a member of the Signal group;
* a WhatsApp account that is a member of the WhatsApp group.

Those two remote accounts may be managed by one dedicated Matrix account.
Current mautrix documentation explicitly permits
[sharing a dedicated Matrix relay account](https://docs.mau.fi/bridges/general/relay-mode.html#shared-relays)
between different bridges. Separate Matrix users are still possible if you want
administrative isolation, but they are not required.

### Security boundary

Signal and WhatsApp still encrypt traffic on their respective networks, and the
Matrix room may use end-to-bridge encryption. However, this arrangement is
**not** end-to-end encrypted from a Signal participant to a WhatsApp
participant. The bridges must decrypt each incoming message and submit it to the
other network. The homeserver and bridge operators, bridge databases, logs,
backups, and remote relay accounts are therefore part of the trust boundary.

Every participant should also understand that messages and attachments are
copied to a second service with a different membership list, retention policy,
and operator.

## Prerequisites

1. **A reachable Matrix homeserver.** Follow your deployment project's actual
   requirements for DNS, HTTPS, ports, and federation. A dedicated public IPv4
   address is not an intrinsic requirement of the bridges.
2. **Current mautrix-signal and mautrix-whatsapp installations.** Version 26.04
   or newer is required for the `bridge` workflow used below.
3. **Your normal Signal and WhatsApp accounts.** Logging your own Matrix user
   into both bridges is useful for administration and for discovering chat IDs.
4. **Two remote relay accounts.** Signal and WhatsApp registration still require
   phone numbers. The same spare number and physical phone can normally host
   both accounts, but each service has its own account and linked-device
   lifecycle.
5. **One dedicated Matrix relay user**, for example `@bridge-relay:YOUR_DOMAIN`.
   This user will own both bridge logins.
6. **A real primary phone is strongly recommended.** WhatsApp warns that
   emulators, VoIP numbers, new accounts, and suspicious outbound activity all
   increase ban risk.

## 0. Install and configure the bridges

Install a Matrix homeserver and both bridges. One maintained option is
[matrix-docker-ansible-deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy):

* [mautrix-signal playbook documentation](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-signal.md)
* [mautrix-whatsapp playbook documentation](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-whatsapp.md)

If you install manually, follow the current
[mautrix bridge documentation](https://docs.mau.fi/bridges/). Do not copy old
sample configuration keys from this article: bridgev2 changed the configuration
layout.

For both bridges:

1. enable relay mode under `bridge.relay.enabled`;
2. give the human administrators the `admin` permission and intended room users
   at least the `relay` permission;
3. decide whether only bridge admins may select relays. `admin_only` is `true`
   by default;
4. allow each bridge bot to join the room and give it enough power to write
   bridge state, change room settings, and invite users.

The bridges can operate in encrypted Matrix rooms when
[end-to-bridge encryption](https://docs.mau.fi/bridges/general/end-to-bridge-encryption.html)
is configured correctly. Current release binaries and Docker images include
encryption support. Set `encryption.allow: true` on both bridges; if the
homeserver forces room encryption, also set `encryption.default: true`. With
next-generation Matrix authentication, review the current `msc4190`
requirements.

Encryption adds more key-sharing failure modes to an already unconventional
topology. Start with test groups, confirm delivery in both directions, restart
both bridges, and confirm it again before adding real users. Disabling
Matrix-room encryption remains a valid operational trade-off, but it is no
longer a technical requirement.

## 1. Log your Matrix account into both bridges

Open private chats with the bridge bots, normally:

* `@signalbot:YOUR_DOMAIN`
* `@whatsappbot:YOUR_DOMAIN`

For Signal, follow the current
[authentication instructions](https://docs.mau.fi/bridges/go/signal/authentication.html)
and send:

```text
login
```

Then scan the QR code from **Linked Devices** in the primary Signal app. If
history transfer is enabled, Signal may offer to transfer recent history.
Primary-device registration through mautrix-signal is no longer supported.

For WhatsApp, follow the current
[authentication instructions](https://docs.mau.fi/bridges/go/whatsapp/authentication.html)
and send either:

```text
login qr
```

or:

```text
login phone
```

Follow the bot's QR, pairing-code, and possible passkey-verification
instructions. WhatsApp normally creates portals after login and can backfill
recent messages. Signal creates or backfills portals after history transfer or
as new messages arrive.

## 2. Attach both remote groups to one Matrix room

The safest modern approach is to attach existing remote groups using the
supported `bridge` command. No direct database editing is necessary or
acceptable.

### Choose the target Matrix room

This example keeps the existing Signal portal as the common room and attaches an
existing WhatsApp group to it. The opposite direction works too: swap the
command prefixes and use the WhatsApp portal as the target.

1. Open the Signal group in the Signal app and send a message if its Matrix
   portal does not exist yet.
2. Open the resulting Signal portal in Matrix.
3. Make sure the room has a name. Signal group names are required and limited to
   32 characters if you later create a Signal group from a Matrix room.
4. Invite the WhatsApp bridge bot to the Signal portal.
5. Give both bridge bots sufficient power. As a room or bridge administrator,
   the following bridge-specific helpers are useful:

   ```text
   !signal set-pl 100
   !wa set-pl 100
   ```

### Find the WhatsApp chat ID

Open the existing Matrix portal for the WhatsApp group and run:

```text
!wa id
```

Save the returned remote chat ID. If there is no portal yet, send a message in
the WhatsApp group and wait for the bridge to discover it.

### Plumb the WhatsApp group into the target room

Return to the Signal portal and run:

```text
!wa bridge <WHATSAPP_CHAT_ID>
```

If you have multiple WhatsApp logins, make the login explicit:

```text
!wa bridge <LOGIN_ID> <WHATSAPP_CHAT_ID>
```

Use `!wa list-logins` in the bot chat to obtain login IDs.

If the WhatsApp group already has another Matrix portal, the bridge refuses to
replace it. After verifying both room and chat IDs, run:

```text
!wa bridge --overwrite <WHATSAPP_CHAT_ID>
```

`--overwrite` is destructive on the Matrix side: it tombstones and deletes the
old WhatsApp portal room before assigning the remote chat to the target room. It
does not delete the WhatsApp group. Preserve anything you need from the old
Matrix room first.

The target Matrix room is now linked to both remote groups. Send test messages
from Matrix, Signal, and WhatsApp and verify exactly one copy arrives at each
destination.

![Screenshots of the same double-bridged group displayed in WhatsApp, Element, and Signal.](https://i.imgur.com/Ld2Ot0X.png)

Old Matrix events are not moved from one portal room to another. Remote-history
backfill is a separate feature and depends on bridge configuration and what the
remote network makes available.

### Creating a new remote group instead

Both bridges expose `create-group`; `create` remains an alias for compatibility:

```text
!wa create-group
!signal create-group
```

The command creates a remote group for the current Matrix room and imports
eligible room metadata and participants. There are two important asymmetries:

* WhatsApp requires at least one WhatsApp-resolvable participant in addition to
  the creator. Signal ghosts and the Signal bridge bot do not count.
* Signal permits a creator-only group, but requires a non-empty name of at most
  32 characters.

Consequently, the old recipe—invite only the WhatsApp bot to a Signal portal and
run `!wa create`—may now fail with no eligible WhatsApp participant. Bridging an
existing WhatsApp group is usually cleaner.

## 3. Create and connect the relay identities

Create the Signal and WhatsApp relay accounts on their primary phone. Add the
Signal relay to the Signal group and the WhatsApp relay to the WhatsApp group
using the native apps.

Create one ordinary Matrix account such as `@bridge-relay:YOUR_DOMAIN`. Log into
Matrix as that user and authenticate it to both bridge bots:

* send `login` to the Signal bot and link the Signal relay account;
* send `login qr` or `login phone` to the WhatsApp bot and link the WhatsApp
  relay account.

Do not confuse the dedicated Matrix user with bridge-managed ghost users. The
Matrix account owns the logins; the two remote accounts actually send the
relayed messages.

Invite the dedicated Matrix relay user to the double-bridged room. Then obtain
each remote login ID with:

```text
!signal list-logins
!wa list-logins
```

For unattended operation, add the relevant login ID to each bridge's
`bridge.relay.default_relays` list. You may also add those IDs to
`bridge.portal_create_filter.always_deny_from_login` so the relay logins do not
create unwanted standalone portal rooms.

## 4. Enable relay mode in the common room

With the relay identities present in their respective remote groups, set each
login as the relay. A bridge administrator can do this from the common room:

```text
!signal set-relay <SIGNAL_LOGIN_ID>
!wa set-relay <WHATSAPP_LOGIN_ID>
```

If `admin_only` is `false`, the dedicated Matrix relay user may instead run the
command without an argument under its own account. With the default
`admin_only: true`, an ordinary relay user cannot necessarily appoint itself.

Test the full path:

1. send from a Signal-only participant and confirm delivery to WhatsApp;
2. send from a WhatsApp-only participant and confirm delivery to Signal;
3. send from Matrix as a user who is not logged into either bridge;
4. test text, an image, a reply, an edit, and a deletion separately.

Do not assume every event type has perfect semantic parity. In particular,
reactions from relayed users are not bridged because the remote relay account
cannot preserve both sender identity and multiple identical reactions.

## 5. Add participants and operate the room

Populate both remote groups through their native apps. Keep the membership lists
and administrator policies aligned manually; neither remote service knows that
the other group exists.

Tell participants:

* which content is copied to the other network;
* that relayed messages are attributed using a display-name prefix rather than
  the sender's native identity;
* who operates the homeserver, bridges, and relay accounts;
* how long logs, database records, media, and backups are retained;
* that reactions and some rich-message features may be lost or simplified.

## Recovery and troubleshooting

### A remote group is attached to the wrong Matrix room

Do not edit SQL. Current bridgev2 uses a different schema from old bridge
releases, and manual writes can corrupt unrelated portals.

1. In the current portal, run `!signal id` or `!wa id` and save the remote chat
   ID.
2. Invite the relevant bridge bot into the intended Matrix room.
3. Ensure the bot has sufficient room power.
4. In the intended room, run the appropriate command:

   ```text
   !signal bridge --overwrite <SIGNAL_CHAT_ID>
   !wa bridge --overwrite <WHATSAPP_CHAT_ID>
   ```

5. Read the warning carefully: overwrite deletes the old Matrix portal room.

The old article edited a `portals.relay_user_id` field that current bridgev2
does not have. Current bridgev2 uses a `portal` table keyed by bridge ID, remote
chat ID, and receiver, with the Matrix room in `mxid` and relay selection in
`relay_login_id`. Besides targeting an obsolete schema, the old example's
`WHERE mxid = mxid` condition would match every row whose `mxid` was not null.
It was a loaded gun pointed at the entire portal table.

### A portal exists but its metadata or membership is stale

Run the relevant portal resync in the affected room:

```text
!signal sync-portal
!wa sync-portal
```

Then inspect the affected bridge service's logs. Restarting the Matrix
homeserver is not a useful first response to a connector-specific failure.

### WhatsApp disconnected the linked device

The bridge uses WhatsApp's linked-device interface. If the primary phone is
offline for more than about two weeks, linked devices may be disconnected;
[mautrix-whatsapp warns](https://docs.mau.fi/bridges/go/whatsapp/authentication.html)
after roughly 12 days without phone data.

Bring the primary phone online. If the login is no longer valid, use `logout` in
the WhatsApp bot chat and authenticate again with `login qr` or `login phone`.
Do not assume the old bare `login` command selects the correct method.

### Signal disconnected the linked device

[Signal requires](https://support.signal.org/hc/en-us/articles/360007320551-Linked-Devices)
the primary phone to come online at least once every 30 days and unlinks linked
devices after 45 days of inactivity. If relinking is required, send `login` to
the Signal bot and scan the new QR code from the primary app. The old
`link matrix` command and signald-specific recovery steps no longer apply;
mautrix-signal was rewritten in Go and stopped using signald in 2024.

### Encrypted-room messages fail to bridge

Confirm that encryption is allowed on **both** bridges and that both bridge
devices receive the room keys. Review `self_sign`, client device-verification
policy, and `msc4190` when applicable. Test from the same client in an
unencrypted disposable room to distinguish a crypto failure from a
remote-network failure.

### Messages loop, duplicate, or bridge notices leak

Temporarily stop one relay path while diagnosing:

```text
!signal unset-relay
```

or:

```text
!wa unset-relay
```

Verify that there is only one Signal portal and one WhatsApp portal attached to
the Matrix room, and that the two native relay accounts are members only where
intended. Inspect the bridge logs and the actual error. Do not blindly unlink
accounts or restart Synapse because an old troubleshooting section mentioned a
Java or signald exception; those stacks cannot come from the current Go bridge.

## Operational maintenance

This is infrastructure, not a one-time incantation.

* Keep the homeserver and both bridges current, and read both bridge changelogs
  before major upgrades.
* Monitor the primary phones' connectivity and linked-device state.
* Back up bridge configuration and databases, but test restoration rather than
  merely accumulating hopeful archives.
* Re-run the three-direction delivery test after upgrades, room encryption
  changes, membership changes, or relinking either account.
* Use each bot's `help` output as the authority for commands supported by the
  version you actually run.

## Conclusion

Matrix can still join Signal and WhatsApp groups effectively, but the mechanism
is less primitive than it was when this article was first published. Existing
groups can now be attached with supported bridge commands, one Matrix account
can own both relay logins, and current Go bridges can operate in encrypted
Matrix rooms.

The bargain has not changed: you gain interoperability by operating trusted
middleware that can read and retransmit the conversation. If that trust model is
acceptable, the result is a practical escape hatch from two walled gardens. If
it is not, no amount of YAML will alchemize the arrangement into end-to-end
encryption.
