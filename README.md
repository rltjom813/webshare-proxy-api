# Webshare Proxy API Documentation Walkthrough: How Does Authentication Actually Work? Which Endpoints Do You Reach For First? How Do You Pull a Proxy List Straight Into Your Code? (With Full Plan Breakdown and Real Snippets)

Picture this: you've got a scraper half-built, deadline breathing down your neck, and your proxy provider expects you to manually copy-paste IPs into a text file every time you rotate. Painful. The whole point of working with **webshare proxy api documentation** is so you never have to touch that text file again.

Webshare exposes a clean REST API that lets your scripts request fresh proxy lists, check subscription status, replace dead IPs, and watch bandwidth burn down in real time. No dashboard hoping, no manual exports. Once you wire it up, your bots ask Webshare for what they need and Webshare answers in JSON.

This piece walks through every part of the API you'll actually touch, from your first `Authorization: Token` header to filtering proxies by country, downloading lists in different formats, and handling the kinds of edge cases that show up at2 AM. Plus a full comparison of every plan currently sold on the platform so you can pick one without guessing.

> **Quick definition**: The Webshare proxy API is a token-authenticated REST service that lets developers programatically retrieve proxy credentials, manage subscriptions, replace bad proxies, and monitor usage statistics. Every endpoint sits under `https://proxy.webshare.io/api/v2/` and returns JSON.

