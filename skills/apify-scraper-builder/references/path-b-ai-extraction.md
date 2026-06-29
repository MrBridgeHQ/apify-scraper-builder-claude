# Path B — AI Extraction

Use Path B when the target's data shape varies wildly between pages and writing maintainable selectors is **impossible** (not just annoying). The LLM does the extraction work instead of CSS selectors.

This file assumes you've read the main `SKILL.md`. It expands Path B specifics.

## When AI extraction is the right answer (and when it isn't)

**Use AI extraction when:**
- ✅ You're aggregating data from many different sites with **no common HTML schema** (e.g. 50+ retailer product pages)
- ✅ The content is **prose** that needs structured extraction (articles, reviews, descriptions → tagged fields)
- ✅ The target **uses dynamic class names** that change every deploy (Stripe-style hashed classes)
- ✅ You need **schema-evolving** extraction (the data shape evolves as you discover new fields per page)

**Don't use AI extraction when:**
- ❌ The site has stable, well-structured HTML — even with messy class names, schema.org markup usually exists. Use Path C with `[itemprop="..."]` selectors.
- ❌ Volume > 100k pages/month — LLM cost ($0.001–$0.10/page) destroys the unit economics. Path C with selectors costs ~$0.00001/page.
- ❌ You need < 100 ms latency per page — LLM calls are 500 ms–5 s.
- ❌ The site has an internal JSON API — use it (hunt for it in the DevTools Network panel).

The first move on **any** target is still the diagnostic phase — internal API first, schema.org markup second, selectors third, LLM fourth.

## Two patterns

### Pattern B1 — Use `apify/ai-web-scraper`

No-code-ish. Configure via the Console:
- Start URLs (the pages to extract from)
- Output schema (describe the fields you want as JSON Schema)
- Optionally, custom LLM provider key (BYO-key — see "LLM-cost handling" below)

The Actor uses an LLM under the hood to extract per your schema. Best for one-off aggregations.

**Trade-offs:**
- ✅ Zero code, fastest to results
- ❌ Limited control over prompts, retries, cost optimization
- ❌ You don't own the Actor (no PPE revenue)
- ❌ LLM provider and model are picked for you (usually OpenAI or Claude — varies)

### Pattern B2 — Path C + LLM call inside a route handler

You own the Actor; Crawlee fetches pages; your code calls Claude/GPT inside the route handler.

```typescript
// src/routes.ts
import { createCheerioRouter } from '@crawlee/cheerio';
import { Actor, log } from 'apify';
import Anthropic from '@anthropic-ai/sdk';
import { z } from 'zod';

const ProductSchema = z.object({
  name: z.string().nullable(),
  price: z.number().nullable(),
  currency: z.string().nullable(),
  description: z.string().nullable(),
  inStock: z.boolean().nullable(),
});

export const router = createCheerioRouter();

router.addHandler('PRODUCT_DETAIL', async ({ request, $, log }) => {
  // 1. Clean the HTML — keep only content, drop scripts/styles
  const cleaned = $('body').clone();
  cleaned.find('script, style, noscript, svg, iframe').remove();
  const html = cleaned.html()?.slice(0, 12_000) ?? '';  // cap tokens

  // 2. Call the LLM
  const client = new Anthropic();
  const response = await client.messages.create({
    model: 'claude-haiku-4-5-20251001',  // cheap & fast for extraction
    max_tokens: 600,
    messages: [{
      role: 'user',
      content: `Extract product info from this HTML.

Return JSON ONLY (no markdown fences), matching this exact schema:
{
  "name": string | null,
  "price": number | null,
  "currency": string (ISO 4217) | null,
  "description": string | null,
  "inStock": boolean | null
}

