---
name: offercheck
description: Read any public web page as clean Markdown with metadata and links ($0.001), verify a product offer (live price, currency, availability with evidence, $0.01), check a URL (alive, redirects, soft-404, domain age, robots policy for AI agents, archive snapshot, $0.002), convert currencies with ECB daily reference rates ($0.002), or get crypto/token prices by symbol or on-chain address with a confidence score ($0.002). Pay per call with USDC on Base via x402 — no signup, no API key. Use when you need page content without a browser, before quoting a price, citing a link, or paying another x402 service.
---

# offercheck

Base URL: `https://offercheck.offercheck.workers.dev` (OpenAPI: `https://offercheck.offercheck.workers.dev/openapi.json`, manifest: `https://offercheck.offercheck.workers.dev/.well-known/x402`)

## When to use
- You need the text of a web page (article, docs, product page, JSON API) without running a browser → `/v1/read` returns Markdown, title, description, canonical URL, links and a content hash.
- You are about to tell a user a product's price or that it is in stock → verify it first.
- You are about to cite or store a URL → check it is alive and not a soft-404 or a parked domain.
- You are about to pay an unknown x402 endpoint → check its domain age and robots policy first.
- You need to convert an amount between currencies (USD, EUR, GBP, JPY, ILS, INR, ... 30 ECB currencies), today or on a past date → `/v1/fx`.
- You need the current price of a coin or any on-chain token (Base, Ethereum, Solana, ...) in USD or another fiat → `/v1/price`.

## Endpoints (x402 v2, scheme `exact`, USDC on Base `eip155:8453`)
| Route | Price | Input | Output |
|---|---|---|---|
| `POST /v1/read` | $0.001 | `{"url": "https://example.com/post", "max_chars": 50000}` | `content` (Markdown; JSON/plain text passed through), `format`, `title`, `description`, `canonical`, `lang`, `author`, `published_time`, `links[{text,href}]`, `link_count`, `word_count`, `truncated`, `evidence{content_sha256,fetched_at,bytes}`, `warnings[]` |
| `POST /v1/verify/offer` | $0.01 | `{"url": "https://store.example/products/x"}` | `product{name,brand,sku,gtin}`, `offer{price,currency,availability,seller}`, `evidence{method,snippet,content_sha256,fetched_at}`, `confidence`, `warnings[]` |
| `POST /v1/check/url` | $0.002 | `{"url": "https://example.com/page"}` | `alive`, `final_url`, `redirect_chain`, `soft_404`, `domain{age_days,registrar}`, `robots{agents}`, `archive{available}`, `confidence`, `warnings[]` |
| `POST /v1/fx` | $0.002 | `{"from": "USD", "to": "EUR,GBP", "amount": 100, "date": "2024-01-15"}` (all optional; defaults USD→EUR, 1, latest) | `rates{EUR{rate,converted},...}`, `date` (ECB rate date), `source`, `warnings[]` |
| `POST /v1/price` | $0.002 | `{"ids": "btc,eth,base:0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "vs": "USD"}` (symbols, `coingecko:<slug>`, or `<chain>:<address>`; up to 25; `vs` any ECB fiat) | `prices{<id>{price,price_usd,symbol,confidence,priced_at}}`, `fx` (when vs≠USD), `source`, `warnings[]` |

Every route also accepts GET with the same fields as query parameters (`/v1/fx?from=USD&to=EUR&amount=100`, `/v1/price?ids=btc,eth&vs=EUR`). Free rate-limited mirrors for evaluation: `GET /demo/read?url=`, `GET /demo/verify/offer?url=`, `GET /demo/check/url?url=`, `GET /demo/fx?from=USD&to=EUR`, `GET /demo/price?ids=btc,eth`.

## How to call (any x402 client)
```bash
# Node: npm i @x402/fetch @x402/evm viem
node -e '
import("@x402/fetch").then(async ({ wrapFetchWithPayment }) => {
  const { privateKeyToAccount } = await import("viem/accounts");
  const f = wrapFetchWithPayment(fetch, privateKeyToAccount(process.env.PK));
  const r = await f("https://offercheck.offercheck.workers.dev/v1/verify/offer", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ url: process.argv[1] }) });
  console.log(await r.json());
})' "https://www.lego.com/en-us/product/millennium-falcon-75375"
```
Without a wallet the first response is HTTP 402 with a `PAYMENT-REQUIRED` header describing the payment; sign it and retry.

## Reading results
- `/v1/read`: `content` is the main region (`<main>`/`<article>`, else the body with navigation, footers, menus and hidden elements removed). No JavaScript is executed; a warning `very little text content` or `bot-block` means the page needs a browser. `truncated: true` means `max_chars` was hit — raise it (up to 200000).
- `confidence ≥ 0.75`: price and availability came from structured data (JSON-LD / Shopify product JSON) on the fetched page.
- `confidence ≤ 0.5` or a warning such as `redirected to a different page`, `bot challenge`, or `price not found`: the page could not be verified. Do not treat that as "offer is false"; say it could not be verified.
- Every result includes the fetch time and a SHA-256 of the fetched HTML so you can cite evidence.
- `/v1/fx`: rates are the European Central Bank's once-a-day reference rates (`date` tells you which day), not live interbank quotes; non-EUR pairs are cross rates through EUR. Good for conversions, reporting and sanity checks, not for executing trades.
- `/v1/price`: `confidence` (0–1) comes from DeFiLlama's aggregation; below 0.9 treat the price as indicative. `priced_at` is the price timestamp; quotes in a non-USD `vs` are USD prices converted with the ECB rate in `fx`.
