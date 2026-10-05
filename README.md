# puppeteer proxy authentication: page.authenticate, 407 fixes, and rotation that survives real targets

You added the proxy flag, the credentials are definitely correct in the dashboard, and the script still dies with `407 Proxy Authentication Required` — or worse, it hangs and returns a blank page. Nine times out of ten this isn't a proxy problem at all. Chromium quietly ignores credentials embedded in `--proxy-server`, and Puppeteer only picks up proxy auth through a separate call that has to happen *before* the first navigation. Get that ordering wrong and everything downstream looks broken.

This guide covers how proxy authentication actually works inside Puppeteer, where it breaks, how to rotate sessions without forcing a new browser every time, and how to keep the bandwidth bill from quietly tripling while you debug.

## Why the flag alone never authenticates

The `--proxy-server` switch is a Chromium command-line option, and it accepts one thing: `host:port`. Drop `user:pass@` in front of it and nothing errors out — the string is simply not parsed the way you'd expect, so the request leaves unauthenticated and the proxy answers with 407.

Authentication travels through a different channel. Puppeteer exposes it as `page.authenticate()`, which registers credentials for that page's network layer. Two properties matter:

- It's **per page**, not per browser. Every page you open needs its own call.
- It must run **before** the first navigation. `page.authenticate()` after `page.goto()` means the first request already went out bare.


npm install puppeteer


That's the whole dependency list if you stick with the built-in approach. Node 18 or newer is a reasonable baseline.

## The minimum working setup

Here's the shape that works, using a rotating residential gateway as the endpoint:

js
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch({
    headless: true,
    args: ['--proxy-server=gw.dataimpulse.com:823'], // host:port only — no credentials
  });

  const page = await browser.newPage();

  // Must run BEFORE the first navigation.
  await page.authenticate({
    username: 'YOUR_LOGIN__cr.us',   // targeting suffix lives on the username
    password: 'YOUR_PASSWORD',
  });

  await page.goto('https://api.ipify.org?format=json', { waitUntil: 'domcontentloaded' });
  console.log(await page.evaluate(() => document.body.innerText));

  await browser.close();
})();


Two details in there carry most of the weight. First, `--proxy-server=gw.dataimpulse.com:823` — port 823 is the HTTP/HTTPS endpoint. Second, the `__cr.us` suffix: that's country targeting, appended to the login, which is how most rotating residential networks work. Country-level targeting is included in the base rate at DataImpulse; finer filters such as city, ZIP and ASN are billed as an add-on.

