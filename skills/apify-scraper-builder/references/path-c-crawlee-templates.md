# Path C - Crawlee Templates (Production Path)

The canonical production path for scraper Actors on Apify. Full walkthrough from `apify create` to a deployed, monetized, observable scraper.

This file assumes you've read the main `SKILL.md`. It expands Path C specifics. The cross-cutting doctrine layer (anti-bot, tool ladder, selectors, error handling, fixtures) is a separate concern this file references but does not duplicate.

## Template sub-decision

```bash
apify create my-scraper                            # interactive picker
apify create my-scraper --template ts-crawlee-cheerio              # static HTML
apify create my-scraper --template ts-crawlee-puppeteer-chrome     # legacy browser
apify create my-scraper --template ts-crawlee-playwright-chrome    # modern browser
apify create my-scraper --template ts-crawlee-playwright-camoufox  # anti-bot
apify create my-scraper --template ts-empty                        # bring-your-own
apify create my-scraper --template ts-start                        # single-page
```

Equivalent Python: replace `ts-` with `python-`, `cheerio` with `beautifulsoup` or `parsel`. Python templates are valid but less commonly used on Apify Store; Node is the default for new scrapers.

| Template | Target characteristics | Memory tier | Typical run cost |
|---|---|---|---|
| `ts-crawlee-cheerio` | Static HTML, no JS challenge, low rate limiting | 256–512 MB | $0.01–0.05 / 1k items |
| `ts-crawlee-puppeteer-chrome` | JS-rendered, basic anti-bot - legacy choice | 1–2 GB | $0.50–1.00 / 1k items |
| `ts-crawlee-playwright-chrome` | JS-rendered, Cloudflare basic, DataDome | 2–4 GB | $0.50–2.00 / 1k items |
| `ts-crawlee-playwright-camoufox` | Cloudflare Enterprise, HUMAN/PerimeterX, Akamai | 2–4 GB | $1.00–3.00 / 1k items |
| `ts-empty` / `ts-start` | Custom architecture | varies | varies |

The full tool-ladder reasoning (cost-per-result, IP economics, escalation triggers) belongs to your scraping anti-bot doctrine. Consult it before picking when uncertain. **Default to Cheerio and escalate on observed failure** - using Playwright for static HTML burns 10× compute for zero benefit.

## Project structure

```
my-scraper/
├── .actor/
│   ├── actor.json
│   ├── input_schema.json
│   ├── dataset_schema.json
│   ├── output_schema.json        # optional; specifies where output lives
│   └── pay_per_event.json        # ADD this - not in default template
├── src/
│   ├── main.ts                   # Actor.init, input read, crawler bootstrap, Actor.exit
│   ├── routes.ts                 # Crawlee router (per-label handlers)
│   ├── extractors/
│   │   └── productCard.ts        # pure function: $card → Product
│   ├── filters/
│   │   └── urlBuilders.ts        # input → pagination URLs (pure functions)
│   ├── ppe/
│   │   └── charge.ts             # chargeEvent wrapper with cap handling
│   ├── utils/
│   │   ├── cache.ts              # KV Store cache wrapper
│   │   ├── errors.ts             # error envelope helpers
│   │   └── status.ts             # Actor.setStatusMessage wrappers
│   └── types.ts                  # Input, Product, EventName
├── tests/
│   ├── fixtures/                 # captured HTML (scrubbed of PII)
│   │   ├── category-page.html
│   │   └── product-detail.html
│   ├── extractors.test.ts
│   └── routes.test.ts
├── storage/                      # local emulation (gitignored)
│   ├── datasets/
│   ├── key_value_stores/
│   │   └── default/
│   │       └── INPUT.json        # local input
│   └── request_queues/
├── Dockerfile
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── .dockerignore
├── .gitignore
└── README.md
```

The `storage/` folder is the local emulator. Files under `storage/datasets/default/` are the JSON output items; `storage/key_value_stores/default/INPUT.json` is the local input. Add `storage/` to `.gitignore`. The Apify Cloud uses the same paths internally so behavior matches.

## `.actor/actor.json` - production template

```json
{
  "actorSpecification": 1,
  "name": "my-scraper",
  "title": "My Scraper - Concise Tagline",
  "description": "One sentence with primary keywords.",
  "version": "0.1",
  "buildTag": "latest",
  "input": "./input_schema.json",
  "dockerfile": "./Dockerfile",
  "storages": {
    "dataset": "./dataset_schema.json"
  },
  "categories": ["E_COMMERCE", "AUTOMATION"],
  "minMemoryMbytes": 256,
  "maxMemoryMbytes": 2048,
  "defaultRunOptions": {
    "build": "latest",
    "memoryMbytes": 512,
    "timeoutSecs": 3600
  }
}
```

