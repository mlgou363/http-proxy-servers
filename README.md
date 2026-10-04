# http proxy server: what it actually does, how to configure one, and why free proxy lists stop working

Most people typing "http proxy server" into a search box aren't after a lecture on request headers. They have one of three problems. Something on their network blocks direct connections. A script keeps getting rate-limited or blocked after a few hundred requests. Or they found a free proxy list and want to know whether using it is a genuinely terrible idea.

Quick answers before the details: an HTTP proxy server is a middleman that accepts your HTTP requests and forwards them to the destination on your behalf. Configuring one is usually a two-field job in the browser, or a single environment variable in a shell. And most free public HTTP proxies are a bad deal — not mainly because they're slow.

The longer version matters, though, because the difference between an HTTP proxy and a SOCKS5 proxy, or between a datacenter IP and a residential one, decides whether a job runs for a week or dies in an hour.

## What an HTTP proxy server actually does

Your client doesn't connect to the website. It connects to the proxy and hands over the request. The proxy opens its own connection to the destination, forwards the request, receives the response, and passes it back. In between it can rewrite headers, cache responses, log the traffic, or block the request entirely.

Two consequences fall out of that design. One, the destination server sees the proxy's IP address rather than yours, which is most of why people want one in the first place. Two, everything you send goes through a machine you don't control unless it's yours.

That second point has a security dimension that's easy to skip past. Plain HTTP is not encrypted, and proxying it doesn't change that — the proxy can read the request, and so can anyone on the path between you and it. The HTTP forwarding model was designed for corporate networks and caching, not for private browsing over untrusted infrastructure. HTTPS traffic is a different story, which brings us to the next section.

For internal use, this is all quite sane. Companies put HTTP proxies in front of employee traffic to enforce access policies, cache frequently requested files, and keep a log of what leaves the network. Libraries and schools have done this for decades.

## HTTP, HTTPS and SOCKS5: what changes when real traffic flows through

An HTTP proxy handles web traffic. It understands HTTP methods, status codes and headers, which is what lets it cache or filter. For HTTPS destinations, it can't read the content, so it switches tactics: the client asks the proxy to open a raw TCP tunnel using the CONNECT method, and the TLS handshake happens end-to-end through that tunnel. Many proxies restrict CONNECT to a small set of ports — port 443 is the usual one — which is why some tools mysteriously fail against a proxy that appears to work fine in a browser.

SOCKS5 does something different. It forwards any TCP traffic without interpreting it, so it doesn't care whether you're sending HTTP, SSH or a game protocol, and it doesn't offer caching or content filtering. It also has no built-in encryption of its own.

The practical rule: for web-only workloads, an HTTP/HTTPS proxy is the more useful tool because it does more. For mixed protocols or non-web traffic, SOCKS5 is the one that will work. Plenty of providers — 9Proxy included — hand out credentials that work for HTTP, HTTPS and SOCKS5 on the same endpoint, so you can switch protocol without changing providers.

## The two error codes that send people back to search

If you spend any time with proxies, you'll meet 407 and 502.

**407 Proxy Authentication Required** means the proxy wants credentials and didn't get usable ones. Either your username and password are wrong, or your client never sent them. Most clients accept credentials embedded in the URL — `http://user:pass@proxy-host:8001/` — or as separate `proxy-user` and `proxy-password` options. Double-check the encoding if your password contains special characters, since a mangled password produces exactly the same error as a missing one.

**502 Bad Gateway** is the proxy saying it couldn't get a valid response from the destination. The upstream server may be down, the proxy may be misconfigured, or a firewall in between may be dropping the connection. When this happens intermittently across many requests, the usual culprit is the proxy itself, not the website you're hitting.

Both errors are debugging clues rather than mysteries, and both appear far more often on free proxies than on paid ones. Free endpoints vanish without warning, and a dead endpoint looks exactly like a misconfiguration from the client side.

## When an HTTP proxy server is the right tool

The use cases are broader than most people assume:

- Pulling public product, price or listing data at a scale that would otherwise trigger rate limits
- Checking how a page, ad or search result renders in another country or city
- Keeping sessions separated when several accounts are managed from the same machine
- Enforcing network policy inside an organisation, and logging what leaves it
- Testing how your own application behaves behind a WAF or a geo restriction
- Accessing a resource that's blocked on your current network for legitimate reasons

What these have in common is that the IP address itself is part of the task. That's the part free lists handle badly.

## Free public HTTP proxy lists: what you're actually borrowing

Free proxy lists mostly consist of open or misconfigured servers, machines that relay traffic on behalf of strangers. Running one is dangerous enough that the documentation for widely used proxy modules carries explicit warnings. The Apache HTTP Server docs, for example, tell administrators not to enable proxying until the server is secured, because an open proxy is a risk to the whole internet. The Xray documentation is blunter: HTTP does not encrypt traffic, it isn't suitable for transmission over the public internet, and using it exposes you to the risk of becoming a zombie for attacks.

