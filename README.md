# n8n proxy: how to route HTTP Request nodes through residential IPs and stop the 403s

Your workflow ran fine for two weeks. Then the same node that used to return clean JSON starts returning `403 Forbidden`, or a CAPTCHA page, or an empty array. Nothing changed in your credentials, nothing changed in the API. What changed is that the target site got a better look at where the requests are coming from.

n8n makes this situation common. It runs on a VPS or in a cloud container, so every outbound request carries a datacenter IP. Those ranges are public, and anti-bot systems treat them as non-human by default. Add a loop node that fires dozens of requests a second and you have a very recognizable fingerprint.

This is the problem a proxy solves. Below is how proxies actually work inside n8n — where to configure them, which settings matter, what n8n quietly ignores, and what the traffic math looks like so you can pick a plan without guessing.

## Why n8n gets blocked more than your browser does

Three things stack up against an unproxied workflow.

The IP is from a hosting provider. Sites that care about bot traffic assign those ranges a lower trust score, and some rate-limit them far more aggressively than they rate-limit home connections.

The User-Agent can be generic. If you leave the default headers on an HTTP Request node, the request looks like automation, not like Chrome. This one is easy to fix — turn on Send Headers and paste a real browser User-Agent string — but it only helps if the IP behind it looks plausible too.

The rate is unnatural. A loop over 500 URLs from a single IP is a pattern, regardless of headers. Rotation spreads those requests across many IPs so no single address looks abusive.

Any one of these alone might survive. Together they don't.

## The three places a proxy can live in n8n

Where you configure the proxy depends on how much of your workflow needs it, and on whether you're running self-hosted or on n8n Cloud.

### 1. Inside the HTTP Request node (works everywhere)

This is the most granular option and the only one available on n8n Cloud, where you don't control the runtime environment.

Open the HTTP Request node, scroll to the bottom, click **Add Option**, and pick **Proxy**. Paste the proxy URL in the standard format:


http://USERNAME:PASSWORD@HOST:PORT


The `http://` prefix is not optional. A common failure mode is pasting just `host:port` or `user:pass@host:port` — the node accepts the string but the request goes out unproxied or errors out. One n8n community thread on exactly this ended with "I added `http://` and now it works."

The upside of node-level config: different nodes in the same workflow can use different countries, or a different proxy entirely. The downside: twenty nodes means twenty edits.

### 2. Environment variables (self-hosted only)

To proxy everything at once, n8n reads the standard variables:

- `HTTP_PROXY` — for unencrypted HTTP traffic from nodes
- `HTTPS_PROXY` — for TLS traffic, and in practice the one you actually need
- `ALL_PROXY` — fallback when the two above aren't set
- `NO_PROXY` — comma-separated hosts that should connect directly, bypassing the proxy

One trap worth knowing: n8n handles these through the `proxy-from-env` package, which gives lowercase variables (`https_proxy`) precedence over uppercase ones when both exist. If you set both somewhere in your stack, the lowercase one wins and your uppercase setting looks broken.

In a `docker-compose.yml` it's an environment block:

yaml
environment:
  - HTTPS_PROXY=http://user:pass@gw.dataimpulse.com:823
  - NO_PROXY=localhost,127.0.0.1,postgres


Always put your database, Redis, and internal service hostnames in `NO_PROXY`. Routing internal traffic through an external proxy is a fast way to break a working instance.

### 3. Community nodes

There are third-party nodes that build proxy URLs for you and hand them to a downstream HTTP Request node — `n8n-nodes-netproxy` and `n8n-nodes-proxyhat` are two examples, and similar packages exist for other providers. They add rotation strategies (random, round-robin, auto-switch on dead proxy) and a "test connection" operation.

These are optional. If your proxy provider gives you one rotating gateway endpoint, you don't need a community node — a single proxy string in the node option does the same job. Community nodes mainly earn their place when you're juggling a manual list of dozens of separate proxy endpoints and want failover logic. They're also mostly a self-hosted affair; installing them requires access to community node settings.

## The exact proxy string for DataImpulse inside n8n

DataImpulse runs a single gateway endpoint with one host and one port, which makes it unusually easy to wire into n8n. Credentials look like this:


http://LOGIN:PASSWORD@gw.dataimpulse.com:823


That string goes directly into the HTTP Request node's Proxy field. No node installation, no middleware.

Country targeting is done inside the username, using the `__` delimiter and dot-separated values:


http://LOGIN__cr.de:PASSWORD@gw.dataimpulse.com:823


