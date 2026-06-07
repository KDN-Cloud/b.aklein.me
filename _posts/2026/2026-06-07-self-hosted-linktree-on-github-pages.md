---
layout: post
title: "Ditching Linktree: Host Your Own Link Hub on GitHub Pages"
date: 2026-06-07
author: Anthony Klein
description: >
  Linktree is fine until it isn't. Here's how I forked an open source alternative,
  customized it to match my own ecosystem, and deployed it free on GitHub Pages
  with a custom domain through Cloudflare.
tags:
  - github-pages
  - linktree
  - linktree-alternative
  - self-hosted
  - cloudflare
  - cloudflare-dns
  - dns
  - cname
  - static-site
  - gitops
  - jekyll
  - github
  - open-source
  - custom-domain
  - link-in-bio
  - html
  - css
  - free-hosting
  - font-awesome
  - personal-branding
---

Linktree works. I'm not going to pretend it doesn't. You sign up, you paste in some URLs, you get a page. For most people that's good enough and I'd never argue otherwise.

What Linktree doesn't give you is control. You're on their platform, under their branding, subject to their pricing tier if you want anything beyond the basics, and your links are living in someone else's database. For me personally, that's just not how I build things. Everything I can reasonably own and self-host, I do. A links page is not a complex infrastructure problem. It's a static HTML file. There's no reason to outsource it.