Memory tiers and cost, as a rule of thumb:
- CheerioCrawler: 256–512 MB default, up to 1 GB at high concurrency
- Light Playwright: 1024–2048 MB
- Heavy Playwright / Camoufox: 4096 MB
- AI-extraction Actors with multiple LLM calls: 4096 MB

## `.actor/input_schema.json` - for a category scraper

```json
{
  "title": "Category Scraper Input",
  "type": "object",
  "schemaVersion": 1,
  "properties": {
    "startUrls": {
      "title": "Start URLs",
      "type": "array",
      "editor": "requestListSources",
      "description": "Category page URLs to scrape. One per line.",
      "prefill": [{ "url": "https://example-shop.com/category/laptops" }]
    },
    "maxProducts": {
      "title": "Max products",
      "type": "integer",
      "default": 100,
      "minimum": 1,
      "maximum": 10000,
      "description": "Stop after this many products are scraped (across all start URLs)."
    },
    "proxyConfiguration": {
      "title": "Proxy configuration",
      "type": "object",
      "editor": "proxy",
      "description": "Datacenter is sufficient for most sites. Residential for anti-bot.",
      "default": { "useApifyProxy": true, "apifyProxyGroups": ["DATACENTER"] },
      "sectionCaption": "Advanced settings"
    },
    "maxRequestsPerMinute": {
      "title": "Max requests per minute",
      "type": "integer",
      "default": 60,
      "minimum": 1,
      "maximum": 600,
      "description": "Target rate. Tune down if the site rate-limits.",
      "sectionCaption": "Advanced settings"
    }
  },
  "required": ["startUrls"]
}
```

For the full editor catalog (`stringList`, `requestListSources`, `proxy`, `select`, etc.), see the Apify input-schema documentation.

## `.actor/dataset_schema.json` - product item shape

Declared via `actor.json` → `storages.dataset`. Validates output and powers the Console's "Dataset preview" UI.

```json
{
  "actorSpecification": 1,
  "fields": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "nullable": true },
      "price": { "type": "number", "nullable": true },
      "priceRaw": { "type": "string", "nullable": true },
      "currency": { "type": "string", "nullable": true },
      "url": { "type": "string", "format": "uri" },
      "imageUrl": { "type": "string", "format": "uri", "nullable": true },
      "availability": {
        "type": "string",
        "enum": ["InStock", "OutOfStock", "PreOrder", null],
        "nullable": true
      }
    },
    "required": ["url"]
  },
  "views": {
    "overview": {
      "title": "Overview",
      "transformation": { "fields": ["name", "price", "currency", "availability", "url"] },
      "display": { "component": "table" }
    }
  }
}
```

Notes:
- `nullable: true` for fields that may be absent (the extractor returns nulls on missing selectors - see §"`src/extractors/productCard.ts`").
- `availability` enum: we normalize to the three most common schema.org `Offer.availability` values (`InStock`, `OutOfStock`, `PreOrder`). Less common values (`Discontinued`, `LimitedAvailability`, `BackOrder`, etc.) map to `null` - document this in the README under "Output schema".
- `views.overview` controls what the Console shows on the Dataset tab. Pick the most informative 4–5 fields.

Full schema validation doctrine (including JSON-Schema constraints, error messages, transformations) belongs to your scraping selector-strategy reference.

## `.actor/pay_per_event.json` - production template

```json
{
  "item-scraped": {
    "eventTitle": "Product item scraped",
    "eventDescription": "Per successful product item committed to the default dataset.",
    "eventPriceUsd": 0.001
  }
}
```

For multi-event scrapers (e.g. category-page scan + product-detail-fetch), declare each event separately and charge per logical operation. Defer price points to your Apify pricing-strategy doctrine - **do not guess prices**.

**Critical:** if you ALSO declare the synthetic `apify-default-dataset-item` event, every `Actor.pushData()` charges BOTH events. This is almost never what you want - declare ONE of the two, not both.

## `.actor/Dockerfile` - Cheerio template

The `apify create` template ships a working Dockerfile; for production you usually only adjust the base image and add `--omit=dev`. Minimal canonical form:

```Dockerfile
FROM apify/actor-node:20

COPY package*.json ./
RUN npm install --omit=dev --omit=optional --prefer-offline --no-fund --no-audit \
    && echo "Installed npm packages:" \
    && (npm list || true) \
    && echo "Node.js version:" \
    && node --version

COPY . ./

CMD npm start --silent
```

For **Playwright** templates (`ts-crawlee-playwright-chrome`), swap `FROM apify/actor-node:20` → `FROM apify/actor-node-playwright-chrome:20`. For **Camoufox**, use `apify/actor-node-playwright-firefox:20` then add `RUN npm install camoufox-js`. The full base-image catalog is in the Apify Docker base-image documentation.

## `src/runtime.ts` - shared mutable state (typed)

Routes need to read/write state across many request invocations. The canonical pattern is a **module-scope runtime object** imported by both `main.ts` (writes config at startup) and `routes.ts` (reads config, mutates state per request). This is preferred over stashing on `crawler.userData` because it stays fully typed and avoids `as any` casts.

```typescript
// src/runtime.ts
export interface State {
  collectedCount: number;
  chargeLimitReached: boolean;
}

export interface Config {
  maxProducts: number;
}

export const runtime = {
  state: { collectedCount: 0, chargeLimitReached: false } as State,
  config: { maxProducts: 100 } as Config,
};
```

Modules are singletons per Node process, so `runtime` is shared across all route invocations within one Actor run. Apify Standby Actors that handle concurrent users would need per-session state (KV Store-keyed by user) - but for batch-mode scrapers this is the right shape.

## `src/main.ts` - production skeleton

```typescript
import { CheerioCrawler } from '@crawlee/cheerio';
import { Actor, log } from 'apify';
import { router } from './routes.js';
import { runtime, type State } from './runtime.js';

interface Input {
  startUrls: Array<{ url: string; userData?: Record<string, unknown> }>;
  maxProducts?: number;
  maxRequestsPerMinute?: number;
}

await Actor.init();

const input = (await Actor.getInput<Input>()) ?? {} as Input;
const { startUrls = [], maxProducts = 100, maxRequestsPerMinute = 60 } = input;

if (startUrls.length === 0) {
  await Actor.pushData({
    error: true,
    errorType: 'INVALID_INPUT',
    message: 'No startUrls provided. Provide at least one category URL.',
    timestamp: new Date().toISOString(),
  });
  await Actor.setStatusMessage('Error: no startUrls. See dataset.', { isStatusMessageTerminal: true });
  await Actor.exit();
}

// Restore state across migrations
const restored = await Actor.getValue<State>('STATE');
if (restored) runtime.state = restored;
runtime.config.maxProducts = maxProducts;

Actor.on('migrating', async () => Actor.setValue('STATE', runtime.state));
Actor.on('aborting', async () => Actor.setValue('STATE', runtime.state));

const proxyConfiguration = await Actor.createProxyConfiguration({
  groups: ['DATACENTER'],  // escalate to RESIDENTIAL on observed blocks
  checkAccess: true,
});

const crawler = new CheerioCrawler({
  proxyConfiguration,
  useSessionPool: true,
  persistCookiesPerSession: true,
  maxRequestsPerMinute,
  maxRequestsPerCrawl: maxProducts * 3,  // 3× headroom for pagination + retries
  retryOnBlocked: true,
  maxRequestRetries: 5,
  requestHandlerTimeoutSecs: 60,  // HTTP: 60s, browser: 120s
  sessionPoolOptions: {
    maxPoolSize: 20,
    sessionOptions: { maxUsageCount: 50, maxErrorScore: 3 },
  },
  requestHandler: router,
});

try {
  await crawler.run(startUrls.map((s) => ({ ...s, userData: { ...s.userData, label: 'CATEGORY' } })));

  await Actor.setStatusMessage(
    `Done. Scraped ${runtime.state.collectedCount} products.`,
    { isStatusMessageTerminal: true },
  );
  await Actor.exit();
} catch (err) {
  log.exception(err as Error, 'Unhandled crawler error');
  await Actor.pushData({
    error: true,
    errorType: (err as Error).constructor.name,
    message: (err as Error).message,
    timestamp: new Date().toISOString(),
  });
  await Actor.setStatusMessage(
    `Error: ${(err as Error).message}. See dataset.`,
    { isStatusMessageTerminal: true },
  );
  await Actor.exit();  // STILL SUCCEEDED - graceful exit pattern
}
```

