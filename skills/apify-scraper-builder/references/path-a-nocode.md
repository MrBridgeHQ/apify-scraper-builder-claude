# Path A - Configure a No-Code Scraper Actor

Use Path A when the scrape is a **one-off or prototype** - you want data once, you don't need to publish or monetize an Actor of your own. No `apify create`, no Docker, no git, no TypeScript. You configure an existing Apify Actor via its input form in the Console.

This file assumes you've read the main `SKILL.md`. It expands Path A specifics.

## Pick the right no-code Actor

| Actor | Best for | Speed | Anti-bot tolerance |
|---|---|---|---|
| [`apify/cheerio-scraper`](https://apify.com/apify/cheerio-scraper) | Static HTML. Default choice. | Fastest | Low |
| [`apify/web-scraper`](https://apify.com/apify/web-scraper) | JS-rendered pages; runs your `pageFunction` inside a real browser with jQuery loaded | Slow | Moderate |
| [`apify/puppeteer-scraper`](https://apify.com/apify/puppeteer-scraper) | When you specifically need Puppeteer's API | Slow | Moderate |
| [`apify/playwright-scraper`](https://apify.com/apify/playwright-scraper) | Modern browser scraper; harder to block than Puppeteer | Slow | Moderate-high |

Default to **cheerio-scraper** until you observe that selectors return nothing (then escalate to web-scraper), following the standard scraping tool ladder.

## Workflow (5 minutes)

1. Open the Actor page (e.g. `apify/cheerio-scraper`). Click "Try for free" - opens the Console with an input form.
2. **Start URLs:** add one or more URLs. The Actor will scrape these first.
3. **Link selector:** CSS selector for links the Actor should follow (for pagination). Often `a.next-page` or `[rel="next"]`.
4. **Pseudo URLs:** patterns that the Actor will enqueue from the link selector. Use `[.+]` placeholders. Example: `https://example-shop.com/category/[.+]?page=[.+]`.
5. **Page function:** the JavaScript that extracts data. See templates below.
6. **Proxy configuration:** turn on Apify Proxy with DATACENTER group. Escalate to RESIDENTIAL if blocks appear.
7. **Save & run.** Inspect the dataset in the Console.

## Page function template - cheerio-scraper

`apify/cheerio-scraper` exposes `context` with `$` (Cheerio), `request`, `body`, `log`. No browser, no JS execution.

```javascript
async function pageFunction(context) {
    const { $, request, log } = context;

    const products = $('[data-testid="product-card"]').map((_, el) => {
        const $card = $(el);
        return {
            name: $card.find('h2, .product-name').first().text().trim() || null,
            price: $card.find('[itemprop="price"], .price').first().text().trim() || null,
            url: new URL($card.find('a').attr('href') ?? '', request.url).href,
            imageUrl: $card.find('img').attr('src') ?? null,
            availability: $card.find('[itemprop="availability"]').attr('content') ?? null,
        };
    }).get();

    log.info(`Extracted ${products.length} products from ${request.url}`);
    return products;  // returning array auto-pushes each to dataset
}
```

Return value rules:
- Return an **array** → each item is pushed to the dataset
- Return an **object** → pushed as one item
- Return `undefined` or `null` → nothing pushed (use for navigation-only pages)
- **Throw** → the request is retried per the Actor's retry config

## Page function template - web-scraper

`apify/web-scraper` runs your function inside a real Chromium page. You get `context.jQuery` (full jQuery), `context.request`, `context.page` (the Puppeteer page), and DOM access.

```javascript
async function pageFunction(context) {
    const { $, request, log, page } = context;

    // $ here is jQuery - extracts from the rendered DOM (after JS executed)
    await page.waitForSelector('[data-testid="product-card"]', { timeout: 10000 });

    const products = $('[data-testid="product-card"]').map((_, el) => {
        const $card = $(el);
        return {
            name: $card.find('h2').text().trim(),
            price: $card.find('.price').text().trim(),
            url: new URL($card.find('a').attr('href') ?? '', request.url).href,
        };
    }).get();

    log.info(`Extracted ${products.length} products`);
    return products;
}
```

The browser context means the JS-rendered content is available, but each request is 10–50× more expensive in compute and memory than cheerio-scraper. Only use when cheerio-scraper observably fails.

## Pagination patterns

### Pattern 1: Link selector + pseudo URLs (simplest)

- **Link selector:** `a.next-page, [rel="next"]`
- **Pseudo URLs:** `https://example-shop.com/category/laptops?page=[.+]`

The Actor follows the link, matches the URL against the pattern, enqueues if it matches.

### Pattern 2: Programmatic enqueue inside pageFunction

```javascript
async function pageFunction(context) {
    const { $, request, log, enqueueRequest } = context;

    // extract products...

    const nextHref = $('a.next-page').attr('href');
    if (nextHref) {
        await enqueueRequest({
            url: new URL(nextHref, request.url).href,
            userData: { label: 'CATEGORY' },
        });
    }

    return products;
}
```

More control, but harder to maintain than declarative link-selector + pseudo URLs.

### Pattern 3: Pre-known pagination URLs in Start URLs

If pagination is `?page=1`, `?page=2`, ..., just list all in Start URLs upfront:

```
https://example-shop.com/category/laptops?page=1
https://example-shop.com/category/laptops?page=2
https://example-shop.com/category/laptops?page=3
...
```

Simplest possible - works when you know the total page count.

## Anti-bot inside Path A

You get less control than in Path C. Available knobs:

| Knob | How |
|---|---|
| **Proxy group** | Input form → Proxy configuration → choose group |
| **Sessions** | Built-in; `apify/web-scraper` rotates sessions automatically |
| **Concurrency** | Input form → Max concurrency |
| **Rate limit** | Input form → Max requests per minute (when available) |
| **Custom headers** | Inside pageFunction with `context.request.headers` modification (limited support) |

For deeper anti-bot tuning, Path C with `ts-crawlee-playwright-camoufox` gives you more control than any no-code scraper can.

## When to graduate from Path A to Path C

Signals that Path A is no longer the right fit:

| Signal | Why graduate |
|---|---|
| You're maintaining the same scraper for the same site for >1 month | The `pageFunction` string in the Console is hard to version-control. Path C lives in git. |
| You want to publish this scraper on the Store | You can't publish someone else's Actor. Path C makes it yours. |
| You want PPE revenue | Path A's revenue goes to the Actor owner (Apify or whoever published it). Path C is yours. |
| Selectors are getting complex (multi-step extraction, conditional logic) | The `pageFunction` becomes a tangle. Path C's `extractors/*.ts` modules + unit tests are cleaner. |
| You need to call external APIs (Claude, OpenAI) inside the extraction | Doable in Path A but with awkward token management. Path C handles env vars properly. |
| You need fixtures-based tests | Path A has no test layer. Path C does. |
| Run cost is >$0.10/run and you can't optimize | Path A doesn't let you tune memory and image - Path C does. |

When you graduate, the `pageFunction` JS translates almost 1:1 to a Crawlee request handler. The extraction logic stays the same; you get a real project around it.

## Storing scraper results outside Apify (Path A specific)

A Path A run pushes to the Apify dataset. To get the data out:

- **Console:** Download as JSON/CSV/Excel from the run's dataset tab.
- **API:** `GET /v2/datasets/{datasetId}/items?format=json` with your API token.
- **Integrations:** Apify has native integrations (Zapier, Make, Google Drive) that auto-push dataset on run finish. Configure via the Actor's "Integrations" tab.

For one-off scrapes, the Console download is usually enough.

## Cost considerations

Path A is billed by the **Actor owner's pricing model**:
- `apify/cheerio-scraper` uses Pay-per-Result at ~$0.30 per 1k results (check current price on the Actor page).
- `apify/web-scraper` is more expensive due to browser overhead.

If you scrape >10k items per month, Path C usually wins on per-item cost because you control memory and image. But the saved engineering time of Path A often outweighs the per-item premium for small scrapes.

## What Path A cannot do

- Custom Dockerfiles (e.g. install a non-npm dependency, set system fonts)
- Persistent state across runs (Path A runs are independent; no KV Store usage by default)
- Multi-Actor orchestration (`Actor.metamorph`, sub-Actors, AI Agent patterns)
- Standby mode (the no-code Actors aren't Standby - Standby is an MCP-server / always-on Actor concern)
- Selling your scraper on the Apify Store

For any of these, you need Path C.