[👉 See All Webshare Plans and Grab the Free 10-Proxy Tier](https://bit.ly/web_share)

## Why developers reach for the Webshare proxy api documentation in the first place

Most proxy providers treat API access as a premium afterthought. Webshare flips that. Even on the free tier, you get the same API surface paying customers do. Same endpoints, same auth, same response shape. The only thing that scales with your plan is the number of proxies and bandwidth you can pull through them.

That maters because automation breaks the moment your tool of choice depends on a UI. Cron jobs don't click buttons. CI pipelines don't wait for Captchas. The webshare proxy api documentation exists so your scrapers, account managers, ad verifiers, and SERP trackers can fetch proxies on demand.

A few common scenarios where teams plug straight into the API:

- Web scrapers that rotate proxies per request
- Sneaker bots that need a fresh IP pool seconds before drop time
- Account farms that assign one sticky residential IP per profile
- SEO tools pulling SERPs from dozens of geos
- QA pipelines testing how a site renders in different countries

## Geting your API token (the only auth step that maters)

Webshare uses a single token for everything. There's no OAuth dance, no client secret rotation, no callback URLs. You log into your dashboard, generate a token, and put it in your request headers. That's it.

The header format:


Authorization: Token YOUR_API_KEY_HERE


Note the word `Token` in front of the key. People miss this constantly and then spend an hour debugging a 401. The format is `Token` (capital T) followed by a single space, then the key.

Once your token is in place, every endpoint you hit will return your account-scoped data. Tokens never expire on their own, but you can rotate them from the dashboard whenever you want. Treat them like passwords. If you're committing code, throw the token in an environment variable and read it at runtime.

## The endpoints you'll actually use

Webshare's API has a long list of endpoints, but in practice, most projects only touch a handful. Here's the working set most developers settle on after a week or two.

### 1. List proxies


GET https://proxy.webshare.io/api/v2/proxy/list/?mode=direct&page=1&page_size=25


This is the bread-and-butter endpoint. Returns your active proxies as a paginated JSON array, with the IP address, port, username, password, country code, last verification time, and whether the proxy is currently valid.

The `mode` parameter accepts `direct` or `backbone`. Direct mode returns IP:port pairs you can hit straight on. Backbone routes through Webshare's gateway, which is what you want if you need session stickiness or country targeting through a single endpoint.

Pagination maters. Default page size is 25, max is usually 100. If you've got a thousand proxies, you'll be looping through pages.

### 2. Download proxy list


GET https://proxy.webshare.io/api/v2/proxy/list/download/{token}/-/any/username/direct/-/


Different beast. This returns a plain text file in a format your scraper or curl script can read directly. No JSON parsing required. The path segments configure country filter, auth format, mode, and ordering. You can swap `username` for `ip` if you've authorized your own IPs and don't want credential auth.

### 3. Profile and subscription


GET https://proxy.webshare.io/api/v2/profile/
GET https://proxy.webshare.io/api/v2/subscription/


Two quick reads. Profile gives you account details. Subscription tells you which plan you're on, your bandwidth quota, your renewal date, and how much bandwidth you've already burned through this cycle. Tools that monitor usage hit subscription on a five-minute cron and alert when you cross 80%.

### 4. Refresh proxies


POST https://proxy.webshare.io/api/v2/proxy/list/refresh/


When a proxy goes stale or gets baned by a target site, you fire this endpoint. Webshare swaps the bad IP for a fresh one from their pool. There's a refresh quota tied to your plan, so don't blast it on every 403.

### 5. Statistics


GET https://proxy.webshare.io/api/v2/stats/aggregate/


Bandwidth usage, request counts, success rates, broken down by day. Fed this into Grafana and you've got a free monitoring dashboard for your scraping infrastructure.

## A real code snippet you can paste into your project

Here's a Python example that pulls your proxy list and rotates through them on every request. No external libraries beyond `requests`.

python
import os
import requests
from itertools import cycle

API_TOKEN = os.environ["WEBSHARE_TOKEN"]
LIST_URL = "https://proxy.webshare.io/api/v2/proxy/list/?mode=direct&page_size=100"

headers = {"Authorization": f"Token {API_TOKEN}"}
resp = requests.get(LIST_URL, headers=headers, timeout=10)
resp.raise_for_status()

proxies = []
for p in resp.json()["results"]:
    auth = f"{p['username']}:{p['password']}"
    addr = f"{p['proxy_address']}:{p['port']}"
    proxies.append(f"http://{auth}@{addr}")

rotator = cycle(proxies)

def fetch(url):
    proxy = next(rotator)
    return requests.get(url, proxies={"http": proxy, "https": proxy}, timeout=15)


That's a working rotator in roughly 20 lines. Slot it into a scraping lop and you're done. For Node, the same pattern with `axios` and `https-proxy-agent` is just as short.

## Filters, formats, and the small flags that save hours

Half the questions on Webshare's support forum boil down to "I didn't know that flag existed." A few worth knowing:

- **Country filter**: append `?countries=US,GB,DE` to the list endpoint, and you get only proxies in those locations. Handy for geo-targeted scraping.
- **Search**: `?search=192.168` filters by partial IP match. Useful for debugging.
- **Ordering**: `?ordering=-last_verification` pulls most recently verified proxies first.
- **Auth method**: switch your account to IP authorization in the dashboard, and the API returns proxies without username/password. Whitelist your server's IP and you're good.
- **Backbone vs direct**: backbone gives session persistence through a single host:port, direct gives raw access. Backbone is better for sticky sessions, direct is faster.

## Rate limits and what happens when you hit them

The API enforces request rate limits per token. Free accounts hit them faster than paid accounts. When you cross the line, you get a `429 Too Many Requests` with a `Retry-After` header in the response.

The fix is simple: respect the header. Use exponential backoff in your client. Most production code wraps every API call in a retry decorator that backs off when it sees a 429 and retries up to three times before giving up.

python
import time

def with_retry(func, max_retries=3):
    for in range(max_retries):
        r = func()
        if r.status_code != 429:
            return r
        wait = int(r.headers.get("Retry-After", 2 ** i))
        time.sleep(wait)
    return r


Production-grade code, eight lines.

## Plan comparison: every option Webshare currently sells

Webshare splits its catalog into four product families. Each family has multiple tiers that scale with the number of proxies, bandwidth, or threads you need. Pricing on Webshare drops as you commit to higher volume, and the API works identically across every plan.

| Plan Family | Best For | Key Configuration | Pricing Model | Get Started |
| --- | --- | --- | --- | --- |
| **Free** | Testing the API, small personal scripts | 10 datacenter proxies, 1 GB/month bandwidth, full API access | $0 forever | [ Claim 10 Free Proxies Now](https://bit.ly/web_share) |
| **Proxy Server (Datacenter)** | Bulk scraping, high-volume requests, sneaker bots | 100 to 30,000+ shared/private datacenter proxies, configurable bandwidth and thread limits | Pay monthly, scales with proxy count and bandwidth | [ Configure Your Datacenter Plan](https://bit.ly/web_share) |
| **Static Residential (ISP)** | Account management, sticky sessions, social media automation | 1 to 1,000+ static residential IPs from real ISPs, unlimited bandwidth on most tiers | Per-proxy monthly pricing | [ Pick Your Static Residential Plan](https://bit.ly/web_share) |
| **Residential Rotating** | Geo-targeted scraping, ad verification, SERP tracking | Access to 30M+ residential IP pool, country/city/state targeting, rotating per request or sticky | Pay per GB of bandwidth | [ Start with Rotating Residential](https://bit.ly/web_share) |
| **Custom / Enterprise** | Teams, agencies, anyone needing custom limits or SLAs | Negotiated proxy count, bandwidth, and dedicated support | Custom quote | [ Talk to Webshare About Custom Pricing](https://bit.ly/web_share) |

A few things worth noting before you pick. Datacenter is the cheapest per request but easiest for sites to detect and block. Residential rotating is the gold standard for stuborn targets like sneaker sites, social platforms, or anything cloudflare-protected, but you pay by the gigabyte so it ads up fast. Static residential lives in the middle: pricier per proxy than datacenter, but each IP is a real residential address and stays the same across sessions. That last bit is gold for account management.

If you're not sure which to pick, the free plan exists for exactly this. Pull 10 datacenter proxies, run them against your real targets, and see if you get blocked. If you do, escalate to residential. If you don't, datacenter saves you a fortune.

## Trust signals worth checking before you commit

Webshare has been operating since 2018 and currently serves over a million users globally according to its own published numbers. On Trustpilot, the service holds a strong rating across thousands of reviews, with most positive reviews caling out reliable uptime and fast speeds, and most critical reviews pointing at the learning curve for picking the right product family.

Three things make the platform stand out for API-first developers:

- The free tier is genuinely useful, not a fake teaser. 10 proxies and 1 GB is enough to test almost any integration.
- Documentation is published openly at the developer portal, with response examples for every endpoint.
- Money-back guarantees on most paid plans give you a runway to test in production before committing.

## Common scenarios and which plan handles them best

**You're scraping product prices from e-commerce sites.** Datacenter is fine. Most retail sites don't aggressively detect proxies. Buy a Proxy Server plan with enough threads for your concurrency, plug the API into your scraper, and you're set.

**You're running social media automation.** Static residential. Each account needs its own sticky IP that doesn't change. Datacenter IPs get baned within hours on platforms like Instagram or LinkedIn.

**You're building an SEO rank tracker.** Rotating residential with country targeting. SERPs vary by location, and you need a fresh IP per query to avoid Google's bot detection.

**You're scraping protected targets like sneaker sites.** Residential rotating, ideally with sticky sessions for the checkout flow. Pay the bandwidth premium. Datacenter won't get past the bot wall.

**You're just learning.** Free plan. No reason to pay until you know what you need.

## Plain-language recap of what we just covered

Webshare's API uses a single token for auth. The main endpoint you'll hit returns your proxy list in JSON. Other endpoints handle subscription status, refreshing dead proxies, and puling usage stats. Free tier gets you 10 proxies and full API access, which is enough to test anything. Paid plans split into datacenter, static residential, rotating residential, and custom. Pick based on what your target site can detect and how sticky your sessions need to be.

[👉 Get Started with Webshare's API on the Free Plan](https://bit.ly/web_share)

## FAQ

### What is the base URL for the Webshare proxy API?

All endpoints sit under `https://proxy.webshare.io/api/v2/`. Older `v1` paths still resolve in some cases, but new integrations should use v2 because the response schemas are stable and v1 is being phased out.

### How do I generate an API token?

Log into your Webshare dashboard, open the API section in account settings, and click "Generate Token." Copy it once and store it in an environment variable. If you lose it, you'll need to regenerate. Tokens never expire on their own.

### Can I use the API on the free plan?

Yes. The full API is available on every tier including free. The only thing that scales with your plan is the number of proxies the API returns and how much bandwidth you can push through them.

### What's the difference between direct mode and backbone mode in the proxy list endpoint?

Direct mode returns individual `IP:port` pairs that you connect to directly. Backbone mode routes everything through a single Webshare gateway address, which is how you get session stickiness and country targeting via a single endpoint. Direct is faster, backbone is more flexible.

### How do I rotate proxies on every request?

Pull your full proxy list once, store it in memory, and use a round-robin pattern (Python's `itertools.cycle` or any equivalent in your language) to pick the next proxy for each outbound request. Refresh the list every few hours to catch any IP changes.

### What happens when a proxy stops working?

Hit the refresh endpoint. Webshare retires the bad proxy and assigns you a fresh one from the pool. Refresh frequency is caped per plan, so don't refresh on every transient error, only when an IP is consistently failing.

### Does Webshare offer a money-back guarantee?

Most paid plans include a refund window. Check the specific plan's terms during checkout for the exact policy. The free plan exists in part so you can validate the service before paying.

### How do I filter proxies by country?

Append `?countries=US,GB,DE` (coma-separated ISO country codes) to the list endpoint. Only proxies in those countries will be returned. Useful for geo-targeted scraping or testing localized site versions.

## Closing thoughts

If you've read this far, you already know more about the webshare proxy api documentation than 90% of people who buy proxy plans. The API is genuinely well-designed: clean REST conventions, predictable JSON, one auth token, no surprises.

Start free, prove the integration works against your real targets, then scale into the plan that matches the kind of sites you're hitting. Most teams end up with a mixed setup: datacenter for the bulk of their scraping, residential for the stubborn 5% of targets that block everything else.

Either way, the API stays the same. Write your client once and you're set for whichever plan you graduate into.