**Rate-limit rule of thumb:** if the target advertises (or you observe) a hard per-IP limit of N req/min, set `maxRequestsPerMinute` to ~80% of N to leave retry headroom. For a 30 req/min target, use 24; for 60 req/min, use 48. Above 80% you hit 429 cascades; below 50% you waste capacity. With proxy rotation (sticky sessions per IP), the **effective** rate is `maxRequestsPerMinute × session_pool_size` - adjust accordingly.

`requestHandlerTimeoutSecs: 60` is the HTTP default; bump to 120 for `PlaywrightCrawler` (page navigation + render takes longer). Without an explicit value, slow pages block autoscaling indefinitely.

The graceful-exit pattern (push error to dataset → SUCCEEDED) is non-negotiable; treat it as core error-handling doctrine.

## `src/routes.ts` - Crawlee router with pagination + atomic charging

```typescript
import { createCheerioRouter } from '@crawlee/cheerio';
import { Actor, log } from 'apify';
import { parseProductCard } from './extractors/productCard.js';
import { runtime } from './runtime.js';

export const router = createCheerioRouter();

router.addHandler('CATEGORY', async ({ request, $, enqueueLinks, crawler }) => {
  const { state, config } = runtime;

  if (state.chargeLimitReached || state.collectedCount >= config.maxProducts) {
    await crawler.autoscaledPool?.abort();
    return;
  }

  const cards = $('[data-testid="product-card"], .product-item, [itemtype$="/Product"]').toArray();
  if (cards.length === 0) {
    log.warning(`No product cards on ${request.url} - possible site change.`);
    await Actor.pushData({
      error: true,
      errorType: 'EXTRACTION_FAILED',
      message: `No product cards found on ${request.url}.`,
      url: request.url,
      timestamp: new Date().toISOString(),
    });
    return;  // do not throw - let other pages succeed
  }

  for (const el of cards) {
    if (state.collectedCount >= config.maxProducts) break;

    const product = parseProductCard($(el), request.url);
    if (!product.name || !product.url) continue;  // discard junk before charging

    // Atomic charge + push - preferred pattern
    const charge = await Actor.pushData(product, 'item-scraped');

    if (charge.eventChargeLimitReached) {
      log.info('Charge cap reached - stopping enqueue.');
      state.chargeLimitReached = true;
      await crawler.autoscaledPool?.abort();
      return;
    }
    state.collectedCount += 1;
  }

  // Pagination - only enqueue next if we haven't hit the cap.
  // enqueueLinks silently no-ops when the selector matches nothing,
  // so the crawl naturally terminates on the last page.
  if (state.collectedCount < config.maxProducts) {
    await enqueueLinks({
      selector: 'a.next-page, [rel="next"], .pagination-next',
      label: 'CATEGORY',
    });
  }
});

router.addDefaultHandler(async ({ request, log }) => {
  log.warning(`Unrouted request: ${request.url}`);
});
```

**Multi-URL note:** `runtime.state.collectedCount` is a **global** cap across all `startUrls`. The first start URLs in the array get scraped to the cap; subsequent start URLs may be skipped if the cap is hit. Document this in the README. For true per-start-URL caps, key the counter on `request.userData.startUrlIndex` and check against an array of per-URL maxima.

## `src/extractors/productCard.ts` - defensive extraction

```typescript
import type { CheerioAPI } from 'cheerio';

export interface Product {
  name: string | null;
  price: number | null;
  priceRaw: string | null;
  currency: string | null;
  url: string;
  imageUrl: string | null;
  availability: 'InStock' | 'OutOfStock' | 'PreOrder' | null;
}

export function parseProductCard($card: ReturnType<CheerioAPI>, baseUrl: string): Product {
  const name = $card.find('h2, [itemprop="name"], .product-name').first().text().trim() || null;

  const priceRaw = $card.find('[itemprop="price"], .price, [data-price]').first().text().trim() || null;
  const priceMatch = priceRaw?.match(/[\d,.]+/);
  const price = priceMatch ? parseFloat(priceMatch[0].replace(/,/g, '')) : null;
  const currency = $card.find('[itemprop="priceCurrency"]').attr('content')
    ?? (priceRaw?.match(/[€$£¥]/)?.[0] ?? null);

  const href = $card.find('a').first().attr('href') ?? '';
  const url = href ? new URL(href, baseUrl).href : '';

  const imageUrl = $card.find('img').first().attr('src') ?? $card.find('img').first().attr('data-src') ?? null;

  const availabilityRaw = $card.find('[itemprop="availability"]').attr('content')
    ?? $card.find('[itemprop="availability"]').text().trim().toLowerCase();
  const availability = availabilityRaw?.includes('InStock') || availabilityRaw?.includes('in stock') ? 'InStock'
    : availabilityRaw?.includes('OutOfStock') || availabilityRaw?.includes('out of stock') ? 'OutOfStock'
    : availabilityRaw?.includes('PreOrder') ? 'PreOrder'
    : null;

  return { name, price, priceRaw, currency, url, imageUrl, availability };
}
```