So I built my own, and it runs at [links.aklein.pro](https://links.aklein.pro) — served free off GitHub Pages, custom domain, no subscription, no platform dependency.

Here's how to do the same thing.

## The Starting Point: johnggli/linktree

I didn't build this from scratch. There's an open source project by [johnggli](https://github.com/johnggli/linktree) that gives you a clean, minimal Linktree-style page in plain HTML and CSS. No framework, no build step, no dependencies beyond Font Awesome for icons. It's exactly the right level of complexity for this problem — simple enough to understand completely, flexible enough to make it your own.

Fork it, customize it, deploy it. That's the whole workflow.

My fork lives at [github.com/KDN-Cloud/linktree](https://github.com/KDN-Cloud/linktree).

## What You're Actually Editing

The repo has three files that matter:

`index.html` is everything. Your profile image, your display name, your quote if you want one, and all your links. Each link is an anchor tag with a Font Awesome icon, a label, and a URL. The structure is obvious once you open the file and you don't need to know anything about web development to swap in your own content.

`style.css` controls the look. Background color, button styles, fonts, the animated starfield if you want to keep it. You can strip it down or go deeper depending on how much you want the page to feel like yours.

`CNAME` is a single line containing your custom domain. This is what GitHub Pages reads to know which domain to serve the site from. More on that in a moment.

## The Generator

Rather than hand-editing HTML every time you add a link, I built a local generator tool — `generator.html` — that lives in the repo. Open it in any browser, no server required.

You pick a template style, fill in the label, URL, icon class, and category, and it spits out the HTML snippet ready to paste directly into `index.html`. Three template styles are included: the terminal-style dark card format I use, a clean minimal white card with arrow prefix, and a bold uppercase dark button style. There's also a history panel so you can save links mid-session and reload them without starting over.

The generator ships with four UI themes: light, GitHub dark, navy, and purple. All are switchable from the topbar. It's a local tool, not deployed publicly, so it doesn't need to match the aesthetic of your linktree page itself.

Grab `generator.html` from the repo: [github.com/KDN-Cloud/linktree](https://github.com/KDN-Cloud/linktree)

## Setting Up GitHub Pages

GitHub Pages will host this for free. The deployment model is dead simple — push to your branch, the site updates. No CI pipeline to configure, no build step to worry about since this is plain HTML.

A few notes on setup that aren't immediately obvious:

**Personal vs. Organization repositories** behave slightly differently. My linktree lives under the KDN-Cloud GitHub organization, not my personal account. For an org, GitHub Pages works the same way — you enable it in the repository settings under the Pages section, set the source branch (I use `master`, root directory), and GitHub handles the rest.

**The Pages settings screen** is where you confirm your custom domain and verify DNS. You'll see a green "DNS check successful" once everything is wired up correctly. Until then, GitHub will show a warning — don't panic, DNS propagation takes a few minutes. You can monitor worldwide propagation at [DNSChecker.org](https://dnschecker.org).

**Enforce HTTPS** is a checkbox in the same settings screen. Check it. GitHub handles the TLS certificate automatically via Let's Encrypt once your DNS is verified. There's no reason to serve this over plain HTTP.

## The CNAME File

This file does one job: it tells GitHub Pages what custom domain to answer for. The content is literally just your domain name:

```
linktree.aklein.pro
```

That's the whole file. No `https://`, no trailing slash. When you push this file to your repository, GitHub Pages picks it up automatically. You'll also see this reflected in the Pages settings UI — GitHub reads the CNAME file and pre-populates the custom domain field.

If you change your domain later, update this file and push. GitHub will pick up the change on the next deployment.

## DNS in Cloudflare: Why You Turn the Proxy Off

This is the part that seems to trip people up. If you're managing DNS in Cloudflare, your instinct might be to enable the proxy (the orange cloud) for everything. Enabling DNS proxy adds DDoS protection, hides your origin IP and gives you caching. Generally that instinct is correct.

For GitHub Pages with a custom domain, it creates a problem. GitHub Pages performs domain verification by checking that your DNS resolves to their servers. The Cloudflare proxy intercepts that and GitHub sees Cloudflare's IP, not its own, which breaks the verification and can cause certificate provisioning to fail.

The fix is straightforward: set your DNS record to **DNS only** (gray cloud, not orange). This lets the request pass through to GitHub directly. You're not losing much here. GitHub Pages already sits behind a CDN with HTTPS enforced, so the traffic is encrypted in transit regardless.

For a subdomain setup like mine (`linktree.aklein.pro`), the DNS record looks like this:

```
Type:    CNAME
Name:    linktree
Target:  kdn-cloud.github.io
Proxy:   DNS only (gray cloud)
```

The target is your GitHub Pages endpoint — `<your-org-or-username>.github.io`. Not the repository name, not the full URL of the site. Just the base Pages hostname for your account or org.

If you're using an apex domain (like `yourdomain.com` with no subdomain), you'd use `A` records instead pointing to GitHub's IP addresses. GitHub documents those in their Pages docs and they occasionally change, so verify them there rather than copying from somewhere else.

## Customizing the Page

Once the plumbing is in place, the actual customization is just editing HTML. A few things worth knowing:

**The profile image** is a URL. I use my GitHub avatar (`https://avatars.githubusercontent.com/u/YOUR_USER_ID`) because it's always available, always current, and I don't have to manage another asset. You can use any publicly accessible image URL.

**Font Awesome icons** are already loaded via CDN in the original template. The full icon set is available at [fontawesome.com/icons](https://fontawesome.com/icons). Search for what you want, grab the class name, drop it into the `<i>` tag. The free tier covers a lot.

**Category labels** aren't part of the original template. I added them to group my links into sections — "The Command Center" for professional and infrastructure links, "The Sound Booth" for music and creative stuff, "Active Uplinks" for communication channels. It's just a `<div>` with a class and some CSS. If your links page has enough entries that grouping makes sense, it's worth doing.

**Open Graph and Twitter Card meta tags** are worth adding properly if you care about how the page looks when someone shares the URL. The original template is minimal on this front. I added full OG tags including image dimensions, a `profile` type, and structured data via `application/ld+json`. None of that is required, but it means when someone drops the link in Slack or Discord it shows your photo and a real description instead of the domain name and nothing else.

## The Alias Domain

My links page lives at `linktree.aklein.pro` as the canonical URL, but I also have `links.aklein.pro` pointing to the same place. GitHub Pages only serves one domain per repository, so the second domain is a simple Cloudflare redirect — a Redirect Rule that sends `links.aklein.pro/*` to `https://linktree.aklein.pro/$1` with a 301. Takes about thirty seconds to set up and gives you a shorter, cleaner URL to hand out if you want one.

## What This Costs

Nothing. GitHub Pages is free for public repositories. The only costs involved are whatever you already pay for your domain and DNS. If you're already on Cloudflare's free tier, that's zero too.

Compare that to Linktree Pro at $9/month for custom domains and analytics, or any of the other link-in-bio SaaS options that want a recurring subscription for what is, again, a static HTML file.

## The End State

You end up with a links page that:

- Lives in a git repository you own and control
- Deploys automatically on every push
- Serves over HTTPS with a certificate GitHub manages for you
- Runs on your own domain
- Costs nothing to host
- Can be styled however you want
- Has no platform dependency that can sunset, reprice, or change terms on you

The whole setup from fork to live site takes under an hour, most of which is waiting for DNS to propagate. If you've got a GitHub account and a domain you're already managing in Cloudflare, you already have everything you need.

Source code for my version is at [github.com/KDN-Cloud/linktree](https://github.com/KDN-Cloud/linktree). The live result is at [links.aklein.pro](https://links.aklein.pro). Fork it, break it, make it yours.

