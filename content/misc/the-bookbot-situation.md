---
title: "The Bookbot Situation"
date: 2025-09-19T22:54:54+05:30
draft: false
tags: ["bookbot", "discord", "javascript", "typescript"]
---

---

This is going to be a relatively short post as I am writing this to shed light on the situation of a bot I built a few years ago called [BookBot](https://github.com/rv178/bookbot).


## Current Situation

As part of Discord's safety policy, verified bots have to be periodically re-verified. The last time the bot was verified was 3 years ago.
It used to be added to around ~2850 servers and actively running before Discord enforced re-verification.

From their [article](https://support-dev.discord.com/hc/en-us/articles/23926564536471-How-Do-I-Get-My-App-Verified):
> The most noteworthy requirement is that the owner of the development team which owns the app will need to verify their identity through Stripe, our identity verification provider. If you have questions about this, please see our Stripe FAQ. __You may be asked to periodically reverify your id with stripe.__

Normally, this would not be an issue but as my team member and co-developer Maks has been inactive for a long time (with no means of contacting him).

He is also the owner of the Discord team that owns bookbot (I had transferred the ownership so he could verify the bot back then as I was unable to, this happened 3 years ago).

![Alt](https://files.catbox.moe/bz8cji.png)


This means that I cannot re-verify the bot without him transferring the ownership back to me. I did contact Discord regarding this, and this was
their response:

![Alt](https://files.catbox.moe/5upgz3.png)

In short, BookBot can no longer be added to new Discord servers until it is re-verified.


## What next?

For the existing users of BookBot, the main bot is going to be down. However, I have gone ahead and created a second bot which *can*
be added to new servers. Here is the [invite link](https://discord.com/api/oauth2/authorize?client_id=1418600171771789503&permissions=2147862592&scope=bot%20applications.commands).

Also, a special thanks to `@codyrocks10` on Discord who kindly decided to provide hosting for the bot.


That's it for this blog. Until next time, bye!


