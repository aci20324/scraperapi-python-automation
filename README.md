# How to Use ScraperAPI with Python: The Complete Developer Guide from Signup to Scaling — API Endpoint, Python SDK, JS Rendering, Premium Proxies, Async Scraping, and Structured Amazon/SERP Data Explained (With Plan Pricing and Free Trial Tips)

If you've ever sat down to scrape a few hundred pages and watched your script die on the 23rd request with a 403, a Cloudflare interstitial, or one of those smug "Pardon the interruption" CAPTCHAs — you already know why this article exists. Learning **how to use ScraperAPI with Python** is the shortest path between "I just want the data" and actually having the data, without running your own proxy pool, headless browser farm, or a CAPTCHA-solving side business.

This guide walks you through everything you'll touch in a real project: signing up, picking the right integration method, writing working Python code for both raw HTML and structured JSON, scaling to concurrent threads, avoiding the credit-multiplier traps that quietly double your bill, and choosing a plan that matches your volume. We'll keep it practical. No marketing fluff, no invented discounts, just the code and the numbers.

---

## Why Developers Reach for ScraperAPI in the First Place

Most Python scraping tutorials start with `requests.get(url)` and end the moment the target site notices you. The boring truth about scraping in 2026 is that the hard part is no longer parsing HTML — BeautifulSoup and `lxml` solved that years ago. The hard part is everything around the request:

- IP rotation, so a single datacenter IP doesn't get rate-limited after 50 hits.
- Header and fingerprint rotation, because modern anti-bot systems read TLS fingerprints, not just User-Agent strings.
- JavaScript rendering, for the large chunk of the web that ships an empty `<div id="root">` and paints everything client-side.
- CAPTCHA solving, including the newer Turnstile, DataDome, and PerimeterX challenges.
- Retries, timeouts, and geographic targeting.

ScraperAPI wraps all of that behind one endpoint. You send a URL, optionally a few flags (`render=true`, `premium=true`, `country_code=us`), and the API returns either raw HTML or, for supported domains, structured JSON. The company advertises being trusted by 10,000+ companies with a 99.9% uptime guarantee, and Trustpilot reviews sit at a 4.5/5 across 40+ reviews, with users repeatedly calling out how easy the initial setup is.

> "ScraperAPI was extremely easy to use out of the box. We are able to get around website blocks easily." — Trustpilot reviewer

The trade-off, and we'll dig into this later, is the credit-multiplier pricing model: a "100,000 credits" plan does not mean 100,000 requests. Read the credit section carefully before you pick a tier.

---

## Step 1 — Create Your Account and Grab an API Key

ScraperAPI gives every new signup a **7-day trial with 5,000 free API credits, no credit card required**. After the trial expires, you drop onto a small recurring **1,000 credits/month free plan** (capped at 5 concurrent connections) — enough for light hobby scraping, but not enough for any serious project.

To get started:

1. Head to the 👉 [ScraperAPI signup page](https://www.scraperapi.com/?fp_ref=coupons) and create an account.
2. Confirm your email and land on the dashboard.
3. Copy your `api_key` from the dashboard home — you'll paste it into every code example below.

That's the entire onboarding. There is no sales call, no credit-card gate, no "contact us for pricing" wall in front of the trial.

---

## Step 2 — Pick Your Integration Method

ScraperAPI exposes three integration paths. All three hit the same backend and consume the same credits; the difference is developer ergonomics.

### Method A — The Raw REST API Endpoint (Most Flexible)

This is what most tutorials show because it works with anything that can make an HTTP request: `requests`, `httpx`, `aiohttp`, `curl`, Postman, even `urllib`. The base URL is `https://api.scraperapi.com/`, and the only required parameters are `api_key` and `url`.

python
import requests

target_url = 'https://www.example.com'
api_key = 'YOUR_API_KEY'

request_url = f'https://api.scraperapi.com?api_key={api_key}&url={target_url}'
response = requests.get(request_url)

print(response.status_code)
print(response.text)


That's the entire "hello world". ScraperAPI picks the proxy, sets the headers, retries on bans, and hands you back the HTML.

### Method B — The Official Python SDK (Built-in Retries)

If you'd rather not write your own retry loop, install the SDK:

bash
pip install scraperapi-sdk


The SDK handles retries for you (default 3, configurable) and gives you clean method calls:

python
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient("YOUR_API_KEY")

# Plain GET
content = client.get('https://example.com/')

# GET with premium residential proxies
content = client.get('https://amazon.com/', params={'premium': True})

# POST and PUT also supported
content = client.post(
    'https://webhook.site/your-endpoint',
    headers={'Content-Type': 'application/x-www-form-urlencoded'},
    data={'field1': 'data1'}
)


The SDK also throws a clean `ScraperAPIException` with an `.original_exception` attribute, which is genuinely useful for logging without losing the underlying error.

### Method C — Proxy Port Mode (Drop-in for Existing Code)

If you already have a scraper that uses `proxies={'http': ..., 'https': ...}` in `requests.get()`, you can point it at ScraperAPI's proxy port and change nothing else. The main caveat: when using proxy mode, set `verify=False` so SSL cert verification doesn't break the request, and don't set a timeout shorter than 60 seconds — ScraperAPI's internal retry cycle can take up to 60 seconds to keep trying fresh proxy/header combinations before it returns a 500.

---

## Step 3 — The Four Parameters That Actually Matter

Most "stuck" scraping jobs come down to one missing flag. The full parameter list lives in the docs, but 90% of the time you need one of these four:

| Parameter | What it does | When to use it | Credit cost |
|---|---|---|---|
| `render=true` | Loads the page in a headless browser, waits for JS to execute | Target is a React/Vue/Angular SPA, content loads via XHR | 10 credits/request |
| `premium=true` | Routes through high-quality residential proxies instead of datacenter IPs | Target is geo-locked or aggressively blocks datacenter ranges | 10 credits/request |
| `country_code=us` | Forces exit IP to a specific country | Site shows different prices/content per region | 0 (free) |
| `session_number=123` | Sticks all requests with the same session number to the same IP | You need a logged-in session or cart to persist across requests | 0 (free) |

Stack them and the costs add up fast: `render=true` plus `premium=true` costs **25 credits per request**, and `ultra_premium=true` plus `render=true` hits **75 credits**. We'll come back to that math in the credit section because it's the single biggest surprise on the bill.

A working example with rendering and premium proxies together:

python
import requests

target_url = 'https://example.com'
api_key = 'YOUR_API_KEY'

request_url = (
    f'https://api.scraperapi.com?api_key={api_key}'
    f'&render=true&premium=true&url={target_url}'
)
response = requests.get(request_url)
print(response.text)


One ordering rule ScraperAPI repeats in its docs: **list all ScraperAPI parameters before the `url` parameter**, otherwise the API may confuse them with query params already attached to your target URL.

---

## Step 4 — Build a Real Scraper with Retries and Concurrency

The official quick-start guide ships a pattern worth memorising. It does three things: sends the request through ScraperAPI, retries up to 3 times on failure (500 status codes are not charged), and parses the result with BeautifulSoup. Here's the distilled version:

python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'
NUM_RETRIES = 3

list_of_urls = [
    'http://quotes.toscrape.com/page/1/',
    'http://quotes.toscrape.com/page/2/',
]
scraped_quotes = []

for url in list_of_urls:
    params = {'api_key': API_KEY, 'url': url}
    for _ in range(NUM_RETRIES):
        try:
            response = requests.get(
                'http://api.scraperapi.com/',
                params=urlencode(params)
            )
            if response.status_code in [200, 404]:
                break
        except requests.exceptions.ConnectionError:
            response = ''

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        for quote_block in soup.find_all('div', class_='quote'):
            scraped_quotes.append({
                'quote': quote_block.find('span', class_='text').text,
                'author': quote_block.find('small', class_='author').text,
            })

print(scraped_quotes)


For production, you almost never want to scrape serially. ScraperAPI's plans are gated by **concurrent threads**, not just total credits, so a Hobby plan with 20 threads can run 20 requests in parallel. Use `concurrent.futures`:

python
import concurrent.futures

NUM_THREADS = 5  # bump this up to match your plan's concurrency cap

with concurrent.futures.ThreadPoolExecutor(max_workers=NUM_THREADS) as executor:
    executor.map(scrape_url, list_of_urls)


If you're on a higher tier and comfortable with async, swap `requests` for `aiohttp` and use `asyncio.gather()` for the same effect with less thread overhead.

---

## Step 5 — Skip the Parsing: Structured Data Endpoints

This is the feature that pulls people from raw HTML scraping into ScraperAPI's paid tiers. For high-value domains — Amazon, Google SERP, Google News, Google Jobs, Google Shopping, eBay, Walmart, Redfin — ScraperAPI maintains dedicated structured endpoints that return clean JSON instead of HTML. You skip BeautifulSoup entirely.

The Python SDK exposes them as namespaced methods:

python
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient('YOUR_API_KEY')

# Amazon product by ASIN
product = client.amazon.product('B0CHVR5K7C')

# Amazon search
results = client.amazon.search('iphone 15 charger')

# Amazon price lookup for multiple ASINs at once
prices = client.amazon.prices(['B0CHVR5K7C', 'B00CL6353A'])

# Google SERP
serp = client.google.search('best mechanical keyboard')

# Google News
news = client.google.news('tornado')


Each method takes optional `country` and `tld` kwargs (e.g. `country='uk', tld='co.uk'`) for region targeting. Independent benchmarks put ScraperAPI's Amazon structured endpoint at around a 98% success rate — meaning if your job is monitoring competitor prices on Amazon, the structured endpoint is more reliable than hand-rolling a BeautifulSoup scraper and praying Amazon doesn't change their DOM next week.

For the raw REST equivalent, the Amazon search endpoint looks like this:

python
import requests

payload = {
    'api_key': 'YOUR_API_KEY',
    'query': 'iphone 15 charger',
    's': 'price-asc-rank'  # Amazon sort: price-asc-rank, price-desc-rank, review-rank, date-desc-rank
}
response = requests.get(
    'https://api.scraperapi.com/structured/amazon/search',
    params=payload
)
print(response.json())


### Markdown Output for LLM Pipelines

A newer trick worth knowing: pass `output_format=markdown` and ScraperAPI returns the page as clean Markdown instead of HTML. This is the fastest way to feed live web data into an LLM without writing custom parsing logic.

python
import requests
import google.generativeai as genai

payload = {
    'api_key': 'YOUR_API_KEY',
    'url': 'https://www.amazon.com/s?k=iphone+15+charger',
    'output_format': 'markdown'
}
markdown_data = requests.get(
    'https://api.scraperapi.com',
    params=payload
).text

genai.configure(api_key='YOUR_GEMINI_KEY')
model = genai.GenerativeModel('gemini-2.0-flash')

prompt = f"Summarise the top 5 chargers under $20 from this data:\n{markdown_data}"
print(model.generate_content(prompt).text)


One API call, one LLM call, zero BeautifulSoup.

---

## Step 6 — Async Scraping for Scale

Once you move past a few thousand pages, synchronous scraping bottlenecks on network wait time even with thread pools. ScraperAPI's async client is built for this: you submit a job, get back a `request_id`, then either poll for the result or have ScraperAPI POST the result to a webhook when it's done.

python
from scraperapi_sdk import ScraperAPIAsyncClient, ScraperAPIException

client = ScraperAPIAsyncClient('YOUR_API_KEY')

# Submit the job
try:
    job = client.create('https://example.com')
    request_id = job.get('id')
except ScraperAPIException as e:
    print(e.original_exception)

# Option A: poll-and-wait (blocking)
result = client.wait(
    request_id,
    cooldown=5,
    max_retries=10,
    raise_for_exceeding_max_retries=False
)

# Option B: webhook callback (non-blocking)
client.create(
    'https://example.com',
    webhook_url='https://your-app.com/scraperapi-webhook'
)


The async client supports the same structured endpoints as the sync client: `client.amazon.product(...)`, `client.google.search(...)`, etc., all return job IDs you can collect in bulk.

---

## Step 7 — Understanding Credits Before You Pick a Plan

This is where most new users get burned, so read it twice. ScraperAPI bills on **API credits**, not requests, and the per-request credit cost scales with how hard the request was:

| Request type | Credits per request |
|---|---|
| Standard request (no flags) | 1 |
| `premium=true` only | 10 |
| `render=true` only | 10 |
| `screenshot=true` | 10 |
| `premium=true` + `render=true` | 25 |
| `ultra_premium=true` only | 30 |
| `ultra_premium=true` + `render=true` | 75 |
| Cloudflare / Turnstile / DataDome / PerimeterX bypass | 10 |

On top of that, three "hard target" domains carry fixed multipliers regardless of flags:

| Domain | Credits per request |
|---|---|
| Amazon (e-commerce) | 5 |
| Google / Bing SERP | 25 |
| LinkedIn | 30 |

Two worked examples that make the math concrete:

- **Hobby plan, scraping JS-rendered pages**: 100,000 credits ÷ 10 credits per `render=true` request = **10,000 rendered pages**. Effective cost ≈ $0.0049 per rendered scrape.
- **Business plan, scraping Google SERPs**: 3,000,000 credits ÷ 25 credits per Google SERP = **120,000 SERP requests**. Effective cost ≈ $0.0025 per SERP pull.

The headline credit count is generous on paper and considerably smaller in practice. Use the `urlcost` API endpoint (`https://api.scraperapi.com/account/urlcost?api_key=...&url=...&render=true`) to programmatically check the credit cost of any URL before you scrape it at scale, and watch the `sa-credit-cost` response header on every request to confirm what you were actually charged.

---

## Full Plan Comparison Table (All Tiers, Current Pricing)

Every plan below includes the same core features: JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, CAPTCHA/anti-bot handling, and unlimited bandwidth. Annual billing takes **10% off every paid tier**.

| Plan | Price (monthly) | Price (annual, billed yearly) | API Credits / month | Concurrent Threads | Geotargeting | Pay-As-You-Go Overage | Best For | Purchase |
|---|---|---|---|---|---|---|---|---|
| Free | $0 | $0 | 1,000 (after trial) | 5 | US & EU | No | Testing, learning, tiny hobby projects |  [Start free trial](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49/mo | $44/mo | 100,000 | 20 | US & EU | No — must upgrade when credits run out | Personal projects, small scraping jobs |  [Get Hobby plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149/mo | ~$134/mo | 1,000,000 | 50 | US & EU | No | Low-volume production scraping |  [Get Startup plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299/mo | ~$269/mo | 3,000,000 | 100 | Global (country-level) | No | Production-grade scraping at moderate scale; first tier with global geo + unlimited analytics |  [Get Business plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475/mo | ~$428/mo | 5,000,000 | 200 | Global | Yes — first tier with PAYG overage | Growing scraping operations (marked "Most popular") |  [Get Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975/mo | ~$878/mo | 10,500,000 | 300 | Global | Yes | Recurring, high-volume scraping; priority support |  [Get Professional plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975/mo | ~$1,778/mo | 21,500,000 | 500 | Global | Yes | Continuous, multi-source data pipelines; priority routing |  [Get Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | Global | Yes | Dedicated support team + Slack channel; complete control and personalised pricing |  [Contact sales for Enterprise](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

A few non-obvious rules attached to those tiers:

- **Lower tiers force a hard upgrade** when credits run out. Hobby, Startup, and Business customers must move up a tier; there's no safety-valve PAYG.
- **PAYG is gated to Scaling and above**, with an optional monthly spending cap you set in the dashboard so overage can't surprise-bill you.
- **Annual plans** are not a "use it or lose it" pool — they're a single 12-month credit allotment. If you burn through it before the year ends, you can either re-subscribe to start a fresh 12-month cycle or jump to a higher monthly plan.
- **Refunds**: 7-day no-questions-asked policy. Cancel anytime from the dashboard, no termination fee.

---

## Step 8 — Common Pitfalls (Read Before You Launch)

A short list of mistakes that show up repeatedly in user reviews and support tickets:

1. **Setting a timeout under 60 seconds.** ScraperAPI's internal retry cycle can run up to 60 seconds before it returns a 500. If your client times out at 30 seconds, you'll get false negatives on perfectly good requests. Either don't set a timeout, or set it to 60+.

2. **Reading "100,000 credits" as "100,000 requests."** It almost never is. Estimate your real volume using the multiplier table above before you commit to a tier.

3. **Forgetting the SSL flag in proxy mode.** If you're using the proxy port integration, your code must skip cert verification (`verify=False` in `requests`), or the request will fail before it ever reaches ScraperAPI's backend.

4. **Hammering the same domain on a low concurrency tier.** Hobby's 20-thread cap is a hard ceiling. If you're scraping Amazon at scale, you'll want at least Business (100 threads) or Scaling (200) — not because of credits, but because of throughput.

5. **Ignoring the `sa-credit-cost` response header.** Every API response includes this header showing the exact credit cost of that single request. Logging it is the cheapest way to catch an unexpected multiplier before it eats your monthly budget.

---

## Step 9 — Pairing the Right Plan with the Right Use Case

Most readers land on one of three archetypes:

- **"I'm a student or building a side project."** Start with the 👉 [free trial](https://www.scraperapi.com/?fp_ref=coupons) (5,000 credits for 7 days), then fall onto the 1,000-credit/month free plan. When you outgrow that, Hobby at $49/mo is the natural next step — just remember that with `render=true` you're really buying 10,000 rendered pages, not 100,000.

- **"I'm running production scraping for a small team."** Business at $299/mo is the first tier that unlocks **global country-level geotargeting** and **unlimited analytics history**. If you're doing anything e-commerce — Amazon price monitoring, SERP rank tracking for clients — this is usually the right starting point.

- **"I'm scraping at scale and can't afford downtime."** Scaling ($475/mo) is the cheapest tier with **pay-as-you-go overage**, which means a traffic spike won't kill your job mid-cycle; it just bills you at a fixed per-credit rate. Professional and Advanced layers add priority support and priority routing on top. For multi-million-credit monthly volumes, 👉 [Enterprise](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) gets you a dedicated Slack channel and account manager.

If you're unsure, the trial is genuinely free and the 7-day refund window means a paid plan is reversible. Test against your real target URLs before committing.

---

## What Users Actually Say

Public reviews cluster around a few consistent themes. On the positive side, developers repeatedly call out the **dead-simple API surface** and the **reliability on supported targets** — Amazon and Google SERP scrapes in particular get cited as "just works" use cases. On the critical side, the recurring complaints are exactly the credit-multiplier surprise and the lack of a permanent free tier beyond the 1,000-credit/month allowance. A representative thread from third-party review sites:

> Setup was straightforward and the structured Amazon endpoint worked first try. The catch was that I burned through my Business plan credits faster than expected because I had `render=true` on by default — turns out that's 10× per request. Check the multiplier table before you commit to a tier.

That's not a bug; it's the model. Plan for it.

---

## Final Checklist Before You Ship

Run through this before pointing any production scraper at ScraperAPI:

- [ ] API key stored in an environment variable, not hardcoded.
- [ ] Timeout set to ≥ 60 seconds (or unset).
- [ ] Retry loop in place (3 retries minimum; the SDK does this by default).
- [ ] `concurrent.futures` or `asyncio` pool size matched to your plan's thread cap.
- [ ] `sa-credit-cost` response header logged for at least the first 100 requests.
- [ ] Credit multiplier estimated for your real target mix (especially if you're using `render=true` or hitting Amazon/Google/LinkedIn).
- [ ] Monthly spending cap set in the dashboard if you're on a PAYG-eligible tier.

If you're ready to test it on your own URLs, the fastest way in is the 👉 [ScraperAPI free trial](https://www.scraperapi.com/?fp_ref=coupons) — 5,000 credits, 7 days, no card. Run your real workload against it, watch the credit headers, and the right plan will reveal itself from the data.