Notes:
- **Layered fallbacks** (`'h2, [itemprop="name"], .product-name'`) - survive A/B tests and minor redesigns.
- **Prefer schema.org** (`[itemprop="..."]`) over visual class names - changes less often.
- **Return nulls, not undefined** - easier to query later in the dataset.
- **Pure function** - easy to test against fixtures with no Apify SDK.

Full selector strategy + schema validation patterns belong to your scraping selector-strategy doctrine.

## `src/ppe/charge.ts` - when not using the atomic shortcut

The atomic `Actor.pushData(data, 'event-name')` is preferred where the event corresponds to a dataset item. For non-dataset events (e.g. "search-performed", "page-scanned" where no dataset item is pushed), use the wrapper:

```typescript
import { Actor, log } from 'apify';

export async function chargeEvent(opts: { eventName: string; count?: number }): Promise<boolean> {
  const count = opts.count ?? 1;
  if (count <= 0) return false;

  const result = await Actor.charge({ eventName: opts.eventName, count });

  if (result.eventChargeLimitReached) {
    log.info(`Event ${opts.eventName} not charged: limit reached.`);
    return false;
  }
  log.info(`Charged event ${opts.eventName} × ${count}`);
  return true;
}
```

Then in handlers:

```typescript
const charged = await chargeEvent({ eventName: 'search-performed' });
if (!charged) {
  await crawler.autoscaledPool?.abort();
  return;
}
```

Full PPE-implementation and error-handling doctrine sits alongside this skill - consult your monetization and error-handling references.

## Anti-bot escalation inside Path C

Your scraping anti-bot doctrine has the full 5-rung escalation ladder.

Quick escalation triggers inside a Crawlee scraper:

| Symptom | Try next |
|---|---|
| HTTP 200 but selectors return nothing | Real browser needed → switch template to `ts-crawlee-playwright-chrome` |
| Sporadic 403/429 | Increase session pool, decrease `maxRequestsPerMinute`, retire bad sessions |
| Consistent 403 from datacenter IPs | Escalate proxy group: `DATACENTER` → `RESIDENTIAL` |
| Residential proxies still blocked | Fingerprint mismatch → Playwright with default Crawlee fingerprints |
| Playwright still detected | Camoufox: switch template to `ts-crawlee-playwright-camoufox` |
| Camoufox detected at scale | Managed API (Firecrawl / ZenRows / ScrapFly) - see your managed-scraping-API doctrine |

**Do not** start at Camoufox. The economics are 100× worse than Cheerio. Escalate only on observed failure.

## AI extraction inside a Crawlee route (hybrid Path B+C)

Sometimes a route fetches an HTML page that has wildly varying structure (different vendors, blog posts, etc.) and selectors cannot work. The hybrid pattern:

```typescript
import Anthropic from '@anthropic-ai/sdk';

router.addHandler('PRODUCT_DETAIL', async ({ request, $, body }) => {
  const cleanHtml = $.html().replace(/<script[^>]*>[\s\S]*?<\/script>/g, '').slice(0, 8000);

  const client = new Anthropic();
  const response = await client.messages.create({
    model: 'claude-haiku-4-5-20251001',  // cheap + fast
    max_tokens: 500,
    messages: [{
      role: 'user',
      content: `Extract product info from this HTML. Return JSON only: {"name": string, "price": number, "currency": string, "description": string}\n\n${cleanHtml}`,
    }],
  });

  const product = JSON.parse((response.content[0] as any).text);
  await Actor.pushData({ ...product, url: request.url }, 'item-scraped');
});
```

Full pattern (prompt template, cost-per-page math, LLM-cost handling strategies) in `references/path-b-ai-extraction.md`.

## Testing strategy

**Never hit the target site from the local machine.** Tests use captured fixtures.

