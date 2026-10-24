# rotating proxies for scraping: how IP rotation works, what ScraperAPI really charges per request, and how to pick a plan without burning credits

You're probably here because a scraper started returning 403s, CAPTCHAs, or the same page over and over, and someone told you the fix is "rotating proxies." The short version: a rotating proxy service gives you one endpoint, and behind that endpoint a pool of IPs takes turns making your requests, so no single IP hammers a site hard enough to trigger rate limits. ScraperAPI is one of the better-known versions of this idea — it bundles rotation, JavaScript rendering, and CAPTCHA handling into a single API — and its plans start at $49/month with a 5,000-credit free trial.

But the part most articles skip is the billing. ScraperAPI charges by credits, and the credit cost of a single request can swing from 1 to 75 depending on your target site and parameters. That difference decides which plan you should actually buy, so we'll walk through the math before anything else matters.

> **Quick take:** Most people scraping e-commerce and search pages at small-to-mid volume do fine on the **Hobby plan ($49/month, 100,000 credits)**, or the Startup plan if they render JavaScript on protected pages. There's a 7-day free trial with 5,000 credits and no credit card required, which is genuinely the cheapest way to find out what your target sites cost per request. 👉 [Check ScraperAPI's current plans and prices](https://www.scraperapi.com/?fp_ref=coupons)

## What a rotating proxy actually does

A rotating proxy sits between your scraper and the target site. Instead of your script connecting to, say, amazon.com directly, it sends the request to your provider's endpoint. The provider forwards it using one IP from its pool, and the next request can come from a completely different IP.

There are two ways rotation gets implemented, and knowing the difference saves you from a common setup mistake:

1. **Per-request rotation** — every single request gets a fresh IP. This is what you want for independent page fetches: product listings, search results, price checks. Nothing needs to "remember" the previous request.

2. **Sticky sessions** — the same IP stays pinned to your session for a stretch of requests. You need this for anything that has to look like one continuous user: paginating through search results, keeping a cart state, or anything behind a login-style flow. ScraperAPI handles this with a `session_number` parameter instead of making you juggle multiple endpoints.

Rotation is the default choice for scraping at scale for one boring reason: spreading requests across many IPs is what keeps you under per-IP rate limits. If a site allows ~1 request per second per IP, ten rotating IPs get you roughly ten times the throughput without changing anything about your code.

## ScraperAPI's full plan lineup

Here's everything currently on the pricing page, in one table. Prices are monthly billing; annual billing knocks roughly 15–20% off.

| Plan | Price | Credits / month | Concurrent threads | Best for | Link |
| --- | --- | --- | --- | --- | --- |
| Free trial | $0 for 7 days | 5,000 total | Limited | Testing your actual target sites before paying | [ Start the free trial](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49/mo | 100,000 | 20 | Solo developers, small scrapers, side projects | [ Get Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | $149/mo | 1,000,000 | 50 | Growing scrapers, JS-heavy or protected targets | [ Get Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | $299/mo | 3,000,000 | 100 | High-volume production jobs, teams | [ Get Business](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom | Custom | Custom | Dedicated account management, custom concurrency | [ Contact sales](https://www.scraperapi.com/?fp_ref=coupons) |

The trial needs no credit card, so you can point it at your real targets and measure credit burn before spending anything. That's worth doing before you even look at the pricing tiers, because — as the next section shows — the number of pages you get per plan depends heavily on where you're scraping.

## The credit math most reviews gloss over

This is the part that trips people up, so let's be specific. ScraperAPI doesn't bill per page. It bills per request, and the base cost changes by domain category and by the parameters you enable.

| Request type | Credit cost |
| --- | --- |
| Plain request (normal sites) | 1 |
| E-commerce domains (Amazon, Walmart, eBay) | 5 |
| JavaScript rendering (`render=true`) | +10 |
| Premium proxy (`premium=true`) | +10 |
| Screenshot | +10 |
| Ultra premium proxy | +30 |
| `premium=true` + `render=true` combined | 25 |
| `ultra_premium=true` + `render=true` combined | 75 |
| Cloudflare / DataDome / Turnstile bypass | 10 each |

Two things in that table deserve a second look.

First, feature stacking isn't linear. Premium proxy plus JS rendering should logically cost 20 credits; ScraperAPI charges 25. Ultra premium plus rendering comes to 75 — nearly double the naive 40. This non-linear stacking is the main reason credits seem to evaporate faster than expected on protected targets.

Second, e-commerce pages cost 5x a plain request before you touch any parameters. An Amazon product page with rendering and premium proxies runs 25 credits. Do the arithmetic on the Startup plan: 1,000,000 credits ÷ 25 = 40,000 actual pages. A million-credit plan does not mean a million pages unless you're scraping plain HTML from friendly sites.

Before you scale anything, ScraperAPI gives you two ways to check the exact cost of a request in advance: a Domain Multiplier tool in the dashboard where you paste any URL and see its credit cost, and a `urlcost` API endpoint that accepts the same parameters as a normal request. The CLI has this too — `sapi cost "https://www.amazon.com/dp/B0CXXXXZ8" -render -premium` prints the cost (25 credits in their example) before you spend anything. Run this on your five most important URLs on day one of the trial and you'll know precisely which plan fits.

## What costs you nothing extra

The flip side of the multiplier table is a list of parameters ScraperAPI charges zero credits for:

- `country_code` — geotargeting your requests to a specific country
- `session_number` — sticky sessions, i.e., keeping the same IP across a request sequence
- `device_type` — mobile vs desktop fingerprints
- `autoparse` — automatic structured-data extraction where supported
- `keep_headers` — forwarding your original headers
- `wait_for_selector` — waiting for a specific element before returning HTML

The sticky-session one matters most in practice. A lot of proxy providers charge session management as a premium feature or cap it aggressively on cheaper tiers. Here it's a free parameter, which means paginating through search results or holding a browsing session costs the same as independent page fetches.

Billing also only counts successful outcomes. Requests are charged for `200` and `404` status codes; everything else — timeouts, blocks that the retry layer couldn't fix — costs nothing. One caveat from the docs: if you cancel a request from your side before the 70-second processing window finishes, that can still count as a charge. Build your timeouts with that in mind.

## When ScraperAPI is the wrong tool

Honesty requires saying this plainly: for heavily protected targets, ScraperAPI's credit model can cost more than bandwidth-billed residential proxies.

If you're scraping LinkedIn, Instagram, or sites behind aggressive bot management at real volume, the ultra-premium multipliers add up fast. A 2026 review of the service made the same point — for hard targets, a residential proxy plan billed by bandwidth often works out cheaper, and if you need sessions pinned far beyond an hour (ad verification, long account workflows), you want a dedicated residential or mobile provider instead.

The competitive landscape, briefly and without links — these are their published entry prices:

| Alternative | Entry price | Model |
| --- | --- | --- |
| ScrapingDog | $40/mo | 200,000 credits; failed requests never charged |
| Scrapfly | $30/mo | Prepaid credits; failed requests never charged |
| Apify | $29/mo | Plus $0.20 per compute unit; marketplace of ready-made scraper "Actors" |
| ScrapingBee | $49/mo | 250,000 credits |
| Oxylabs (Web Scraper API) | from $49/mo | Usage-tiered; SOC 2 / ISO 27001 compliance pitch |
| Bright Data (Web Scraper API) | ~$499/mo | Per-record PAYG from $1.50/1,000 records; biggest proxy network |

Where ScraperAPI holds its ground: it's the simplest drop-in. One endpoint, no zone configuration, no proxy pool management, and the network behind it is 40M+ IPs across 50+ countries. Against Bright Data's typical $499/month minimum for the Web Scraper API, ScraperAPI's $49 entry point is in a different weight class for small and mid-size projects. Against the cheaper per-credit rivals, its differentiator is the free trial length (7 days, 5,000 credits, no card) versus their one-time 1,000–200-credit tasters, plus the free sticky sessions and geotargeting noted above.

The realistic recommendation: use the trial to benchmark. If your targets clear on default proxies, ScraperAPI is likely the cheapest fully-managed option at your volume. If everything needs `ultra_premium`, price a bandwidth-billed residential plan against your actual credit burn before committing.

## Getting started without wasting credits

1. **Sign up for the trial** — 5,000 credits, 7 days, no credit card.
2. **Cost-check your target URLs** first, using the dashboard's Domain Multiplier or the `urlcost` endpoint. Multiply by your expected monthly volume to pick a plan.
3. **Test with the cheapest settings that work.** Many standard e-commerce and search targets clear without `premium` at all; only escalate to paid multipliers when a site actually blocks you.
4. **If you work from the terminal**, ScraperAPI ships a CLI (`sapi`) that wraps the API: `sapi scrape "URL" -render -autoparse | jq '{title, price}'` returns parsed JSON in one line, `sapi cost` previews credit spend, and `sapi jobs batch urls.txt` fans out up to 50,000 URLs with retries handled. It's the shortest path from "URL list" to clean data, with no SDK code.

If you want to compare plans against your own volume numbers before the trial clock runs out, 👉 [see the full plan breakdown on ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons).

## FAQ

**Do I need to rewrite my scraper to use rotating proxies?**
No. With a provider-managed service like ScraperAPI you change one thing: the base URL of your requests. Your parsing code, retries, and everything else stay the same, because rotation happens behind the endpoint.

**Are failed requests charged?**
No. ScraperAPI only bills successful responses — `200` and `404` status codes. A request that times out or gets blocked without resolution costs zero credits.

**Is there a discount for paying annually?**
Yes — annual billing reduces the monthly rate by roughly 15–20% versus month-to-month pricing.

**How do I know how many credits a URL costs before running a big job?**
Three ways: the Domain Multiplier in your dashboard, the `urlcost` API endpoint (it accepts all the same parameters as a scrape request), or `sapi cost` from the CLI.

**Can I keep the same IP for a sequence of requests?**
Yes, via the `session_number` parameter, and it costs no extra credits — which is unusual for this category.

## Picking a plan, concretely

- **Just testing an idea or scraping plain-HTML sites:** the 5,000-credit free trial will tell you everything, and Hobby at $49/month covers roughly 100,000 plain requests or 20,000 Amazon pages.
- **Rendering JavaScript on moderately protected pages:** Startup at $149/month. At 25 credits per rendered-plus-premium request, that's about 40,000 heavy pages monthly.
- **High-volume production or team use:** Business at $299/month, 3,000,000 credits, 100 concurrent threads.
- **LinkedIn, Instagram, or aggressively protected targets at scale:** benchmark carefully with the trial first, and compare against bandwidth-billed residential providers — the credit multipliers can make this more expensive than it looks.

One last thing to check before you commit to any paid tier: run your five most important URLs through the cost preview with the exact parameters you'll use in production. Two minutes of math there is the difference between a plan that lasts the month and one that dies mid-job. 👉 [Start ScraperAPI's free trial and cost-check your own targets](https://www.scraperapi.com/?fp_ref=coupons)