Multiple countries work the same way — `LOGIN__cr.de,au` for Germany or Australia. This matters for n8n workflows that check prices, SERPs, or availability, because the data you get back is rendered according to the exit IP's location. A price-monitoring workflow exiting from the wrong country returns the wrong prices, and your downstream node builds reports on top of bad data without ever throwing an error.

The gateway serves both HTTP and SOCKS5, so you can swap the scheme if a target or library behaves better on SOCKS5. Country targeting is included at no extra charge; city, ZIP, and ASN targeting are paid add-ons set either in the DataImpulse dashboard or manually in the username string.

Because this is a standard endpoint, it works identically with Scrapy, Selenium, Puppeteer, or any other tool you might be calling from an Execute Command node.

👉 [Get DataImpulse proxy credentials for your n8n workflows](https://bit.ly/dataimPulse)

## Rotating vs sticky sessions: pick by workflow shape

Most n8n jobs fall into one of two patterns, and they want opposite session behavior.

**Single-request nodes want rotation.** If each HTTP Request node fetches an independent URL — a product page, a SERP, a public dataset — you want a fresh IP per request. DataImpulse's gateway rotates by default, which is the behavior you want with no configuration at all. Nothing to set in n8n.

**Multi-step flows against one site want sticky.** If your workflow logs in, adds to cart, then reads a confirmation page, all three requests need to come from the same IP or the session breaks. Sticky sessions hold one IP for a defined window. The DataImpulse dashboard and username parameters both let you pin a session, so you set it once and the same string keeps working in n8n.

A practical pattern for mixed workflows: keep the default rotating string as an n8n credential, and create a second credential with a sticky session for the login-heavy branch. Swap between them per node instead of rewriting the workflow.

## What n8n won't proxy for you

Two limitations are worth knowing before you assume everything is covered.

n8n's proxy support has historically been narrower than "all outbound traffic." Older versions proxied n8n's own request utilities and axios calls only. A later fix extended HTTP proxy support across native Node.js `http`/`https` traffic by configuring the global agents, which brought libraries like the RSS Read node's parser into scope. Even so, some requests still escape: libraries that use `undici` as their HTTP client, or that explicitly override the default agent, can bypass `HTTP_PROXY` and `HTTPS_PROXY` entirely.

The practical consequence: if a third-party node in your workflow keeps getting blocked despite env vars being set correctly, the node's HTTP client is probably ignoring them. Test that path with a plain HTTP Request node using the node-level Proxy option, which is applied by n8n itself rather than by environment inheritance.

Also worth separating: `N8N_PROXY_HOPS` is unrelated to outbound traffic. That variable tells n8n how many reverse proxies it sits behind when reading client IPs. It has nothing to do with routing your requests out through a proxy.

## How much traffic does an n8n workflow actually burn?

Proxy pricing is per gigabyte, so the question that matters is how many requests a gigabyte buys.

A typical HTML page response — before images, fonts, and scripts — lands somewhere between 100 KB and 500 KB. On that basis, 1 GB covers roughly **2,000 to 10,000 page fetches**. At $1/GB, that's on the order of $0.0001 to $0.0005 per request. Treat it as a rough planning estimate, not a guarantee: heavy JSON APIs with large payloads inflate the number, and a node that fetches a 5 MB data dump burns 50 times the budget of one pulling a SERP.

Rough sizing for common n8n jobs at $1/GB:

| Workflow | Requests per run | Rough cost per run |
| --- | --- | --- |
| Daily SERP rank check, 200 keywords, 1 run/day | 200 | ~$0.02–0.10 |
| Price monitor, 1,000 product pages, 4×/day | 4,000 | ~$0.40–2.00 |
| Lead enrichment, 10,000 pages, monthly | 10,000 | ~$1.00–5.00 |

That last column is the reason pay-as-you-go pricing fits automation work well. n8n workflows are bursty — a cron job fires, burns 40 MB, then sits idle for six hours. Subscription plans with monthly traffic caps that expire don't match that shape.

## DataImpulse plans and pricing

DataImpulse bills per gigabyte with no subscription. Traffic doesn't expire, so unused balance from a quiet month stays available for the next one. All plans run through the same `gw.dataimpulse.com:823` gateway, which means anything you configure in n8n keeps working if you change tier.

| Plan | Starting price | Volume pricing | What you get | Billing | Get started |
| --- | --- | --- | --- | --- | --- |
| Residential proxies | $1/GB | $0.80/GB at 1 TB, $0.70/GB at 5 TB | 90M+ ethically sourced IPs, 195 countries, HTTP(S) + SOCKS5, rotating and sticky sessions, country targeting included | Pay-as-you-go per GB, never expires | [Start with residential at $1/GB](https://bit.ly/dataimPulse) |
| Datacenter proxies | $0.50/GB | Bulk pricing available | Low latency, built for high-volume scraping on targets without strict bot defenses | Pay-as-you-go per GB, never expires | [Compare datacenter proxy pricing](https://bit.ly/dataimPulse) |
| Mobile proxies | $2/GB | Bulk pricing available | 3G/4G/5G/LTE mobile IPs, hardest to detect and block, rotating and sticky sessions | Pay-as-you-go per GB, never expires | [Check mobile proxy plans](https://bit.ly/dataimPulse) |
| Premium residential | From $5/GB | Custom volume terms | Dedicated account manager, 99.9% uptime target, extra-low latency for the most sensitive targets | Pay-as-you-go, choose GB volume then add funds | [See premium residential options](https://bit.ly/dataimPulse) |
| Starter package | $5 for 5 GB | — | Same gateway and features across proxy types, to test against your own targets before scaling | One-time top-up, no subscription | [Start with the $5 starter package](https://bit.ly/dataimPulse) |

DataImpulse publishes a 99.51% success rate and roughly one-second response times. Third-party listings reference a 4.8/5 G2 rating, and at least one independent review site scores it 4.3/5 while noting fewer enterprise-grade tooling features than Bright Data or Oxylabs. The gap that matters for n8n users is real: you get a gateway endpoint and a dashboard, not a full orchestration suite. If your workflow needs managed headless rendering and CAPTCHA solving baked into the provider, this isn't that.

## Which tier fits which n8n workflow

**Most n8n automation should start on residential at $1/GB.** A SERP tracker, a price monitor, a lead-scraping workflow, an RSS-plus-enrichment pipeline — these hit targets with some bot protection, and residential IPs are what keeps them returning data. Start with the $5 / 5 GB package, point it at your actual target, and measure what your specific workflow costs per successful request. That number is what you scale on, not the sticker price.

**Switch to datacenter at $0.50/GB when the target doesn't fight back.** Internal APIs, your own services, public government datasets, open directories — if nothing on the other side is scoring IP reputation, paying residential rates is waste. Keep the datacenter credential on those nodes and residential on the rest; n8n lets you mix both in one workflow.

**Reach for mobile at $2/GB only when residential fails repeatedly.** Mobile IPs are genuinely harder to block because carrier NAT means thousands of real users share the same address. That's worth 2× the residential rate on social platforms or heavily defended endpoints. It's not worth it for a product page that already works on residential.

**Consider premium residential if the workflow runs on a schedule you can't miss.** The $5/GB tier is a different price bracket. Justify it with the cost of a failed run, not with the cost per gigabyte.

## Troubleshooting the three errors you'll actually see

**`407 Proxy Authentication Required`** — credentials are wrong, or your login string got mangled. DataImpulse usernames can contain `__cr.us` suffixes; if you're assembling the string dynamically in a Set node, a stray character breaks auth. Test the identical string in a terminal with `curl -x` first, then paste it back into n8n.

**`ECONNREFUSED` or connection timeout** — the proxy is unreachable from your host, or a firewall is blocking outbound port 823. Test the workflow with `NO_PROXY` unset for that hostname. If you're in Docker, confirm the container has outbound access at all.

**Requests still coming from your server IP** — you set env vars, but the node or library ignores them. Check the node-level Proxy option on a test HTTP Request node. If node-level works and env vars don't, the offending path is a library that bypasses the global agent. If neither works, verify you didn't set lowercase and uppercase variants where the lowercase one is empty or wrong.

A five-minute diagnostic worth building once: a small workflow with an HTTP Request node pointed at an IP-echo service, through your proxy, and a second node outputting the result. When something breaks, run it. It tells you immediately whether the problem is your proxy setup or the target site blocking you anyway.

## The short version

Set the proxy in the HTTP Request node for granular control, or via `HTTPS_PROXY` if you're self-hosted and want everything routed. Remember the lowercase-over-uppercase precedence rule and put internal hosts in `NO_PROXY`. Use the standard `http://user:pass@host:port` format, keep rotation on for independent requests, and switch to sticky sessions for multi-step flows.

On the provider side, the reason DataImpulse comes up for n8n work is structural: one gateway endpoint that pastes straight into the node's Proxy field, country targeting embedded in the username, $1/GB residential with traffic that doesn't expire, and a $5 entry point that's small enough to measure your real cost per successful request before committing to a bigger top-up. Non-expiring traffic is the part that matters most for automation — cron-driven workflows are lumpy, and paying per gigabyte without a monthly reset fits that pattern better than a subscription sized for a busy month.

👉 [Set up a DataImpulse account and test it against your n8n workflow](https://bit.ly/dataimPulse)
