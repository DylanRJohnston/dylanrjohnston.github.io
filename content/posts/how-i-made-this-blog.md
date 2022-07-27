---
title: "How I Made This Blog"
date: 2022-07-27
draft: false
tags: ["hugo", "github", "ci", "cd", "cloudflare", "actions"]
---

# Plan of Attack

When evaluating options for starting a blog I wanted something that was **fast**, **free**, and **easy** to setup CI / CD for. GitHub Pages ticks both the **free** and **easy** boxes but it can be a bit slow to respond. To solve the speed issue I stuck cloudflare in front of GitHub to leverage its CDN caching, ensuring that response times are fast.

```goat
┌──────────────────────────┐
│    GitHub Repository     │    ┌────────────────┐     ┌──────┐
│                          ├────► GitHub Actions ├─────► Hugo │
│ dylanrjohnston.github.io │    └────────────────┘     └───┬──┘
└──────────────────────────┘                               │
                                                           │
                                                           │
             ┌────────────┐     ┌────────────┐             │
┌──────┐     │ Cloudflare │     │ Cloudflare │     ┌───────▼──────┐
│ User ├─────►            ├─────►            ├─────► GitHub Pages │
└──────┘     │ Nameserver │     │    CDN     │     └──────────────┘
             └────────────┘     └────────────┘
```

## Domain Name Provider

I use Hover for my domain name provider, but you can use whatever provider you want. You need to delegate the nameservers for your domain to cloudflare's nameservers. Under DNS in your Cloudflare Console you'll see two nameservers assigned to your account. For me they are `arnold.ns.cloudflare.com` and `reza.ns.cloudflare.com`.

## Cloudflare

Now that Cloudflare is acting as the nameserver for your domain, we need to configure it to point to GitHub pages and set up caching rules. You can find more detailed instructions on using a custom domain with GitHub Pages [here](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages) and [here](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain). The second link contains list of IP addresses we need to set up as A records in Cloudflare.

```bash
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

## GitHub Pages

GitHub pages works for any repository with it enabled, but there is a special repository for each user called `{username}.github.io` that GitHub uses for the corresponding URL of the same name. You need a file called CNAME in the root of project that contains the domain name you're pointing towards GitHub. Placing it in the `{username}.github.io` repository will let you use that custom domain for the GitHub pages of all your other repositories at `https://{domain}/{project-name}` for example https://dylanj.xyz/julia-webgl.

## Hugo

To actually generate the site, I decided to go with [Hugo](https://gohugo.io/) which is a very fast static site generator written in Go. You could just as easily use any other static site generator like [Next.js](https://nextjs.org/docs/advanced-features/static-html-export) if you'd prefer to mess around with React instead. I was going for simplicity and being focused on writing content and not writing components which is why I chose Hugo.

## GitHub Actions

To automatically deploy our static site to GitHub pages I'm using GitHub actions which interact really nicely with GitHub pages.

It's as simple as dropping a file in `.github/workflows`. This is the current GitHub action script used for this blog.
{{<import-code lang="yml" file="/.github/workflows/deploy.yml">}}