If you are still choosing an endpoint to test against, 👉 [👉 Start with a DataImpulse residential endpoint and run your first authenticated request](https://bit.ly/dataimPulse) — the entry pack is 5 GB for $5, and unused traffic doesn't expire, so a debugging session costs cents rather than a monthly minimum.

## Confirm the tunnel before you blame your code

This step saves the most time of anything in this article. Test the exact same credentials outside Puppeteer first:

bash
curl -x "http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823" https://api.ipify.org


Three outcomes, three different problems:

- **An IP comes back** → credentials and endpoint are fine. Any failure inside Puppeteer is a code issue.
- **407** → the proxy rejected the credentials. Re-copy them from the dashboard, and watch for a broken targeting suffix, stray whitespace, or a plan that hasn't been activated with a top-up.
- **Connection refused or timeout** → wrong host or port, or a protocol mismatch (HTTP endpoint addressed as SOCKS5, or the reverse).

Never hardcode credentials in source files. Environment variables keep them out of git:

js
const PROXY_URL = `http://${process.env.PROXY_USER}:${process.env.PROXY_PASS}@gw.dataimpulse.com:823`;


That string form is perfect for `curl` and for HTTP clients, and it's exactly the form Puppeteer's flag will ignore — which is why you'll see both patterns in the same codebase.

## Three places a proxy can attach

Puppeteer lets you scope a proxy at three levels, and picking the right one is mostly about what you need to vary.

**1. Whole browser.** One proxy for everything, set via `args`. Simplest, fastest to write, and completely fixed for the browser's lifetime — `--proxy-server` cannot be changed after launch. Every page shares one exit IP unless the upstream gateway rotates by itself.

**2. Per browser context.** One browser process, several isolated cookie jars, each with its own exit IP. This is the pattern for running parallel accounts without letting them share a fingerprint:

js
const context = await browser.createBrowserContext({
  proxyServer: 'gw.dataimpulse.com:823',
});
const page = await context.newPage();
await page.authenticate({
  username: 'YOUR_LOGIN__cr.us;sid.acct07;sessttl.600',
  password: 'YOUR_PASSWORD',
});


Note the method name: it was `createIncognitoBrowserContext()` until Puppeteer 22, then renamed to `createBrowserContext()`. Copying an older snippet is a common source of "is not a function" errors. If you're on a version older than 22, use the old name.

**3. One browser per job.** The heaviest in memory, the cleanest in isolation. Launch, authenticate, do the work, close. The session ID you send decides whether the next job lands on the same IP or a fresh one.

For anything above a handful of parallel workers, option 2 is usually the sweet spot. A separate Chrome process per job burns a few hundred MB each and gains you very little once contexts are properly isolated.

## Targeting and session control ride on the username

Rotating residential providers generally don't give you a dashboard toggle per location. They give you a credential string where the username carries the instructions:


LOGIN__cr.us;sid.job01;sessttl.600


- `__cr.us` — target country
- `;sid.job01` — session ID; the same ID keeps the same exit IP
- `;sessttl.600` — session lifetime in seconds
- `;ci.newyork` — city-level targeting where the plan allows it (billed as an add-on on residential plans)

Once you internalize this, a class of confusing errors makes sense. A typo in the suffix doesn't just lose you the geo — it can fail the whole authentication handshake, which is why "the password is definitely right" and "407" can both be true at the same time. Keep the suffix in one constant and construct the username from it instead of retyping it in six files.

## Rotation versus sticky sessions in Puppeteer

Rotation isn't a Puppeteer feature. It's a property of the session ID you send. No `sid` means a new IP per connection; a `sid` means the same IP until the TTL runs out. That single distinction determines your whole job design:

| Job | Session config | Why |
| --- | --- | --- |
| Crawling many pages of one site | No `sid`, rotating | Spreads load across the pool |
| Logged-in workflow | `sid` + long `sessttl` | The site sees one stable location |
| Parallel workers on the same target | One `sid` per worker | Workers never share an IP |
| Checkout or multi-step form | `sid`, longer TTL | An IP change mid-flow looks like session hijacking |

The rule worth writing on a sticky note: never reuse one session ID across two accounts. Two accounts on the same IP link themselves together far more convincingly than any browser fingerprint.

Rotation with `--proxy-server` has one trap. Because the flag is fixed at launch, explicit rotation means launching a new browser per proxy — unless you point Puppeteer at a local wrapper that rotates upstream for you.

## When you want `proxy-chain` instead

The built-in approach has one real limitation: the credentials must be applied per page, and there are flows where that's awkward or impossible to wire cleanly. `proxy-chain` starts a local proxy that holds your credentials and forwards to the upstream gateway, so Chromium only ever sees `127.0.0.1`:

js
const proxyChain = require('proxy-chain');

const upstream = 'http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823';
const localProxy = await proxyChain.anonymizeProxy(upstream);

const browser = await puppeteer.launch({
  args: [`--proxy-server=${localProxy}`], // no credentials needed here
});

// ... scraping ...

await proxyChain.closeAnonymizedProxy(localProxy, true);


Because the flag is now pointed at your own wrapper, you can swap the upstream URL between runs and get rotation without shipping credentials into the browser process.

Version note: `proxy-chain` v3 dropped CommonJS support, so `require('proxy-chain')` throws `ERR_REQUIRE_ESM`. Either install v2 explicitly (`npm install proxy-chain@2`) or convert the file to ESM. Half the "proxy-chain is broken" reports online are this.

## The SOCKS5 authentication gotcha

Chromium does not support username/password authentication over SOCKS5. Not partially — not at all. So:

- HTTP/HTTPS endpoints (port 823 on DataImpulse) handle credentialed auth.
- SOCKS5 (a separate port, shown in your dashboard) is for IP-whitelisted access, where no credentials are sent.

If you configured SOCKS5 with a username and password and got strange failures rather than a clean error, that's the reason. SOCKS5 also uses a different port from the HTTP gateway, and published numbers vary between provider docs and tutorials — always confirm the port in your own dashboard rather than trusting a blog snippet, including this one.

## Decoding the errors you'll actually see

| Symptom | Most likely cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials never sent, or sent after the first navigation | Move `page.authenticate()` above `page.goto()`; stop putting `user:pass@` in the flag |
| `ERR_NO_SUPPORTED_PROXIES` | Malformed proxy string, protocol mismatch, or credentials embedded in the flag | Confirm the format with `curl` first, then mirror it exactly |
| `ERR_INVALID_AUTH_CREDENTIALS` | Proxy received the request and rejected the login | Re-copy credentials; check the targeting suffix and that the plan is active |
| `ERR_TUNNEL_CONNECTION_FAILED` | Host/port unreachable, or the endpoint isn't the one you think | Re-check host and port; verify you're on HTTP and not SOCKS5 |
| Blank page, no error | Navigation resolved before content arrived, or the request never left | Add `waitUntil`, and log the exit IP to confirm the route |
| `ERR_REQUIRE_ESM` on `proxy-chain` | v3 installed, CommonJS `require` | `npm install proxy-chain@2` or switch to ESM |

A cheap sanity check inside the browser itself: after authenticating, fetch `https://api.ipify.org?format=json` and print the result. If the returned IP resolves to the country you targeted, auth, routing and geolocation are all confirmed in one shot. If it returns your own ISP's IP, the proxy was never used — usually a flag typo, not a credential problem.

## Which proxy to budget for a Puppeteer workload

Headless Chrome is bandwidth-hungry compared to a plain HTTP client. It fetches images, fonts and JS bundles you may not care about. A request that costs a few hundred kilobytes through `requests` can cost a couple of megabytes through Puppeteer, so the per-GB rate matters more here than in almost any other scraping setup.

That pushes toward two decisions. Block what you don't need — an inline request interceptor that aborts image and font requests can cut transfer volume substantially before any proxy is involved. And pick an IP type that matches the target: datacenter IPs are cheaper and faster but get challenged on protected sites, residential IPs pass as ordinary users, and mobile IPs are the escape hatch for the hardest targets at the highest cost per GB.

DataImpulse prices all three on a pay-as-you-go basis with traffic that doesn't expire, which fits the erratic bandwidth profile of browser automation better than a monthly quota:

| Proxy type | Entry package | Per-GB rate | Volume tier | Billing | Where it fits in Puppeteer |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 / 5 GB | $1.00/GB | $800 / 1 TB ($0.80/GB); 5 TB tier at $0.70/GB | Pay-as-you-go, GBs never expire | Default for protected sites, SERPs, e-commerce |
| Datacenter | $5 / 10 GB | $0.50/GB | $450 / 1 TB ($0.45/GB); custom from $2,250 at 5 TB+ | Pay-as-you-go, GBs never expire | Fast, cheap runs against targets without aggressive blocking |
| Mobile | $5 / 2.5 GB | $2.00/GB | $1,600 / 1 TB ($1.60/GB); custom from $8,000 at 5 TB+ | Pay-as-you-go, GBs never expire | Mobile-web and app data, hardest targets |
| Premium Residential | $5 / 1 GB | $5.00/GB | Custom from $20,000 at 5 TB+ | Pay-as-you-go, GBs never expire | High-trust residential traffic, dedicated account manager |

All four are HTTP/HTTPS and SOCKS5, with country targeting included and city/ZIP/ASN available as a paid add-on. DataImpulse publishes a pool of 90M+ IPs across 195 countries, a 99.51% success rate, a 4.8/5 G2 rating, and 24/7 human support — treat those as the vendor's own figures rather than independently audited numbers, the same way you should treat any provider's marketing page.

The cost math is straightforward once you know your volume. At $1/GB, 5 GB of residential traffic costs $5 whether you burn it in an afternoon or across three months. If your Puppeteer job transfers 2 MB per page, that entry pack covers roughly a couple of thousand pages — which is usually enough to find out whether the proxy survives your targets before you commit to a volume tier.

👉 [👉 Compare DataImpulse's residential, datacenter and mobile rates on the pricing page](https://bit.ly/dataimPulse) — the per-GB numbers are public and there's no subscription to cancel afterwards.

One caveat worth stating plainly: if you need static ISP proxies, a fully managed scraping API, or access to banking and government sites, this isn't the right tool. Rotating residential, mobile and datacenter IPs for public data collection is the use case.

## A pre-flight checklist for authenticated Puppeteer runs

Before you blame the proxy, confirm these in order:

1. `curl -x` with the same credentials returns an IP, not a 407.
2. The URL is HTTP/HTTPS, not SOCKS5, when credentials are required.
3. `--proxy-server` contains `host:port` and nothing else.
4. `page.authenticate()` runs before the first `page.goto()`, and on **every** page you create.
5. The targeting suffix in the username is spelled correctly, including separators.
6. `headless: true` vs headful makes no difference to auth — if flipping it changes the result, you're looking at a fingerprint problem, not a credential one.
7. Credentials come from environment variables, not source files.
8. You log the exit IP after authenticating, so you know the route is live before the real requests start.

## FAQ

**Does `page.authenticate()` work in new headless mode?**
Yes. Proxy authentication is handled at the network layer, not the rendering layer, so headless mode doesn't change it. If switching headless modes "fixes" your auth error, the original failure was almost certainly something else — most often a missing `await`.

**Why does my first request fail but the rest succeed?**
Classic ordering bug. The first navigation escaped before `page.authenticate()` ran. Move the call above `page.goto()` and the asymmetry disappears.

**Can I reuse one session ID across many parallel workers?**
Technically yes, practically no. Workers sharing a session ID share the exit IP, which is fine for load-spreading across independent pages and disastrous for anything involving logged-in accounts.

**Is the `--proxy-server` flag enough for IP-whitelisted proxies?**
Yes. If the proxy authenticates by source IP, no credentials are sent and the flag alone is sufficient — that's the standard setup for SOCKS5 access.

**How much does debugging an authenticated Puppeteer setup cost?**
With pay-as-you-go pricing at $1/GB and a $5 entry pack, effectively nothing. The expense of headless browser automation shows up at scale, not during setup — which is a good argument for measuring real traffic volume before committing to any volume tier.