That last phrase is the one to remember. When you route through a random open proxy, you're sending your traffic through someone else's machine and trusting that machine not to log it, modify it, or inject something into the response. The IP you're borrowing may also be shared with whoever else found the same list, and its reputation may already be trashed.

There's a practical failure mode on top of the security one. Free proxies die constantly, so a working list from this morning is a graveyard by dinner. If you build a pipeline on them, a meaningful share of your engineering time goes into health checks and retries instead of the actual work.

So: free proxies are fine for a one-off curl request you don't care about. They are a poor foundation for anything that needs to run tomorrow.

## How to point your tools at an HTTP proxy

Once you have proxy credentials — host, port, username, password — the configuration is straightforward almost everywhere.

**Environment variables.** On Linux and macOS, many command-line tools respect `http_proxy` and `https_proxy`, and `no_proxy` takes a comma-separated list of hosts to reach directly:


export http_proxy=http://127.0.0.1:8080
export https_proxy=$http_proxy
export no_proxy=localhost,.internal.example


**Python.** The `httpcore` library accepts a proxy URL directly, and automatically chooses forwarding for HTTP targets and tunnelling for HTTPS ones:

python
import httpcore
proxy = httpcore.HTTPProxy(proxy_url="http://user:pass@proxy-host:8001")
r = proxy.request("GET", "https://example.com/")


**Browsers and desktop tools.** Chrome, Firefox and most anti-detect browsers accept host, port, username and password in their network or profile settings. With 9Proxy's IP-based packages, the flow is different: you filter proxies by country or city in the desktop app, forward an IP to a local port, and then point your tool at `localhost:port` — which means software with no proxy settings at all can still be routed.

**Cloud and automation.** For the GB-based packages, you generate credentials in the dashboard and authenticate either with username and password or by whitelisting your server's IP, then target traffic by country, state, city, ZIP code or ISP. Sticky and rotating session modes are both available, and you can export endpoints as `.txt` or `.csv`.

👉 Check 9Proxy's residential HTTP proxy plans if you'd rather skip the free-list maintenance.

## Renting HTTP proxies: what free lists can't cover

Paid residential proxies exist because IP reputation is the bottleneck. Datacenter IP ranges are easy to identify and get flagged quickly. Residential IPs come from real consumer devices through real ISPs, so they pass the reputation checks that stop datacenter traffic at the door — and they also see the locally correct version of a page, which is the whole point of geo-targeted work.

9Proxy's network covers 20 million-plus residential IPs across 90+ countries, with targeting down to city, ZIP code and ISP level. It supports HTTP, HTTPS and SOCKS5, offers rotating and sticky sessions, and runs on a balance-based model with no monthly subscription — you buy a package, and unused IPs don't expire.

Two features are worth knowing about before you compare price tags. Any proxy you used in the previous 24 hours can be reused at no extra cost if it comes back online, which 9Proxy names the Today List. And if a forwarded IP fails within the first 60 seconds, it gets replaced automatically; auto-refresh handles IPs that drop later.

## What the numbers and the limits actually look like

Third-party testing gives a reasonable picture. A Geekflare review ran 300 sequential requests through rotating residential IPs against a major e-commerce site sitting behind Cloudflare's bot protection, and reported 293 successful passes (97.7%), 5 CAPTCHA challenges and 2 hard blocks, with an average response time of 0.63 seconds. The same test pattern through a datacenter pool produced a 34% block rate on the first pass, according to that review.

An iTWire review reported speeds typically between 50 and 100 Mbps, uptime close to 99% for scraping and account management, and compatibility with automation software for multi-account and price-intelligence tasks. It also noted two caveats worth repeating: the IP-based model requires the desktop app, which is more awkward across multiple devices than a browser extension, and traffic to streaming services like Netflix may still be detected.

Neither of those caveats is a dealbreaker, but they change who the product suits. If you want to click a button in a browser extension and be done, this isn't the most frictionless option. If you're running pipelines, the app requirement is a one-time cost.

One more thing to plan around: residential IPs are naturally unstable. An IP-based proxy stays online for a few hours up to roughly 24 hours, then drops. That's inherent to the category rather than a flaw in this provider, and it's why the auto-refresh and replacement policies matter more than the headline success rate.

## Which 9Proxy model fits your workload

There are only two questions that matter here.

**Does your job need the same IP for a while?** Account sessions, cart flows, and anything behind strict anti-bot detection want session stability. That's the IP-based model: you pay per IP, bandwidth is unlimited, and an IP is only deducted when you actually forward it to a port.

**Is your job bursty and rotation-heavy?** Lightweight scraping, ad verification, API polling and geo-checks consume little data per request but want many different IPs. That's the GB-based model, where you pay for traffic, generate as many endpoints as you like, and traffic stays valid for 180 days — or indefinitely on Enterprise packages.

