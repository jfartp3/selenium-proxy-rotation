# selenium rotating proxy: wiring authenticated IP rotation into Python scripts without burning your per-GB budget

A Selenium scraper rarely dies on the first request. It dies somewhere around request four hundred, when the logs stop showing product pages and start showing 403s, 429s, and CAPTCHA walls. Nothing in the code changed — the target site simply noticed that a lot of traffic was coming from one address and decided to slow it down.

Proxy rotation is the standard fix, but Selenium makes it more annoying than it should be. The obvious approach of stuffing `user:password` into `--proxy-server` doesn't work, because ChromeDriver ignores credentials in that flag. You don't get an error message explaining that. You get a Windows-style auth popup, or a `407 Proxy Authentication Required` buried in a stack trace.

This walks through the rotation approaches that actually work with a headless browser, how to handle proxy authentication, when rotating per-request is the wrong strategy, and what that traffic costs in practice with DataImpulse.

## Why an IP change alone won't save your scraper

Rotating proxies gets treated as a complete anti-blocking solution. It isn't. The target evaluates several things at once:

- **IP reputation and origin.** Residential IPs sit in consumer pools, so a request looks less synthetic than one from a cloud subnet.
- **Fingerprint consistency.** If your `User-Agent` says Chrome 124 but `sec-ch-ua` says 108, that mismatch is a signal on its own, even without knowing you're running automation.
- **Request cadence.** Ten requests per second from a fresh IP still looks like a machine.
- **Session coherence.** Jumping from a Tokyo exit to a Toronto exit mid-flow reads less like a returning user and more like a VPN switch.

That last point matters with a headless browser specifically, because Selenium carries cookies and local storage between page loads. If you rotate the IP but keep the cookie jar, you're producing an identity that no real browser would ever produce.

The practical rule: rotate for broad, anonymous collection. Keep the same exit for multi-step flows, logins, and carts. Match the IP's country to your `Accept-Language` header and the locale the page expects — a US IP requesting pages with `Accept-Language: de-DE` is a silent quality problem that shows up as wrong prices rather than as a block.

## Three ways to rotate proxies in Selenium, and where each one breaks

### 1. Random pick from a local proxy list

This is the pattern in most tutorials, and it's the one you'll outgrow fastest. You keep a list, shuffle it, and hand one entry to Chrome:

python
import random
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

proxy_pool = ["191.96.100.33:3155", "167.86.115.218:8888", "20.205.61.143:80"]

def driver_for(proxy):
    opts = Options()
    opts.add_argument("--headless=new")
    opts.add_argument(f"--proxy-server=http://{proxy}")
    return webdriver.Chrome(options=opts)

driver = driver_for(random.choice(proxy_pool))
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)  # check the exit IP
driver.quit()


It works, right up until you need it not to. You now own a maintenance job: testing which IPs are alive, throwing out the dead ones, and guessing whether a working IP has already been burned by someone else. Free proxy lists are short-lived by nature, and the recycled ones bring their reputation with them.

The bigger structural problem is that credentials can't go in this URL. ChromeDriver drops them, so anything beyond open, unauthenticated proxies needs a different mechanism entirely.

### 2. Reassigning `driver.proxy` at runtime with selenium-wire

Selenium Wire sits between Selenium and the browser and gives you proxy control with authentication. It's the approach most current tutorials land on.

bash
pip install selenium-wire
pip install selenium==4.17.2 selenium-wire==5.1.0   # pin deliberately


python
from seleniumwire import webdriver
from selenium.webdriver.chrome.options import Options

opts = Options()
opts.add_argument("--headless=new")

proxy_url = "http://USERNAME:PASSWORD@gw.dataimpulse.com:823"
driver = webdriver.Chrome(
    options=opts,
    seleniumwire_options={
        "proxy": {"http": proxy_url, "https": proxy_url,
                  "no_proxy": "localhost,127.0.0.1"},
        "request_storage": "memory",
    },
)

driver.get("https://api.ipify.org")
print(driver.find_element("tag name", "body").text)
driver.quit()


Because `driver.proxy` is a live attribute, you can swap the upstream without rebuilding the browser:

python
driver.proxy = {"http": new_proxy_url, "https": new_proxy_url,
                "no_proxy": "localhost,127.0.0.1"}


Two things bite people here. First, HTTPS interception needs a certificate installed (`python -m seleniumwire extractcert`), and without it you'll see connection errors that look like proxy failures but aren't. Second, Selenium Wire buffers every request and response by default, which is fine for a five-page test and fatal for a job that runs for six hours. Restrict what it captures and clear the buffer between pages:

python
driver.scopes = [r".*/api/.*"]   # images, fonts, analytics not stored
# ...after each page...
del driver.requests