```typescript
// tests/extractors.test.ts
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'fs';
import { load } from 'cheerio';
import { parseProductCard } from '../src/extractors/productCard.js';  // .js extension, even in tests, for ESM consistency

const fixtureHtml = readFileSync('tests/fixtures/category-page.html', 'utf-8');
const $ = load(fixtureHtml);

describe('parseProductCard', () => {
  it('extracts all fields from a real fixture', () => {
    const card = $('[data-testid="product-card"]').first();
    const product = parseProductCard(card, 'https://example-shop.com/category/laptops');

    expect(product.name).toMatch(/.+/);
    expect(product.price).toBeGreaterThan(0);
    expect(product.url).toMatch(/^https:\/\//);
  });

  it('returns null for fields missing in the fixture', () => {
    const incompleteFixture = load('<div data-testid="product-card"><h2>Foo</h2></div>');
    const card = incompleteFixture('[data-testid="product-card"]');
    const product = parseProductCard(card, 'https://example-shop.com');
    expect(product.price).toBeNull();
    expect(product.imageUrl).toBeNull();
  });
});
```

**ESM import note:** test files use the same `.js` extension on relative imports as production code (`from '../src/extractors/productCard.js'`). Vitest's resolver tolerates both forms, but consistency with `main.ts` / `routes.ts` avoids confusion when copy-pasting between test and prod code.

Fixtures live in `tests/fixtures/`, scrubbed of PII. Capture them once via the Apify Console (run the scraper, save the HTML response) and commit to git. The full fixtures-capture and scrubbing pattern belongs to your scraping testing doctrine.

## Running locally

```bash
# Install
npm install

# Set local input
cat > storage/key_value_stores/default/INPUT.json <<EOF
{
  "startUrls": [{ "url": "https://example-shop.com/category/laptops" }],
  "maxProducts": 20,
  "maxRequestsPerMinute": 30
}
EOF

# Run (wipes prior local storage)
apify run --purge

# Inspect
ls storage/datasets/default/
cat storage/datasets/default/000000001.json
```

In tests:

```bash
npx vitest run                # one-shot
npx vitest --watch            # local TDD only (never in CI)
```

`Actor.charge()` and `Actor.pushData(_, eventName)` are no-ops locally - charges only fire on the Apify platform. Log lines should still appear so you can audit charge correctness.

## Mid-run graceful abort

Three legitimate reasons to abort mid-crawl:
1. `eventChargeLimitReached` returned by a charge call
2. `maxProducts` reached
3. Selectors returned nothing on N consecutive pages (site changed)

The correct API is `crawler.autoscaledPool?.abort()` - NOT `Actor.exit()` (which would terminate the process before in-flight requests finalize) and NOT `process.exit()` (which skips graceful shutdown). After `abort()`, the existing crawler.run() Promise resolves, your main.ts continues to the `Actor.setStatusMessage` + `Actor.exit()` lines, the dataset is finalized.

This follows the core monetization principle: never abort mid-crawl in a way that drops in-flight work.

## Deployment & verification

See `references/checklist-publish.md` for the full pre/post-deploy checklist.

Quick smoke test after `apify push`:

1. Console → Runs → Start with a small input (5 products).
2. Confirm status SUCCEEDED, dataset has 5 items, "Billing" tab shows 5 `item-scraped` events charged.
3. Hit the URL with an intentionally bad input (empty `startUrls`) - verify it returns SUCCEEDED with an error record, not FAILED.
4. Source-code hygiene: `apify actors info <user>/<actor> --json | jq '.versions[0].sourceFiles[].name'` shows ONLY public files.

## What this file does NOT cover

- **Tool ladder reasoning (Cheerio vs Playwright vs Camoufox at deeper level)** - your scraping anti-bot doctrine.
- **Anti-bot escalation tactics, vendor signatures** - your scraping anti-bot doctrine.
- **Diagnostic (Wappalyzer-first, hidden APIs)** - your scraping diagnostic doctrine.
- **PPE doctrine** - your Apify monetization doctrine.
- **README / Store description / pricing copy** - your Apify Actor content-authoring doctrine.
- **Managed APIs (Firecrawl / ZenRows / ScrapFly) when Crawlee economics break** - your managed-scraping-API doctrine.
- **Path A no-code workflow** → `references/path-a-nocode.md`
- **Path B AI extraction (full pattern)** → `references/path-b-ai-extraction.md`
- **Pre/post-publish checklist** → `references/checklist-publish.md`
