# scrapy rotating proxies: working settings, ban detection, and a $5 way to test before you scale

Most "scrapy rotating proxies" tutorials hand you the same twelve lines of `settings.py` and stop there. That config does work, up to a point. The problem is what happens at request 40,000: your spider keeps returning 200s with a captcha page inside, the middleware thinks everything is fine, and your CSV quietly fills with garbage. Or every request comes back 407 and you spend an evening debugging `ROTATING_PROXY_LIST` when the real issue is how your credentials are formatted.

This guide covers the parts that actually break: where rotation lives in Scrapy's middleware chain, how to tell a ban apart from a slow page, why your `DOWNLOAD_DELAY` suddenly applies per proxy instead of per domain, and how to figure out whether a proxy bill of $5 or $500 is the right size for your crawl.

## Rotation in Scrapy isn't one thing

Scrapy ships with `HttpProxyMiddleware`, and it does almost nothing you'd want here. Give it a proxy and it attaches it to every request forever. When that exit IP gets blocked, the middleware has no opinion about it. Your spider just keeps hammering the same dead route.

So rotation has to be added, and there are two places to add it:

**Client-side rotation.** You maintain a list of proxy endpoints and a middleware picks one per request, watches for failures, and retires bad ones. This is what the `scrapy-rotating-proxies` package does. You control the pool, and you can see which proxies are dead and which got revived.

**Provider-side rotation.** You point your spider at a single gateway URL that hands out a fresh exit IP on every connection. There's no list to maintain, nothing to health-check, and no pool size to guess at. If the endpoint is up, rotation is happening.

Neither is strictly better. A list makes sense when you own static IPs you're paying for by the month. A gateway makes sense for residential traffic, where you're billed per gigabyte and the provider is already managing tens of millions of addresses. Plenty of production spiders use both: a gateway for the hard targets, a static list for cheap bulk work.

## The working setup


pip install scrapy-rotating-proxies


The package registers two downloader middlewares. `RotatingProxyMiddleware` assigns a proxy per request. `BanDetectionMiddleware` decides whether a response looks like a block and rotates away from the failing IP. Priorities matter, and 610/620 is the combination that puts them in the right order relative to Scrapy's built-in transport middleware at 750.

python
# settings.py
ROTATING_PROXY_LIST = [
    'http://user:pass@gw.dataimpulse.com:823',
]

DOWNLOADER_MIDDLEWARES = {
    'rotating_proxies.middlewares.RotatingProxyMiddleware': 610,
    'rotating_proxies.middlewares.BanDetectionMiddleware': 620,
}


That single-line list looks like it defeats the purpose of a rotating-proxy package. It doesn't, when the endpoint itself rotates. A residential gateway from DataImpulse picks a new exit IP per request on the provider side, so the middleware's list has one entry while the actual IP behind it changes constantly. The two rotation layers stack: the gateway swaps IPs, the middleware retries on the same gateway when a response gets flagged.

If you'd rather set the proxy per request, do it through `request.meta`:

python
import scrapy

class GatewaySpider(scrapy.Spider):
    name = 'gateway'
    urls = ['https://httpbin.org/ip']

    def start_requests(self):
        proxy = 'http://user:pass@gw.dataimpulse.com:823'
        for url in self.urls:
            yield scrapy.Request(url, meta={'proxy': proxy}, dont_filter=True)


## Keep the credentials out of the repository

Hardcoding `http://user:pass@host:port` into `settings.py` puts a working credential into your git history permanently. Every clone, every fork, every time someone prints the settings object into a log line, that password is sitting there in plain text.

The `ROTATING_PROXY_LIST_PATH` option exists for this. Point it at a file with one endpoint per line and keep that file in `.gitignore`:

python
ROTATING_PROXY_LIST_PATH = 'proxies.txt'


text
# proxies.txt — one endpoint per line, never committed
http://user:pass@gw.dataimpulse.com:823


For anything running in CI or a container, an environment variable plus `python-dotenv` is tidier still, because the same code can read different credentials per environment without a file swap.

One detail worth getting right early: proxy auth failures return HTTP 407, not 403. A 407 means the proxy rejected your credentials, and no amount of retrying will repair that. A 403 means the target server refused you, which could be your headers, your session, or your IP. Bucketing those two into the same "bad proxy" counter is how people burn healthy endpoints for no reason.

## Ban detection that isn't guesswork

The default `BanDetectionPolicy` in `scrapy-rotating-proxies` is deliberately blunt: if the status isn't 200, the body is empty, or an exception fired, it calls the proxy dead. On a site that returns a soft block as a 200 with a challenge page inside, that heuristic misses everything.

The fix is to score responses on several independent signals instead of one. DataImpulse's own Scrapy writeup calls this a four-signal approach, and the logic is easy to reproduce:

- **Status code.** 403, 429 and 503 are ban candidates. 200, 301 and 302 are usually fine.
- **Redirect target.** A 302 that lands on a login or challenge path is a soft block even though the status looks healthy.
- **Body fingerprint.** Scan the first few kilobytes for `captcha`, `access denied`, `Just a moment` and similar markers.
- **Exceptions and timeouts.** Counted separately, because a connection reset is a different animal from a refusal.

Subclassing the default policy is a few lines:

python
# myproject/policy.py
from rotating_proxies.policy import BanDetectionPolicy

class MyPolicy(BanDetectionPolicy):
    def response_is_ban(self, request, response):
        ban = super().response_is_ban(request, response)
        ban = ban or b'captcha' in response.body
        return ban

    def exception_is_ban(self, request, exception):
        return None


python
# settings.py
ROTATING_PROXY_BAN_POLICY = 'myproject.policy.MyBanPolicy'


The reason this matters more than it looks: retrying a failed request and rotating away from a bad proxy are two different actions. If the target is having a bad minute, rotating burns a good IP. If the IP is burned, retrying the same route just wastes requests. You can't tell the two apart without signals.

Dead proxies aren't dead forever, either. The package re-checks retired endpoints with randomized exponential backoff — first recheck quickly, then later, then later still. `ROTATING_PROXY_BACKOFF_BASE` and `ROTATING_PROXY_BACKOFF_CAP` control the curve. On residential gateways this barely matters, since there's no endpoint to revive. On a static list it's the difference between a pool that recovers overnight and one that silently shrinks to nothing.

## Prove that rotation is happening

Never assume it. Hit an IP echo endpoint ten times and read the log:

python
class IPCheckSpider(scrapy.Spider):
    name = 'ipcheck'
    start_urls = ['https://httpbin.org/ip'] * 10

    def parse(self, response):
        self.logger.info('Exit IP: %s', response.json()['origin'])


bash
scrapy crawl ipcheck


You want ten different values. If they're all identical, rotation is off, and the usual suspects are: middlewares not actually enabled, an empty proxy list, or Scrapy serving the requests out of its HTTP cache. Disable the cache while testing.

## The concurrency trap

Here's the setting nobody mentions in the quickstart. When `RotatingProxyMiddleware` is active, Scrapy's concurrency controls become *per proxy*, not per domain. Set `CONCURRENT_REQUESTS_PER_DOMAIN = 2` and your spider makes at most two concurrent connections to each proxy, regardless of which sites it's hitting.

With a 20-proxy list that's a 20x throughput multiplication you didn't ask for. With a rotating gateway, you effectively have one endpoint in the list, so the ceiling is low — which is often exactly what you want, because concurrency limits are also rate limits, and rate limiting is most of what keeps residential IPs alive.

The same applies to `DOWNLOAD_DELAY` and to `AUTOTHROTTLE_*`. Both become per-proxy. If you're moving a working spider from a single fixed proxy to a rotating gateway, expect your effective request rate to change, and expect it to change in whichever direction you didn't plan for.

## Cheap proxies on easy targets, expensive where it counts

Rotating isn't the same as hiding. Putting 500 datacenter IPs behind a spider doesn't help on a site that scores by ASN, because every one of those IPs still looks like a hosting provider. The rotation is real, the cover isn't.

That gives you a rough routing rule that holds across most crawl jobs:

- **Open targets** — public documentation, news archives, your own infrastructure. Datacenter IPs at the lowest per-GB rate you can find.
- **Protected targets** — e-commerce, SERPs, marketplaces, most social platforms. Residential exits, because the ASN reputation is what gets you through, not the IP count.
- **The genuinely hostile ones** — targets that specifically filter for mobile carrier traffic. Mobile IPs, and only where nothing else works, because you're paying a premium.

DataImpulse sells all three tiers from one account, on the same pay-as-you-go balance, which matters more than it sounds. Routing by target typically means routing by vendor, and two vendors means two dashboards, two bills and two sets of expiring credit. One balance you can draw datacenter traffic from on Tuesday and residential from on Wednesday removes that whole category of overhead.

## All DataImpulse plans and current rates

The pricing model is pay-as-you-go per gigabyte. There are no subscriptions and purchased traffic doesn't expire, which changes the arithmetic considerably if your crawl volume is lumpy.

