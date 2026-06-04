---
layout: post
title: "15 Years of Self-Hosted Email. Here's Why I Stopped."
date: 2026-06-04
description: I ran Postfix, Dovecot, Rspamd, OpenDMARC, and a replicated OpenLDAP backend for fifteen years. Here's the honest reason I handed it all to Fastmail.
tags:
  - email
  - self-hosted
  - fastmail
  - postfix
  - dovecot
  - rspamd
  - infrastructure
  - homelab
---

I ran my own email infrastructure for fifteen years. Postfix, Dovecot, Rspamd, OpenDMARC, and a replicated OpenLDAP backend holding it all together. It worked. Most of the time it worked really well. I knew every config file, every queue, every header that touched the system. I built it, I maintained it, and at a certain point I automated the whole stack — provisioning, configuration, updates — to cut down on the time it demanded. That bought me a few more years. For a long time I was genuinely proud of it.

Then life changed the math.

When you're a single person with time to burn and a homelab that doubles as a hobby, self-hosted email is a satisfying challenge. You tune your spam filters, you watch your delivery rates, you feel a specific kind of satisfaction when a complex mail routing rule does exactly what you designed it to do at 2am. It's infrastructure as a creative outlet.

What it is not, is something you can casually ignore when something breaks.

And things break. Deliverability suddenly tanks because your IP ended up on a blocklist you didn't know existed. A Rspamd update changes scoring behavior and legitimate mail starts getting flagged. OpenLDAP replication hiccups and your mail client starts throwing authentication errors at the worst possible moment. None of these are catastrophic on their own — but debugging them requires focus, time, and the ability to context-switch out of whatever else you're doing.

Here's the thing about context-switching when you have a family and two people building their own businesses at the same time: there's no such thing as a quick debugging session anymore. What used to be a focused 45-minute dive into mail logs is now interrupted, resumed, interrupted again, and eventually finished at midnight when everyone else is asleep — if you finish it at all. The system doesn't care that you have a client call in the morning. It just sits there, broken, waiting.

My wife and I are both building our own business niches. She runs Klein Legacy Wealth, helping families build tax-advantaged wealth through IUL and annuities. I run Klein Technology Consulting. Between client work, business development, and actually having a life outside of screens, the evenings I used to spend deep in mail logs are now spent on things that actually move the needle. Self-hosted email stopped being a satisfying project and started being a liability I couldn't afford to babysit.

There's also a deliverability reality that nobody in the self-hosting community talks about enough: the major providers — Google, Microsoft, Apple — have made it increasingly hostile to run your own mail server. SPF, DKIM, DMARC, BIMI, ARC — you can have all of it configured perfectly and still end up in spam because your IP is a residential or small VPS range that a bulk filter doesn't trust. The goalposts keep moving, the standards keep evolving, and staying ahead of it is genuinely a full-time job disguised as a hobby.

I've been there. I've done the blocklist delisting requests. I've tuned Rspamd scoring until my eyes glazed over. I've rebuilt OpenLDAP replication after an unclean shutdown left the replicas in a state that made me question my life choices. All of it was educational. None of it is something I want to do again at 11pm on a Tuesday.

So I migrated to [Fastmail](https://fastmail.com).

Not Gmail — I have no interest in handing my personal and business communications to a company whose product is advertising. Not iCloud. Fastmail — an independent, privacy-focused provider that's been doing this since 1999, doesn't mine your email for advertising, and gives you proper custom domain support with full DNS control. The migration was straightforward: export, import, update MX records, done. My addresses stayed the same. My family didn't notice a thing. My wife's business email came along for the ride too.

What I gained was time. Time I was spending on email administration that now goes toward things that actually matter — client work, the homelab projects I actually enjoy, and occasionally sleeping before midnight. The system doesn't page me. It doesn't need quarterly tuning. Fastmail's deliverability is handled by a team whose entire existence depends on getting it right. That's their problem now, not mine.

To be clear — fifteen years of running your own mail stack teaches you how email actually works at a level most people never bother to understand. The headers, the routing, the authentication chain, the spam filtering pipeline — I know all of it, and that knowledge has been genuinely useful in ways that extend well beyond email. I don't regret a single config file.

But knowing how something works doesn't mean you have to keep running it yourself forever. Sometimes the most senior engineering decision you can make is recognizing when a system has served its purpose, handing it off to someone whose entire job it is, and directing that reclaimed time toward something that actually needs you.

The homelab is still very much alive. It's just one less thing that can wake me up at night.