Bundles exist for people who answered yes to both, usually agencies running a mix of persistent sessions and high-rotation jobs in the same week.

👉 See the current IP-based package prices if your work depends on stable sessions.

## Full plan list and current prices

9Proxy adjusted its IP-based and bundle pricing on 1 June 2026, and its own announcement confirmed that GB-based prices were left untouched. The tables below reflect that split.

### Residential proxies by IP (unlimited bandwidth, IPs don't expire)

| Package | Price per IP | Total | Purchase |
| --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | Buy the 100 IP pack |
| 500 IPs | $0.144 | $72 | Buy the 500 IP pack |
| 1,000 IPs + 500 bonus | $0.084 | $126 | Buy the 1,500 IP pack |
| 100,000 IPs | $0.023 | $2,300 | Buy the 100,000 IP pack |
| 500,000 IPs | $0.018 | $8,625 | Buy the 500,000 IP pack |

The pricing page lists additional tiers between these points, and the effective per-IP rate keeps falling as the package grows — the checkout shows the current figure for whichever volume you pick.

### Residential proxies by GB (traffic-based, rotating or sticky)

| Package | Price per GB | Total | Validity | Purchase |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | Buy the 5 GB pack |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | Buy the 55 GB pack |
| 100 GB | $1.50 | $150 | 180 days | Buy the 100 GB pack |
| 200 GB | $1.00 | $200 | 180 days | Buy the 200 GB pack |
| 1,000 GB | $0.80 | $800 | 180 days | Buy the 1,000 GB pack |
| 2,000 GB | $0.75 | $1,500 | 180 days | Buy the 2,000 GB pack |
| 3,000 GB | $0.72 | $2,160 | Unlimited | Buy the 3,000 GB pack |
| 6,000 GB | $0.70 | $4,200 | Unlimited | Buy the 6,000 GB pack |
| 10,000 GB | $0.68 | $6,800 | Unlimited | Buy the 10,000 GB pack |

### Bundle packages (IPs plus traffic, 180-day traffic validity)

| Bundle | Contents | Price | Purchase |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | Buy the Starter bundle |
| Popular | 1,500 IPs + 50 GB | $180 | Buy the Popular bundle |
| Pro | 5,000 IPs + 500 GB | $720 | Buy the Pro bundle |

If you only need a handful of IPs for a one-off test, the entry IP package at $24 is the smallest commitment. If your traffic is unpredictable and you'd rather not think about IP counts at all, the GB side starts at $15.

👉 Compare the bundle packages if your workload mixes both patterns.

## Coupons, trials and what's verifiable right now

This is where a lot of proxy coupon pages quietly mislead you. Search for a 9Proxy discount code and you'll find lists of expired campaigns presented as live offers.

Here's how the discount system actually works. 9Proxy runs timed campaigns and issues codes to your account rather than publishing permanent ones. During a GB sale in April 2026, for example, completing your first paid GB order of the month automatically generated a personal 9% coupon under My Coupons in the dashboard, which then auto-applied at checkout on the next GB order. It was single-use, limited to GB orders, not stackable, and it expired on 30 June 2026. That campaign is over — the details are useful mainly as an illustration of the pattern: buy first, get a code for the next order.

Checkout has a field for entering coupon codes, so if you do find a current one, it goes there. To find out what's live today, the dashboard coupon section and support are more reliable than any third-party code list.

On trials: 9Proxy doesn't advertise a permanent free tier. iTWire's review notes that trials depend on promotions and that you have to request a code from support. That matches how the rest of their pricing works — nothing is promised permanently.

👉 Open a 9Proxy account and check what's currently available on your dashboard.

## Quick answers

**Is an HTTP proxy server the same as a VPN?** No. A VPN typically routes all device traffic through an encrypted tunnel. An HTTP proxy handles web traffic for the applications you configure, and plain HTTP through it isn't encrypted.

**Can I use free HTTP proxies for scraping?** You can, and you'll spend more time on health checks than on scraping. Clean, stable IPs are exactly what free lists don't provide.

**Do I need the desktop app?** Only for the IP-based packages, which use local port forwarding. The GB-based packages work straight from the dashboard with username/password or IP whitelisting.

**Will a proxy get me around every block?** No. Cloudflare-protected sites still throw occasional CAPTCHAs, and some streaming platforms detect proxy traffic regardless of IP type. Residential proxies improve your odds substantially; they don't guarantee anything.

**What's the minimum spend?** $15 for the smallest GB package, $24 for the smallest IP package, or $30 for the Starter bundle that includes both.

The short version of everything above: an HTTP proxy server is a simple, useful piece of infrastructure, and the free version of it carries costs that aren't visible in the price. If your only need is a single request that doesn't matter, any list will do. If the work needs to run again tomorrow, the money you spend on clean IPs is cheaper than the time you'd spend babysitting dead ones.