Pin your versions too. Selenium moves faster than the surrounding libraries, and the widely copied tutorial combination is `selenium==4.17.2` with `selenium-wire==5.1.0` — a version mismatch will produce proxy errors that have nothing to do with your proxy.

### 3. Rotating at the provider gateway

Instead of maintaining a list, you point every browser instance at a single endpoint that hands out a new exit IP per connection. DataImpulse's rotating gateway runs on `gw.dataimpulse.com:823` for HTTP/HTTPS and port `824` for SOCKS5, so "rotation" stops being something your script manages. You don't test IPs, you don't refresh lists, and you don't get different behaviour between your local test and your server.

Where this pays off is failure handling. With a local list, a dead proxy means an exception and a retry loop. With a gateway, a bad exit is a non-event — the next connection gets a different IP.

Authentication is done with credentials (username/password) or by whitelisting your server IP, which removes credential handling from the script entirely. Whichever you pick, the credential string carries your targeting and session settings, so copy it straight from the dashboard's "Proxy access" panel rather than reconstructing it. The gateway and ports are the parts worth hard-coding; the parameter order changes as options get added.

## Handling proxy authentication without selenium-wire

If you'd rather not add a dependency, Selenium 4 can inject credentials through Chrome DevTools Protocol. You enable the `Fetch` domain with `handleAuthRequests`, then answer each `Fetch.authRequired` event with `Fetch.continueWithAuth` and your username and password. It works, it's browser-native, and it requires more code than most projects want to maintain.

The other option is the auth extension trick: a small zipped extension with a `background.js` that calls `chrome.proxy.settings.set` and listens on `chrome.webRequest.onAuthRequired` to return credentials. It's popular for one-off scripts and dated the moment you have more than one proxy configuration.

If you're getting `407` responses, stop debugging the rotation logic. That status code means the credentials never reached the proxy. Check for a stale password, a typo in the username suffix, or the ChromeDriver credential-stripping issue above.

## Rotating or sticky? Decide per target, not per mood

| Job type | Session mode | Why |
| --- | --- | --- |
| SERP collection, catalogue discovery | Rotate per request | Each page is independent; fresh IPs spread the load |
| Product page scraping | Rotate per page, keep headers consistent | No cookie state needed, but fingerprints must not drift |
| Login, cart, checkout, multi-step forms | Sticky | State lives on the server side; rotating mid-flow breaks it |
| Ad verification, price checks by region | Sticky within the region | Location must stay fixed while you compare results |

With a gateway, sticky mode is just a session token appended to your credentials. The same token keeps the same exit IP for the session's life, and the practical ceiling is whether the device behind that IP stays online — sticking a 10-minute session token into a 40-minute job gets you a new address halfway through without warning.

## What DataImpulse charges for rotating proxies

DataImpulse sells first-party residential, datacenter, mobile, and premium residential IPs on a pay-as-you-go model. Residential starts at $1/GB with a $5 minimum, no subscription, and traffic that doesn't expire. The pool is 90M+ IPs across 195 countries, with HTTP/HTTPS and SOCKS5, rotating and sticky sessions, and country targeting included at no extra cost.

Residential plans:

| Plan | Traffic | Price | Effective rate | Notes | Purchase |
| --- | --- | --- | --- | --- | --- |
| Intro | 5 GB | $5 | $1.00/GB | Smallest commitment; enough to measure your real cost per page | [ Get the Intro plan](https://bit.ly/dataimPulse) |
| Basic | 50 GB | $50 | $1.00/GB | Same rate, more headroom for a production crawl | [ Get the Basic plan](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $800 | $0.80/GB | 20% volume discount, dedicated account manager | [ Get the Advanced plan](https://bit.ly/dataimPulse) |
| Custom | 5 TB+ | from $0.70/GB | negotiated | Configured with support, priority handling | [ Ask about a custom plan](https://bit.ly/dataimPulse) |

The other proxy types, if your Selenium job needs a cheaper or a more trusted exit:

| Proxy type | Entry plan | Mid tier | 1 TB tier | Custom | Purchase |
| --- | --- | --- | --- | --- | --- |
| Datacenter | $5 / 10 GB ($0.50/GB) | $50 / 100 GB ($0.50/GB) | $450 / 1 TB ($0.45/GB) | from $2,250 / 5 TB+ | [ See datacenter plans](https://bit.ly/dataimPulse) |
| Mobile (4G/5G/LTE) | $5 / 2.5 GB ($2.00/GB) | $50 / 25 GB ($2.00/GB) | $1,600 / 1 TB ($1.60/GB) | from $8,000 / 5 TB+ | [ See mobile plans](https://bit.ly/dataimPulse) |
| Premium residential | $5 / 1 GB ($5.00/GB) | $50 / 10 GB ($5.00/GB) | — | from $20,000 / 5 TB+ | [ See premium residential plans](https://bit.ly/dataimPulse) |

Two pricing details worth knowing before you commit, because they change your effective cost:

**Country targeting is free; city, ZIP, and ASN targeting is billed at 2× the standard rate on residential plans.** If your Selenium script needs a specific city to match local pricing or search results, your $1/GB residential traffic is effectively $2/GB. Country-level targeting costs nothing extra, so use it unless the job genuinely needs finer granularity. Datacenter plans reportedly include the advanced filters without that surcharge — worth confirming with support if granular targeting is central to your setup.

**There's no free trial.** The entry point is $5 across all four proxy types, which buys 5 GB of residential, 10 GB of datacenter, or 2.5 GB of mobile traffic. Intro plans carry a 7-day money-back guarantee on card payments as long as you've used less than 80% of the traffic; crypto purchases aren't refundable. That's a workable way to test, but budget for it rather than expecting free credits.

## Choosing a plan from traffic math, not from the label

The plan name tells you nothing. What matters is how much proxied bandwidth one successful page costs you, and that's a number only your own scraper can produce.

Rough starting point: a lightweight HTML page might transfer a few hundred KB through the proxy, while a JavaScript-rendered product or search page with images, fonts, and tracking scripts can easily move 1.5–3 MB. At $1/GB, 5 GB is roughly 1,700–3,300 fully rendered page loads. That's plenty to run the measurement:

1. Buy the Intro plan — [👉 start with 5 GB for $5](https://bit.ly/dataimPulse).
2. Run 500 pages with your real settings, including rendering and screenshots.
3. Check the dashboard for traffic consumed, and divide by pages that returned usable data — not pages that returned HTTP 200.
4. Multiply by your monthly page target.

The failure mode to avoid is buying 1 TB because the per-GB rate looks better. The Advanced tier's $0.80/GB only matters once you're actually moving a terabyte. Below 50 GB a month, the flat $1/GB with a $5 minimum is the part that saves money, because you're not paying for a bundle you won't finish — and unused GBs don't expire.

If your targets are aggressive enough that residential success rates drop, mobile exits are the escalation path, but they cost twice as much per GB and you'll burn through a test budget quickly. Try the cheaper exit first and measure whether the block rate actually justifies the upgrade.

## Seven habits that quietly burn your proxy traffic

1. **Loading what you don't parse.** Block images and fonts in Chrome options, or scope Selenium Wire's capture, and your per-page cost drops.
2. **Retrying against the same identity.** A 403 means the exit is suspect. Retrying it three times costs traffic and buys nothing. Rotate, then retry once.
3. **Taking screenshots on every page.** They're useful for debugging, expensive at scale, and they don't go through the proxy — but the render behind them does.
4. **No page-load timeout.** A hanging page on a slow exit keeps the browser alive and the meter running.
5. **Uniform retry logic.** A timeout deserves a retry with the same session; a CAPTCHA deserves a new identity and a longer pause. Treating them the same turns a recoverable block into a pattern.
6. **Concurrency without backoff.** Ten parallel browsers through one gateway is fine; ten parallel browsers hammering one endpoint with no pause is a signature.
7. **Reusing a session across account boundaries.** Cookies from one job leaking into the next is how one burned identity becomes three.

## Common questions from people setting this up for the first time

**Does Selenium support SOCKS5 proxies?** Yes, via `--proxy-server=socks5://host:port`. DataImpulse exposes SOCKS5 on port 824 and HTTP/HTTPS on 823, so you can match whichever your stack prefers.

**Why do I keep getting a username and password prompt?** ChromeDriver ignores credentials in the `--proxy-server` flag, so the browser has nothing to authenticate with and asks you. Move to Selenium Wire, CDP auth, or an IP whitelist.

**Do I have to rotate on every single request?** No, and doing so is often counterproductive. Rotate per page on independent targets, and keep a session for anything stateful.

**Can I use the same proxy credentials for `requests` and Selenium?** Yes. The credential string works in both, so you can fetch static pages with `requests` at low cost and reserve the browser for pages that genuinely need JavaScript.

**Is a 90M-IP pool big enough?** For most scraping workloads, yes — the same IPs aren't resold across multiple brands, which reduces the odds of arriving at a target with an already-burned reputation. If you're crawling at a scale where IP overlap itself drives block rates, the enterprise providers advertising 175M+ to 400M+ IPs still have an edge, at several times the per-GB price.

## The minimum viable rotating proxy stack

Selenium Wire for authenticated proxy control, a gateway endpoint instead of a local IP list, sticky sessions only where state demands it, and scoped capture so your headless browsers don't drown in buffered requests. Add failure-type-aware retry logic, and you've handled nearly every rotation problem that shows up in production.

Start by measuring. Five dollars of traffic tells you more about your real cost per successful page than any pricing table can — including this one. [👉 Grab 5 GB of residential traffic and check the exit IP for yourself](https://bit.ly/dataimPulse).