| Proxy type | Entry pack | Standard rate | Bulk rate | Billing | Buy |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 / 5 GB | $1.00/GB | $0.80/GB at 1 TB+ ($800 / 1 TB) | Pay-as-you-go, no subscription, traffic never expires | [ Buy the 5 GB residential test pack](https://bit.ly/dataimPulse) |
| Datacenter | $5 / 10 GB | $0.50/GB | $0.45/GB at 1 TB ($450 / 1 TB); custom from $2,250 at 5 TB+ | Pay-as-you-go, 99.9% uptime | [ Start with datacenter traffic at $0.50/GB](https://bit.ly/dataimPulse) |
| Mobile (4G/5G/LTE) | $5 / 2.5 GB | $2.00/GB | $1.60/GB at 1 TB ($1,600 / 1 TB); custom from $8,000 at 5 TB+ | Pay-as-you-go, carrier IPs | [ Compare mobile proxy rates](https://bit.ly/dataimPulse) |
| Premium residential | $5 / 1 GB | $5.00/GB | Custom volume pricing | Pay-as-you-go, dedicated proxy manager | [ See the premium residential pool](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

A few things about that table that aren't obvious from the numbers.

Country-level targeting is included in the base rate, but city, ZIP and ASN targeting are paid add-ons — and on standard residential, traffic routed through those advanced filters is billed at a higher rate. If your spider needs city-level rotation, budget for it rather than discovering it on the first invoice.

Sticky sessions are supported alongside rotating ones, both over HTTP/HTTPS and SOCKS5. In a Scrapy context this matters for login flows and multi-step wizards, where a mid-session IP change resets your state and kills the crawl. Sticky sessions are time-limited rather than open-ended, so check the current cap before you design a flow around one.

## The number that actually decides your bill

Cost per gigabyte is a headline. Cost per *successful request* is the real one:


successful request cost = (average page size × $/GB) ÷ success rate


At $1/GB, a 500 KB HTML page costs roughly $0.0005 — about $0.50 per thousand requests, before success rate. Divide by 0.95 and you're at roughly $0.53. A million pages a month at half a megabyte each is 500 GB, so about $500. Run the same million through a provider charging $5/GB and it's $2,500 for the same bytes, minus whatever you save on failures.

That last part is the honest complication. A pool at $0.50/GB that gets blocked half the time costs more per usable record than a clean pool at $1/GB. Which is why the $5 test pack is the right first move rather than a bulk purchase: point your actual spider at your actual targets, count successes, and only then decide your tier. Non-expiring traffic makes this cheap to do, because the test gigabytes you don't use this week are still there next month.

## Limits worth knowing before you commit

No proxy provider is the right tool for everything, and it's more useful to say where this one stops.

**There's no managed scraping layer.** No hosted scraper API, no built-in anti-bot handling, no target templates. You bring the code. If your team doesn't write Python, this model means a real learning curve — the middleware approach in this article is the baseline, not an optional extra.

**Pool depth is mid-pack.** One independent test measured around 172,893 live IPs across the US, UK, France, Canada and Germany against a deeper competitor's 306,410, with France the thinnest at 63 carriers. For normal crawl volumes that gap is invisible. At very high volume against defended targets, addresses start repeating. Worth knowing rather than discovering.

**No formal enterprise SLA.** There's 24/7 human support and a dedicated manager on the premium tier, and the published success rate is 99.51%, but this isn't a contract-with-penalties vendor.

**Sticky sessions aren't unlimited**, which rules out the coffee-break-length authenticated flows some providers allow.

If your job is "collect public data at reasonable cost without signing a contract," those limits are mostly irrelevant. If it's "scrape a bank's authenticated dashboard on a compliance timetable," they matter a lot.

## Questions that come up while wiring this together

**Does Scrapy rotate proxies on its own?** No. The built-in proxy middleware assigns one endpoint and leaves it there. Rotation is always something you add: a list plus middleware, a rotating gateway, or both.

**Do I need `scrapy-rotating-proxies` if I'm using a rotating gateway?** Not strictly — the gateway rotates by itself. You still want the middleware's ban detection, otherwise your spider will happily retry a blocked route forever without telling you.

**How many proxies should the list contain?** If you bought static IPs, enough that no single one exceeds the site's tolerance. If you're on a residential gateway, the question doesn't apply — the pool is the provider's, and you're paying per gigabyte instead of per endpoint.

**Rotating or sticky for a paginated crawl?** Rotating, generally. Sticky is for state that has to survive between requests: logins, carts, multi-step forms. Pages that only need a session cookie work fine on rotating exits.

**Why is my throughput lower after switching to a gateway?** Because concurrency limits went from per-domain to per-endpoint, and one endpoint means one queue. Raise `CONCURRENT_REQUESTS` in small steps and watch your block rate — the ceiling you're looking for is the point where success rate starts dropping, not the highest number that runs without crashing.

## What to do next

Get the middleware working against `httpbin.org/ip` until you see ten different exit IPs. Then build the ban policy around the specifics of your target, because no off-the-shelf heuristic knows your site. Then spend $5 rather than $500 and measure cost per successful request on the pages you actually care about.

The setup is maybe forty lines of configuration. The measurement is what separates a spider that scales from one that silently fills a database with captcha pages.

👉 [Follow the DataImpulse Scrapy setup walkthrough](https://dataimpulse.com/blog/scrapy-rotating-proxies/?aff=86938) if you want the gateway endpoints and authentication formats in one place.
