# ScraperAPI Python example: what is the shortest script that works, why do requests return 500, and which plan do you need? (Requests, SDK, proxy port and async code you can copy)

Three lines of Python get ScraperAPI working: send a GET to `https://api.scraperapi.com/` with your `api_key` and the target `url` as query parameters, then read `response.text`. What costs people an afternoon is everything around that call. A timeout shorter than 60 seconds aborts requests ScraperAPI is still retrying internally. Proxy mode throws an SSL error until you pass `verify=False`. Failed requests come back as `500`, and nothing retries them unless you write the loop.

The rest of this page is the code I would hand someone starting today: a minimal script, a retry loop worth shipping, the three ways to connect and when each one is right, thread-based scaling, the async endpoint for long jobs, and the plan facts that decide your bill. One of those facts catches most people out: on Google or Bing, a single request burns 25 of your credits.

> **Start here:** the free plan gives you 5,000 credits for the first 7 days, then 1,000 credits a month with 5 concurrent connections, and no card is required. Every script below runs inside that allowance. Hobby at $49/month is the next step, and you only need it when the 5-connection ceiling or the 1,000-page monthly allowance is what blocks you. 👉 [Check the current plan prices](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI plans: what each tier is for and what it costs

ScraperAPI sells credits, and every plan includes the same core features: JS rendering, premium proxies, JSON auto parsing, custom headers and sessions, desktop and mobile user agents, automatic retries, and unlimited bandwidth. The paid tiers differ in credit allowance and concurrency, so the decision is about volume and parallel threads, not about unlocking features.

| Plan | Best for | Price | What changes | Get started |
| --- | --- | --- | --- | --- |
| Free | Running every script on this page | $0 | 5,000 credits in the first 7 days, then 1,000 credits/month; 5 concurrent connections | [Create a free account](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | One small project, solo | $49/month | First paid tier; auto-upgrade available when credits run out | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | Low-volume production jobs | Scales with credit volume, shown on the pricing page | More credits and more concurrency than Hobby | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | Teams that need country targeting outside the US and EU | $299/month | Country-level geotargeting beyond the US and EU starts here | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | Volume outgrowing Business | Scales with credit volume | Pay-as-you-go with a monthly spending cap available | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | Higher sustained volume | Scales with credit volume | Pay-as-you-go available | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | Heavy, continuous scraping | Scales with credit volume | Pay-as-you-go; plan changes go through sales | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom contracts | Quoted | Volume and concurrency negotiated | [See current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Three billing details matter more than the sticker price:

- **A credit is not a page.** A standard HTML page costs 1 credit. Amazon costs 5. Google and Bing, including subdomains, cost 25. LinkedIn costs 30. Sites behind Cloudflare, DataDome or PerimeterX add 10 credits per request when ScraperAPI has to bypass them.
- **Your 5,000 trial credits are 5,000 ordinary pages.** Point the same trial at Google and you get 200 requests. Point it at LinkedIn and you get about 166. Check the Domain Cost Estimator in the dashboard before you plan a budget, and set `max_cost` on a request if you want a hard ceiling per URL. Responses carry an `sa-credit-cost` header showing what each one actually cost.
- **Hitting your concurrency limit is free but useless.** You get a `429`, the request is turned away rather than queued, and you are not charged. That is the signal to buy more threads, not more credits.

Paid plans carry a 7-day, no-questions-asked refund. Credits do not roll over month to month. On an annual plan you receive the full year's allocation upfront and can draw it down on any schedule.

## The minimal ScraperAPI Python example

python
import requests

API_KEY = "YOUR_API_KEY"
target = "http://quotes.toscrape.com/page/1/"

response = requests.get(
    "https://api.scraperapi.com/",
    params={"api_key": API_KEY, "url": target},
    timeout=70,
)

print(response.status_code)
print(response.text[:500])


That is the whole integration. ScraperAPI handles proxy and header rotation, automatic retries, ban detection and CAPTCHA bypass on its side; you get HTML back as if you had fetched the page yourself. The official quick-start uses `urlencode(params)` and passes the result to `params`, which produces the same request as the dictionary above.

Two settings belong in the first draft, not the debugging session:

**Timeout.** When a response is invalid (a ban, a CAPTCHA, a page that takes too long), ScraperAPI swaps proxy and header configuration and tries again, for up to 60 seconds, before it returns a `500`. Set your client timeout to 70 seconds or leave it unset entirely. `timeout=10` is the single most common reason a script "fails" against a site that would have answered on the second or third internal attempt.

**SSL verification.** In proxy mode, Python Requests must run with `verify=False`, otherwise requests do not complete. This applies to the proxy port method below, not to the API endpoint.

## The retry loop you should actually ship

The quick-start guide reports that over 97% of requests succeed on the first attempt and 99.9% succeed after three retries, unless the site itself is broken. The header comment in the official Selenium sample uses a more conservative figure: 95% first try, 99% after three retries. Treat those as two estimates rather than one number, and set `NUM_RETRIES = 3`.

Failed requests return `500` and are not charged, which is why retrying aggressively is cheap.

python
import csv
import requests
from bs4 import BeautifulSoup

API_KEY = "YOUR_API_KEY"
NUM_RETRIES = 3

urls = [f"http://quotes.toscrape.com/page/{n}/" for n in range(1, 6)]

def fetch(url):
    params = {"api_key": API_KEY, "url": url}
    response = None
    for _ in range(NUM_RETRIES):
        try:
            response = requests.get(
                "https://api.scraperapi.com/",
                params=params,
                timeout=70,
            )
            if response.status_code in (200, 404):
                break
        except requests.exceptions.ConnectionError:
            response = None
    return response

def parse(html):
    soup = BeautifulSoup(html, "html.parser")
    rows = []
    for block in soup.find_all("div", class_="quote"):
        rows.append({
            "quote": block.find("span", class_="text").text,
            "author": block.find("small", class_="author").text,
        })
    return rows

rows = []
for url in urls:
    r = fetch(url)
    if r is not None and r.status_code == 200:
        rows.extend(parse(r.text))

with open("quotes.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["quote", "author"])
    writer.writeheader()
    writer.writerows(rows)

print(len(rows))


The loop breaks on `200` and on `404`. A 404 is a real answer from the target site, so retrying it wastes credits, and the parse step is skipped because there is nothing to parse. Only `500` and connection errors keep the loop going.

## Endpoint, SDK, or proxy port: which one to use

All three send the same request; they differ in how much control you keep and what else is running in your stack.

**API endpoint** is the code above. Use it for scripts you control, one URL at a time. It is the simplest path and the one official examples are built around.

**Python SDK** wraps the same endpoint and puts the retry logic inside the library, defaulting to 3 attempts and accepting an override such as `retry=5`. If you would rather not maintain the loop yourself, this is the reason to pick it. The trade-off is one more dependency for behaviour you can write in fifteen lines.

**Proxy port** is for anything that already speaks HTTP through a proxy, including Selenium and Playwright. Send traffic through `http://scraperapi:YOUR_API_KEY@proxy-server.scraperapi.com:8001` and the target URL never leaves your script:

python
import requests

proxies = {
    "http": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
    "https": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
}

r = requests.get(target, proxies=proxies, verify=False, timeout=70)


For Selenium, the proxy port is the officially recommended route. The repository's API-endpoint Selenium sample opens with a warning that it is *not* the recommended approach. With `seleniumwire`, the setup looks like this:

python
from seleniumwire import webdriver

seleniumwire_options = {
    "proxy": {
        "http": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
        "https": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
        "no_proxy": "localhost,127.0.0.1",
    }
}

driver = webdriver.Chrome(seleniumwire_options=seleniumwire_options)
driver.get("http://quotes.toscrape.com/page/1/")


Selenium will not hand you a response status code, so the official sample parses the performance log for `Network.responseReceived` and reads `status` from it. One more thing to know before you copy that file: it clicks through pages with `driver.find_element_by_link_text("Next →")`, a call Selenium 4 removed. Rewrite it as `driver.find_element(By.LINK_TEXT, "Next →")` if your version is current.

## Scaling with threads is a plan decision, not a code decision

Concurrency is the product. Going from a few hundred pages a day to millions is a matter of raising the concurrent thread limit on your plan; the code does not change shape.

python
from concurrent.futures import ThreadPoolExecutor

NUM_THREADS = 5  # keep at or below your plan's concurrency limit

with ThreadPoolExecutor(max_workers=NUM_THREADS) as pool:
    results = list(pool.map(fetch, urls))


`NUM_THREADS = 5` matches the free plan's ceiling exactly, so it is a safe starting point. Set it above your plan's limit and the extra requests come back as `429` and are discarded, which is why "add more threads" and "upgrade the plan" have to happen together. If your threads sit pinned at the limit every day, the next tier is the fix, and the credit balance is a secondary question. 👉 [Compare concurrency limits across plans](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## When jobs are long or the site is slow: the async API

The synchronous endpoint holds your connection while ScraperAPI works. For batch jobs, slow targets or anything you want to fire and collect later, submit the job instead.

python
import time
import requests

job = requests.post(
    "https://async.scraperapi.com/jobs",
    json={"apiKey": API_KEY, "url": "https://example.com"},
).json()

while True:
    result = requests.get(job["statusUrl"]).json()
    if result["status"] == "finished":
        break
    time.sleep(5)

print(result["response"]["statusCode"])
print(result["response"]["body"][:500])


The POST returns an `id`, a `status` of `running`, and a `statusUrl` that is personal to that job. Poll it until `status` flips to `finished`, at which point the payload contains `response.headers`, `response.body` and `response.statusCode`. The headers include `sa-final-url` and `sa-statuscode`, which tell you where the request actually landed and what the target returned.

Async results are stored for up to 72 hours, with 24 hours guaranteed, or until you retrieve them, whichever comes first. Miss the window and the job is deleted; you resubmit and pay again. If a job matters, persist the body as soon as it finishes. The same endpoint accepts POST targets via `"method": "POST"` and a `"body"` field such as `"var1=value1&var2=value2"`.

## Two things that break on real pages

**JavaScript-rendered sites.** If the HTML you get back is an empty shell, add the render flag:

python
params = {"api_key": API_KEY, "url": target, "render": "true"}


That runs the page through a headless browser before returning it, and costs more credits than a plain fetch. Try without it first.

**HTML tables.** Tables look easy until a class selector misses. On the DataTables stripe example, the table's class is `stripe` only; adding `dataTable` matches nothing, while `id="example"` also works. The page holds 57 rows, all present in the source, so no AJAX handling is needed: `soup.find("table", class_="stripe")`, then `tbody`, then `tr`, then `td` per row. Tables with two-level headers need `table.select("thead tr:nth-of-type(1) th")` and `nth-of-type(2)` combined into one header list before you build a DataFrame. For tables split across pages, ScraperAPI's Render Instruction Set handles the clicking without a headless browser of your own.

## FAQ

**Do failed requests cost credits?** No. A `500` from ScraperAPI is not billed, and only successful responses (`200` and `404`) are charged. This is why three retries per URL is the recommended default rather than a wasteful habit.

**Should I set a timeout?** Either leave it out or set it to at least 60 seconds, since ScraperAPI can spend that long cycling proxies before giving up. 70 seconds is a comfortable ceiling.

**Is there a file size limit?** Images, PDFs and other files are fetched like any other URL, with a 2MB limit per request.

**Why does my proxy-mode request raise an SSL error?** Because `verify=False` is mandatory in that mode. It is not needed for the API endpoint.

**Why does Selenium return no data?** Usually because it was pointed at the API endpoint instead of the proxy port. Switch to `seleniumwire` and port 8001, and remember to read status codes from the performance log.

## Which plan to buy

Work down this list and stop at the first line that fits:

- **Just verifying your script works:** free plan, 5,000 credits for 7 days, 5 threads.
- **One project, under 1,000 standard pages a month, under 5 parallel requests:** free plan is enough after the trial.
- **Credit balance runs dry mid-month, or threads sit at the limit:** Hobby at $49/month, then step up a tier whenever the allowance or the concurrency, not the feature list, becomes the constraint.
- **You need country-level targeting outside the US and EU:** that starts at Business, $299/month.
- **You are scraping Google, Bing or LinkedIn at volume:** price the job in credits first, because 25 and 30 credits per request turn a modest page count into a large bill.

Two things to confirm before you pay: run your target domains through the Domain Cost Estimator so the credit maths is real, and check the plan's concurrency against the thread count in your script. The 7-day refund means a wrong first choice is recoverable, but credits do not roll over, so an annual plan is a commitment to use the allocation within the year.

👉 [Open the pricing page and pick the smallest plan that clears your concurrency limit](https://www.scraperapi.com/pricing/?fp_ref=coupons)
