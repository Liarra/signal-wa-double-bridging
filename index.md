---
title: "How to unite Signal and WhatsApp groups using Matrix"

---

Written by Nina Buchina in collaboration with [James Collier](https://james.thecolliers.xyz/)

* TOC
{:toc}

## Introduction

Our friends and family are spread over the globe so communication with them over the internet is especially important to us. But if your friends and family are relatively close, chances are communication with them over the internet is also important to you. Luckily today we have a great variety of apps and services to accomplish this. Unfortunately this is also a problem because many of these services are "walled gardens" that do not follow the open communication infrastructure of the early internet. For better or worse, these services force users to use their app. So very often we end up with 3 or 4 communication apps with non-overlapping groups of friends on each app. 
> Do I use Signal to talk to Amy, or do I use WhatsApp? Oh did she move to Telegram?

This is where [Matrix](https://matrix.org) comes in. Matrix is _"an open network for secure, decentralized communication"_.

This fits the vision of the early internet. Anyone can communicate with anyone, no matter which service they're using. Think e-mail but for instant messaging, voice, and video calls, but better than e-mail because it is _secure by design_.

In this manual we document the steps we followed to self-host a Matrix service and use it to connect group chats in Signal and WhatsApp.

## Our intended audience

We assume that you are an experienced Linux user with some affinity for self-hosting and system administration. We tried to make all the steps in this guide as clear and easy to follow as possible, but these skills will likely be needed when something inevitably goes wrong.

Note that this manual is **not** a guide on how to install Matrix, Synapse, or any Matrix Bridges. For these subjects we refer to to the excellent documentation that already exists.

## What is Matrix

Matrix is an [open standard comunications protocol](https://spec.matrix.org/latest/). It has a reference service implementation that is called [Synapse](https://matrix.org/docs/projects/server/synapse). Although matrix.org hosts a Matrix "homeserver", anyone can run their own Matrix server and users from every Matrix server can chat with users from all other Matrix servers.

You don't even have to run Synapse or the default client [Element](https://element.im). Because the protocol is open and freely implementable there are several existing [servers](https://codeberg.org/yarmo/delightful-matrix#server-implementations) and [clients](https://codeberg.org/yarmo/delightful-matrix#clients) to choose from.

## General idea

Suppose we have two group chats: one is a Signal group, another is a Whatsapp chat. In order to unite them and get the different platform users to talk to one another, we need to have *something* in between that communicates with both platforms, sending those cat pictures back and forth. That *something* is going to be a Matrix "room".

A Matrix room can be linked to a chat on some other platform with help of special software called a [Bridge](https://matrix.org/bridges/). For this exercise, we are going to use a [Whatsapp bridge](https://github.com/mautrix/whatsapp) and a [Signal bridge](https://github.com/mautrix/signal). If we *bridge* our Matrix room to both the Signal group and the Whatsapp group, then this bridged Matrix room will enable communication between the two chats.

![A cartoon representation of bridged chat rooms. The Matrix room is in the middle connecting WhatsApp and Signal.](https://i.imgur.com/4q1urLH.png)

Seems simple, right? Collect messages from both chats into one Matrix room, and let Matrix send the messages from the Signal group to a Whatsapp chat and the other way around. But there is a big caveat.

In both Signal and Whatsapp, every message has a sender. **If a person doesn't have a Signal account, they cannot send a message to any Signal group**. Same story with Whatsapp. So what should happen if a Whatsapp user sends a message into our Whatsapp chat? How would her message show up in a bridged Signal group?

There is no clever answer to this question. You need *some* user, who already has a WhatsApp and Signal account, to be the sender of every message in your group. Therefore, in order to deliver messages from one platform to another, we are going to need a _"service account"_ on each of them. These service accounts will send messages to Signal on behalf of WhatsApp users, and to WhatsApp on behalf of Signal users. They will also need access to the Matrix room in order to read the messages that are not sent from their home platform (WhatsApp or Signal). From now on, we are going to call these service accounts *"Relays"*.

To summarise: we are going to create a Matrix room and bridge it to a Whatsapp chat and to a Signal group. The messages for users who are not present on one of the WhatsApp or Signal will be relayed by two Relay Accounts (one for WhatsApp, one for Signal). The Signal relay will need its own Matrix account and a Signal account. The WhatsApp relay will need its own Matrix account and a WhatsApp account.


## Pre-requisites

1. **A server on which you can install Matrix software.**
This can be a Virtual Private Server (VPS), or your old laptop. The important thing is that it meets the [current networking requirements](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/prerequisites.md), including public DNS and valid HTTPS.

2. **A domain name.** You can use a subdomain of an existing domain name, if you have one.

3. **Accounts on Signal and Whatsapp.** We assume you already have them, since you are reading this article. But just to make things clear - you will need to be registered on both platforms yourself. 

4. **A phone number that is not registered with Signal or Whatsapp.**
Relay Accounts will *relay* messages between Whatsapp and Signal. This means that you'll need to register separate accounts for them - one in Signal and one in Whatsapp. This will require a spare SIM card [^simcard]. 

5. **A spare phone on which you can install Signal [^phoneforsignal] and WhatsApp for the Relay Accounts.** Likewise, for Relay accounts to be able to use Whatsapp and Signal, they need their own device. A physical phone is strongly recommended for the Whatsapp relay account; emulators, VoIP numbers and brand-new accounts increase the risk of Whatsapp banning it.

## Steps

> **Note:** This manual was last checked against current Go/bridgev2 releases in October 2026. 

### 0. Installation
Install a Matrix server and bridges for Signal and Whatsapp[^installmatrix]. The easy way to do this is by using this super helpful Ansible playbook: https://github.com/spantaleev/matrix-docker-ansible-deploy. We went for a default Synapse homeserver and enabled the bridges as described in the current docs for [Signal](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-signal.md) and [Whatsapp](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-whatsapp.md); however, the playbook is highly configurable. Make sure to study the playbook's docs so that you don't miss out on any cool tweaks or features.

If you decide to not use the aforementioned playbook, you'd have to install a Matrix server and bridges separately; follow their documentation for details.

> An important note on bridges configuration: Current Signal and Whatsapp bridges can both work in an encrypted Matrix room when [end-to-bridge encryption](https://docs.mau.fi/bridges/general/end-to-bridge-encryption.html) is configured correctly on both bridges. Test delivery in both directions before adding real users. This setup is not end-to-end encrypted between Signal and Whatsapp: both bridges necessarily decrypt and retransmit the messages, and are therefore part of the trust boundary.

Once you have your Matrix installation working, go ahead and [create a Matrix account for yourself](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/registering-users.md). Then login to your brand new Matrix account using one of the [many matrix clients](https://matrix.org/clients/). We used [Element](https://element.io/), as it's the most stable and feature-rich one. Have a look around, play with some settings, pat yourself on the shoulder, as you're halfway there.

Next, set up Whatsapp and Signal bridges for yourself by chatting with Whatsapp Bridge Bot (`@whatsappbot:YOUR_DOMAIN`) and Signal Bridge Bot (`@signalbot:YOUR_DOMAIN`), respectively.  Both bots accept `help` command, which should get you started. With current bridges, Signal uses `login`, while Whatsapp uses `login qr` or `login phone`. When in doubt, consult the [bots' documentation](https://docs.mau.fi/bridges/index.html). 

If you've done well, you will now have a few moments to enjoy your first milestone - being able to chat with friends from both Signal and Whatsapp using one single interface - Element. 

![A screenshot depicting interface of Element. On the right are icons of the Signal, Whatsapp, and Matrix accounts of the author, plus two accounts for bridge bots.](https://i.imgur.com/tzcSBU3.png)

*A screenshot depicting interface of Element. On the right are icons of the Signal, Whatsapp, and Matrix accounts of the author, all available to chat from one interface. There are also two accounts for bridge bots.*


That's pretty cool already! Now let's get to uniting the two...

### 1. Unite existing group chats on Signal and Whatsapp

Current bridges have a supported `bridge` command for linking an existing Signal or Whatsapp chat to an existing Matrix room.

#### Signal
Locate the Signal group you want to bridge. Send a message to that group from the Signal app. Then, the Signal Bridge Bot should create a Matrix portal room for that group. This should happen automatically if you have correctly set up `mautrix-signal`.

![Screenshots of the same Signal group chat displayed in Signal and Element, side by side.](https://i.imgur.com/pdU6Ha9.png)

*Screenshots of the same Signal group chat displayed in Signal and Element, side by side.*


#### Whatsapp
Locate the Whatsapp group you want to bridge and send a message to it if its Matrix portal room does not exist yet. In that Whatsapp portal room, send `!wa id` and save the remote chat ID returned by the bridge.

Return to the Signal portal room and invite the Whatsapp Bridge Bot (`@whatsappbot:YOUR_DOMAIN`). Make sure it has enough power to manage the portal; a room or bridge administrator can use `!wa set-pl 100`.

The Whatsapp group already has its own Matrix portal, so attaching it to the Signal portal requires the overwrite option:

```
!wa bridge --overwrite <WHATSAPP_CHAT_ID>
```

If you have multiple Whatsapp logins, use `!wa list-logins` to find the correct login ID and include it before the chat ID. Read the warning before confirming: `--overwrite` tombstones and deletes the old **Matrix portal room**, but does not delete the Whatsapp group. Save anything you need from the old Matrix room first.

The Signal portal room is now _"double bridged"_. Try sending messages from Element, Signal and Whatsapp. If you've done well, each message should appear once on both other platforms!

![Screenshots of the same double-bridged Signal group chat displayed in Whatsapp, Element and Signal, side by side.](https://i.imgur.com/Ld2Ot0X.png)

*Screenshots of the same double-bridged Signal group chat displayed in Whatsapp, Element and Signal, side by side.*

#### If you create both chats from scratch
It is better to start on the Whatsapp side. Create and populate the Whatsapp chat first, let its Matrix portal room appear, invite the Signal Bridge Bot to that portal, and make sure that the room has a name no longer than 32 characters. Then send `!signal create` in the Whatsapp portal.

Whatsapp participants are not copied into Signal: after creating the Signal group, populate it separately on the Signal side.

Starting with Signal and then sending `!wa create` may fail: current mautrix-whatsapp requires at least one additional Whatsapp-resolvable participant when it creates a group, and Signal ghosts do not count.

Nice, we are almost there! Now the last bit: enabling relaying of messages for people who lack a Signal or Whatsapp account.


### 2. Create Relay accounts on Singal and Whatsapp

#### Signal and Whatsapp
Create accounts for your Relays on Signal and Whatsapp using your spare simcard. They can share the same phone number and the same device. _Don't forget to save their credentials and pin codes to a safe location!_

#### Matrix
Create a Matrix account for each of the Relays. Depending on your installation, account creation can be performed with [Synapse Admin](https://github.com/Awesome-Technologies/synapse-admin), with help of the [same Ansible script you used to install Matrix](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/registering-users.md), or by [sending a special HTTP request](https://matrix-org.github.io/synapse/latest/admin_api/register_api.html) to your server.

In our setup, we named the Relays `@signal-relay` and `@whatsapp-relay`. You could check that the accounts are created successfully by chatting with them from your own Matrix account. You could have these accounts [ignore](https://discourse.ubuntu.com/t/how-to-manage-your-ignore-list/43641) each other so that error messages aren't relayed in a loop.

#### Bridge
Now that each Relay has a Matrix account and a Signal / Whatsapp account, we need to let our system know that these are in fact related to one another. We achieve this by setting up Whatsapp / Signal bridges for both of the relays.

Remember setting up Signal and Whatsapp bridges for your own account in step 0? Now you will have to do the same for each of the Relays. Use `login` for Signal, and `login qr` or `login phone` for Whatsapp. 

The general sequence is the following:

1. Log in to Relay's Matrix account using Element or other client [^bots-login];
2. Set up a bridge by chatting with Signal Bridge Bot or Whatsapp Bridge Bot (depending on which Relay Account you are setting up);
3. Follow the bot's instructions, but remember to bridge to the **Relay's Whatsapp / Signal account** and not your own;
4. Test that the bridging worked by sending some messages from the Relay Account to your own Signal / Whatsapp account using a Matrix client. If your bridging works, you should also see these messages in your Signal / Whatsapp App.

![Screenshots of interaction between a human and his Whatsapp Relay account.](https://i.imgur.com/skXdRaz.png)

*Screenshots of interaction between a human and his Whatsapp Relay account. Whatsapp Relay account has sent a PM to the user using Element (left screenshot), which led to the user receiving a message from Whatsapp Relay in Whatsapp (screenshot on the right[^screenshot-edited])*


#### Cordially invite the Relays
Once both of your Relay Accounts are set up, it's time to invite them to our bridged room, so that they may do this message-relaying job we hired them for.

Go to your Signal group in the Signal app on your phone / PC and invite the Signal account of the Signal Relay to this group. Do the same for the Whatsapp group chat and the Whatsapp Relay Account.

Now, login to your own Matrix account in Element and open our double-bridged Matrix room that we have created in Step 1. Invite both Relays to this room. **Make sure to invite the original Matrix accounts (`@signal-relay:YOUR_DOMAIN` and `@whatsapp-relay:YOUR_DOMAIN`)**. There should already be "puppet" accounts for both relays that Signal and Whatsapp Bridges have created. Ignore these.

Now we have created two Relay Accounts that can send messsages to Signal and WhatsApp, as well as read and send messages in the bridged Matrix room. The last step is to instruct them to relay messages from Matrix back to their own platforms.

### 3. Enable relaying in your Matrix room

Luckily, both the Signal Bridge Bot and the Whatsapp Bridge Bot come with relaying functionality. Messages sent into the bridged room from Matrix users who are not present in Signal / Whatsapp will be relayed to Signal / Whatsapp. The only thing they need from you is to specify which account on their platform should appear as author of these messages. Luckily, we just created our Relays especially for this purpose!

First of all, make sure that you have enabled the relaying feature in the bot's settings. If you are using Ansible playbook, check out this section of docs [for Signal](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-signal.md) and [for WhatsApp](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-bridge-mautrix-whatsapp.md). Otherwise, revisit the documentation of the bridges.

Once relaying is enabled, log in to Matrix with Element as the Signal Relay and send `!signal list-logins` to find its Signal login ID. Do the same as the Whatsapp Relay with `!wa list-logins`. (You will need those IDs for the next step). Then login as a Matrix room owner (your normal user) and return to our double-bridged Matrix room. In bridged Matrix rooms you can send special messages to issue commands to the bridge bots. For instance, to address the Signal Bridge Bot, start your message with `!signal`. Likewise, to address the Whatsapp Bridge Bot, use `!wa` prefix. 

Current bridge configurations normally have `admin_only: true`, so an ordinary Relay cannot necessarily appoint itself. As a bridge administrator, type `!signal set-relay <SIGNAL_LOGIN_ID>` in the double-bridged Matrix room[^set-relay]. You can test that it works by logging in as the *Whatsapp relay* (which by definition does not have an account in Signal), and sending a message to the Matrix room or the corresponding Whatsapp chat. You should see this message being delivered to the *Signal* chat by the Signal Relay, like so:

![Bridged messages between Matrix, Signal, and WhatsApp Showing WhatsApp Relay relaying messages to WhatsApp and Signal Relay relaying messages to Signal.](https://i.imgur.com/TRqVI6Y.png)

*Bridged messages between Matrix, Signal, and WhatsApp Showing WhatsApp Relay relaying messages to WhatsApp and Signal Relay relaying messages to Signal. [^screenshot-edited]*


Once that worked, repeat this step for the Whatsapp Relay by issuing `!wa set-relay <WHATSAPP_LOGIN_ID>` and use the *Signal* relay to test the connectivity. 

### 4. Invite everyone and chat!

Populate both Signal and WA groups on their respective platforms.
Enjoy seamless delivery of cat memes across platforms, and maybe use this opportunity to lure some people into your Matrix server ;-)

## When something goes wrong
What went wrong with our installation:
### 1. Whatsapp logged out our relay account. 
With the current way it's implemented, Whatsapp requires all users to appear online on their primary device at least once every 2 weeks. Otherwise, the user will be logged out of their linked devices.

**How to fix it:** Turn on a device with the Whatsapp account. If the bridge login is no longer valid, re-login the Relay to the Whatsapp bridge:
1. Log in to Matrix as the Whatsapp Relay Account (`@whatsapp-relay:YOUR_MATRIX_DOMAIN`);
2. Text `logout` to Whatsapp Bridge bot (`@whatsappbot:YOUR_MATRIX_DOMAIN`) if the old login still exists;
3. Text `login qr` or `login phone` to the Whatsapp Bridge bot and follow its instructions;
4. Relaying should be restored automatically.

### 2. Signal logged out our relay account.
Signal requires the primary phone to connect at least once every 30 days, and linked devices may be removed after 45 days of inactivity.

**How to fix it:** Bring the primary phone online. If the bridge must be linked again, text `login` to Signal Bridge Bot (`@signalbot:YOUR_MATRIX_DOMAIN`) and scan the new QR code from the Signal app's Linked Devices screen. The old `link matrix` command is obsolete.

### 3. Whatsapp / Signal group got un-bridged.
Tough luck.
What's probably going to happen is the Bridge bot will create a new Matrix room for the group that got un-bridged. This is very sad for you, because you wanted one group / room to begin with! Luckily, current bridges can link the original room back to its Signal / Whatsapp counterpart without database queries.

**How to fix it:**
1. In the newly created portal room, issue `!wa id` or `!signal id` and save the remote chat ID.
2. Invite the relevant Bridge bot to the original double-bridged Matrix room.
3. Make sure that Bridge bot has enough power in the room. A room or bridge administrator can use `!wa set-pl 100` or `!signal set-pl 100`.
4. In the original Matrix room, issue `!wa bridge --overwrite <WHATSAPP_CHAT_ID>` or `!signal bridge --overwrite <SIGNAL_CHAT_ID>`.
5. Read the warning before confirming: `--overwrite` tombstones and deletes the newer **Matrix portal room**. It does not delete the remote Signal or Whatsapp group.

### 4. ⚠ Your message was not bridged

That one can be very confusing (especially when a Relay Account happily relays the error message to unsuspecting users, who will then experience a major WTF moment). Your course of action depends on the reason the Bridge bot gives you.

![A screenshot of Whatsapp window, where Whatsapp Relay account dutifully relays Signal Bridge Bot errors](https://i.imgur.com/T7kGSnq.png)

*A screenshot[^screenshot-edited] of Whatsapp window, where Whatsapp Relay account dutifully relays Signal Bridge Bot errors.*

**How to fix it:**
* Inspect the logs of the affected bridge service instead of restarting the Matrix homeserver blindly.
* Confirm that the bridge login is connected and the Relay account is still a member of its remote group.
* In the affected portal, issue `!signal sync-portal` or `!wa sync-portal`.
* Re-authenticate only if the bridge reports that its login is invalid or disconnected.
* Upgrade the affected bridge if it is not current, then diagnose the actual current error before unlinking anything.

## Conclusion

At the time of publishing this article, our bridged chat has existed for 8 months with little trouble (apart from what is described in the previous section). Sometimes we seem to lose a message or two, and software updates of the Synapse and the bridges are always a bit of nerve-wracking moment, but overall it works surprisingly well. 

Hopefully this article will help bring more communities closer together, and perhaps inspires people to go and explore the exciting world of Matrix.


[^simcard]: Signal normally uses a phone number for registration. Signal Android beta also supports paid phone-numberless registration through Signal Login, and mautrix-signal supports those accounts since v26.10. A spare number is still required for the Whatsapp Relay.

[^phoneforsignal]: As tempting as it is to place the Relay accounts onto a decade-old Android phone collecting dust in your junk drawer, our experience has shown that a pretty modern device is required to be able to bridge the Signal account. This is because Signal's QR code reader curiously requires quite some computational power.

[^installmatrix]: If you have more money than time, you could also consider having someone else host your Matrix infrastructure. For instance: https://element.io/server-suite

[^set-relay]: With `admin_only: true`, a bridge administrator selects the remote relay login explicitly. With `admin_only: false`, a Relay user can still select its own login by issuing `set-relay` without an argument.

[^bots-login]: To avoid the hassle of switching back and forth between accounts, it is a good idea to use a different device for these manipulations. Or if you are using a web-based client, your browser's Private mode can help too.

[^screenshot-edited]: These screenshots have been edited to have a name of the Whatsapp Relay and Signal Relay accounts consistent with the text of the manual. In reality, our test user decided to call his relay accounts "Whisky". We do not judge him.
