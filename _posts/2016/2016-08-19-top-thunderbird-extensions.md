---
title: 'My Top Thunderbird Mail Extensions'
description: The best Thunderbird extensions in 2026 — updated for accuracy, current compatibility, and what actually still works.
date: 2016-08-19
lastmod: 2026-06-12
tags:
- Mail
- Thunderbird
- Extensions
- Top-Thunderbird-Mail-Extensions
- email-client
- productivity
- open-source
- email-productivity
- open-source-email
---

I originally wrote this post back in 2016. Thunderbird has changed a lot since then. It moved from the old XUL/XPCOM extension model to WebExtensions, gained native OpenPGP encryption, and has been on an aggressive development trajectory backed by a full-time team funded almost entirely by user donations. Some extensions from the original list are deprecated, a few got replaced by built-in features, and a couple have community forks that picked up where upstream left off. This is the updated list for 2026.

Before getting into it — if you're still running Enigmail, uninstall it. Thunderbird has had native OpenPGP support built in since version 78. You don't need an extension for that anymore. Go to Account Settings, find End-to-End Encryption, and configure it from there. Enigmail won't work on any current Thunderbird build and hasn't been actively developed in years.

With that out of the way, here's what's actually worth installing right now.

### [uBird](https://addons.thunderbird.net/en-US/thunderbird/addon/ubird/)

uBlock Origin dropped official Thunderbird support upstream, but the community responded with uBird — a fork built directly from uBlock Origin's own build scripts with only minor changes to make it work properly inside Thunderbird. It does exactly what you'd expect: blocks ads, trackers, and remote content that gets loaded when you open HTML emails. Remote images in emails are a well-known tracking vector. A sender can know when you opened a message, from what IP, and roughly where you are just from a 1x1 pixel loaded silently in the background. uBird handles that without you thinking about it.

### [Send Later](https://addons.thunderbird.net/en-US/thunderbird/addon/send-later-3/)

Still active, still the best way to schedule outgoing email. You write the message whenever, set the send time, and it fires when you told it to. The same caveat from 2016 applies — Thunderbird has to be open for it to actually send — but for anyone running Thunderbird as their daily driver that's a non-issue. If you work across time zones or just want to write something at 11pm without it landing in someone's inbox at 11pm, this is the extension for that.

### [Manually Sort Folders](https://addons.thunderbird.net/en-US/thunderbird/addon/manually-sort-folders/)

A decade later and Thunderbird still hasn't shipped native manual folder sorting. The built-in options are limited and the default sort order is opinionated in ways that don't fit everyone's workflow. This extension gives you drag-and-drop control over folder order and lets you reorder accounts in the folder pane however you want. It's one of those small things that matters a lot if you're managing multiple accounts with a lot of folders.

### [CardBook](https://addons.thunderbird.net/en-US/thunderbird/addon/cardbook/)

CardBook is a full replacement for Thunderbird's built-in address book, built on the CardDAV and vCard standards. If you self-host your contacts on Nextcloud, Radicale, or any CardDAV-compatible server, this is how you get proper two-way sync working inside Thunderbird. The native address book gets the job done for basic use but CardBook is what you want if you're running your own identity stack and need contacts that actually stay in sync.

### [Quicktext](https://addons.thunderbird.net/en-US/thunderbird/addon/quicktext/)

One of the most widely used extensions in the Thunderbird ecosystem and for good reason. Quicktext lets you define email templates and insert them with a keystroke or toolbar click. If you send any recurring type of email — client updates, support responses, meeting follow-ups — this eliminates the copy-paste cycle entirely. It also supports variables so you can inject the recipient's name, today's date, or other dynamic fields into a template automatically.

### [Display Quota](https://addons.thunderbird.net/en-US/thunderbird/addon/display-quota/)

Still relevant if you run your own mail server with quotas configured. This extension shows current mailbox usage per folder and can warn you before you hit a threshold. It's a small utility that does one thing well. If you manage your own Dovecot/Postfix stack and want quota visibility without having to SSH in and check, it's worth keeping around.

### [Signature Switch](https://addons.thunderbird.net/en-US/thunderbird/addon/signature-switch/)

If you juggle multiple email identities — personal, consulting, business — Signature Switch is worth installing. You define multiple signatures and switch between them from the toolbar when composing. No more manually editing the signature block depending on who you're writing to. It pairs well with having separate account identities already set up in Thunderbird.

### [FileLink for Nextcloud / ownCloud](https://addons.thunderbird.net/en-US/thunderbird/addon/filelink-nextcloud-owncloud/)

Thunderbird has built-in FileLink support for handling large attachments by uploading them to a hosted service and inserting a download link instead of embedding the file in the message. This extension extends that to Nextcloud and ownCloud. If you self-host Nextcloud, this is the cleanest way to handle large file sends — the file goes to your server, the recipient gets a link, and you don't have to deal with attachment size limits from mail providers.

### [DKIM Verifier](https://addons.thunderbird.net/en-US/thunderbird/addon/dkim-verifier/)

This extension verifies DKIM signatures on incoming messages and surfaces the result directly in the message header. If you care about mail authentication — and if you're running your own mail server you definitely should — this makes it immediately visible whether a message's From domain actually signed it. Useful for catching spoofed headers and useful for testing your own outbound DKIM setup when you're configuring a new domain.

### A note on Sieve

The Sieve extension from the original post is still around and still works. If you run Dovecot with pigeonhole for server-side filtering, the Sieve extension is the cleanest way to manage those scripts without SSHing into the box. Worth keeping if your mail stack supports ManageSieve.

That's the updated list. The extension ecosystem cleaned up considerably after the move to WebExtensions and most of what exists now actually works reliably on current Thunderbird builds. If you're on anything past version 115 you should be in good shape with all of the above. Drop a comment if there's something you find indispensable that I missed.