HTML:
${html}`,
    }],
  });

  // 3. Parse + validate
  const raw = (response.content[0] as any).text.trim();
  let product;
  try {
    product = ProductSchema.parse(JSON.parse(raw));
  } catch (err) {
    log.warning(`LLM response not valid JSON or schema mismatch: ${(err as Error).message}`);
    await Actor.pushData({
      error: true,
      errorType: 'EXTRACTION_FAILED',
      message: 'LLM response did not match schema.',
      url: request.url,
      rawLlmOutput: raw.slice(0, 500),
    });
    return;  // do not charge
  }

  // 4. Push + charge atomically
  const charge = await Actor.pushData({ ...product, url: request.url }, 'item-extracted');
  if (charge.eventChargeLimitReached) {
    log.info('Charge cap reached.');
  }
});
```

## Prompt engineering for extraction

The single biggest cost lever is **how much HTML you send to the LLM**. Strategies in order of impact:

### 1. Strip noise before sending

```typescript
const cleaned = $('body').clone();
cleaned.find('script, style, noscript, svg, iframe, header, footer, nav, aside').remove();
const html = cleaned.html() ?? '';
```

Reduces tokens by 50–80%. Always do this.

### 2. Use schema.org or microdata markup if present

If the page has `<div itemtype="schema.org/Product">...</div>`, extract just that subtree:

```typescript
const productEl = $('[itemtype$="/Product"]').first();
if (productEl.length > 0) {
  // Use Path C with selectors — LLM not needed!
  return parseSchemaOrgProduct(productEl);
}
// Otherwise fall back to LLM
```

This is the **hybrid Path B+C pattern** — use selectors when markup is present, LLM as fallback. Cuts LLM calls by 60–90% on real-world e-commerce sites.

### 3. Truncate, don't expand

`html.slice(0, 12_000)` caps at ~3,000 tokens. Above that, costs spike and most LLMs lose attention.

For pages with relevant content at the bottom, sniff the section first:

```typescript
const productSection = $('main, [role="main"], #content, .product').first().html() ?? $('body').html() ?? '';
const truncated = productSection.slice(0, 12_000);
```

### 4. Use a small model

For extraction (not generation), Haiku-class models match Sonnet/Opus quality at ~10× lower cost. Defaults:

| Task | Model | Reason |
|---|---|---|
| Pure extraction from cleaned HTML | `claude-haiku-4-5-20251001` | Best price/perf for structured output |
| Extraction with reasoning ("infer category from description") | `claude-sonnet-4-6` | Trade-off |
| Multi-step extraction with chained calls | Custom orchestration | Beyond this skill's scope — consult the Claude/Anthropic API docs |

### 5. Use prompt caching for repeated prompts

If you're extracting the SAME shape from THOUSANDS of pages, the schema instructions are stable. Use Anthropic prompt caching to cache the system prompt:

```typescript
const response = await client.messages.create({
  model: 'claude-haiku-4-5-20251001',
  max_tokens: 600,
  system: [
    {
      type: 'text',
      text: `You extract product info. Schema: {...}. Return JSON only.`,
      cache_control: { type: 'ephemeral' },  // cached across calls
    },
  ],
  messages: [{ role: 'user', content: `HTML:\n${html}` }],
});
```

Reduces system-prompt cost to ~10% of normal after the first call. See the Anthropic prompt-caching docs for full details.

## Cost math — when AI extraction breaks the bank

Cheerio (Path C, selectors): ~$0.00001 per page (compute + proxy only)

AI extraction (Path B2, Haiku, 3k tokens in / 200 out):
- Input: 3000 × $1/M = $0.003
- Output: 200 × $5/M = $0.001
- Total: **$0.004/page** ≈ 400× more expensive than selectors

At 1k pages/run with PPE $0.001/item:
- Selectors revenue: $1, cost: $0.01 → 99% margin
- AI extraction revenue: $1, cost: $4 → **$3 LOSS per run**

You MUST raise the PPE price to ~$0.005–0.025/page to be profitable with AI extraction. Or charge a per-page LLM fee separately. See LLM-cost handling below.

## LLM-cost handling — pricing strategies

When the Actor calls an LLM internally, the developer must decide how to handle the LLM provider's per-call cost. Four strategies:

| Strategy | What | When |
|---|---|---|
| **1A — Two-tier transparency** | Developer's PPE event + sub-Actor pass-through, sub-Actors named/hyperlinked in README | Enterprise B2B audience that audits cost lines |
| **1B — Two-tier implicit** | Sub-Actor pass-through but NOT enumerated in pricing copy | Avoid — bad reviews about hidden costs |
| **2 — Absorbed cost** | PPE event price includes the LLM cost in the unit price | Self-serve users, simple "one price" UX |
| **3 — BYO-key** | User provides their own OpenAI/Anthropic key; developer charges only orchestration overhead | Developer-audience Actors |

For AI-extraction scrapers, **Strategy 2 (absorbed)** is the most common. Strategy 3 (BYO-key) works for developer-audience tooling. **Strategy 1B is an anti-pattern** — never hide LLM cost.

For the full discussion and decision criteria, consult your Apify monetization doctrine on LLM-cost handling.

## Reliability patterns specific to AI extraction

LLMs hallucinate. Plan for it.

### 1. Validate every response against a Zod / JSON Schema

Already shown above with `ProductSchema.parse(JSON.parse(raw))`. If parsing or validation fails, push an error record and don't charge. Don't trust the LLM blindly.

### 2. Retry once with a strict reprompt

```typescript
async function extractWithRetry(html: string, attempt = 1): Promise<Product | null> {
  const response = await callLlm(html, attempt === 2 ? STRICT_PROMPT : NORMAL_PROMPT);
  try {
    return ProductSchema.parse(JSON.parse(response));
  } catch {
    if (attempt < 2) return extractWithRetry(html, 2);
    return null;
  }
}
```

`STRICT_PROMPT` adds "Return ONLY valid JSON. No prose. No markdown." Often fixes the second-try failures.

### 3. Score confidence and gate the charge

```typescript
const charge = await Actor.pushData(product, 'item-extracted');
```

Only charge after schema validation passes. Charging for unvalidated LLM output is a refund magnet.

### 4. Cache LLM responses

Same page extracted twice in a month should hit the cache. Use the KV Store with a 30-day TTL keyed by the page URL hash:

```typescript
const cacheKey = `llm-extract:${md5(url + html.slice(0, 1000))}`;
const cached = await store.getValue(cacheKey);
if (cached && Date.now() - cached.ts < 30 * 24 * 3600 * 1000) {
  return cached.product;  // skip LLM call AND skip charging
}
// ... call LLM, then:
await store.setValue(cacheKey, { product, ts: Date.now() });
```

This is also a PPE consideration — caching means re-running the same input is free for the user (no charge on cache hit). Document this in the README so users understand the pricing model.

## Memory & timeout tuning for AI extraction

LLM calls are slow (500 ms–5 s each). Configure:

```json
{
  "minMemoryMbytes": 1024,
  "maxMemoryMbytes": 4096,
  "defaultRunOptions": { "memoryMbytes": 2048, "timeoutSecs": 7200 }
}
```

In Crawlee:

```typescript
new CheerioCrawler({
  // ...
  requestHandlerTimeoutSecs: 120,  // double the LLM timeout
  maxConcurrency: 5,                // LLM calls are I/O-bound but rate-limited
  maxRequestsPerMinute: 30,         // respect LLM provider's rate limits
});
```

## When to NOT use Path B at all

If after reading this you're still on the fence, default to Path C with selectors. The pattern "AI extraction is easier than writing selectors" usually evaporates when you do the cost math. The legitimate cases are narrow (multi-vendor aggregation, prose extraction). Everything else, selectors win.

## See also

- **Claude API patterns (prompt caching, model choice, batch, citations)** — the Anthropic API docs.
- **LLM cost-handling strategies (1A/1B/2/3) full doctrine** — your Apify monetization doctrine.
- **Schema.org / `[itemprop]` selectors (the cheap escape from AI extraction)** — your scraping selector-strategy doctrine.
- **Path C base architecture (where the LLM call slots in)** → `references/path-c-crawlee-templates.md`
