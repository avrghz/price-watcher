---
name: tracking-prices-nl
description: Use when the user gives a product name or model code (e.g. washing machine, dryer, appliance, electronics) and wants its price tracked daily across Dutch / Netherlands shops until they say stop, wants to know if now is the best time to buy, or says "stop tracking" a product.
---

# Tracking Prices in the Netherlands

## Overview
Daily, multi-shop price check for an exact product in NL, logged to a live dashboard, ending in a BUY / WAIT / NEUTRAL verdict backed by price history. **Every price must come from a fetched page — never a search snippet, never memory.**

## Start tracking (first request)
1. **Pin the exact SKU.** Search the code; list variants found (e.g. `RT90X8` vs `RT90X8YB` vs `RT90X8C`) with EAN, colour, capacity. Ask the user which one(s). Track only confirmed SKUs; never mix variants in one comparison.
2. **Run today's check** (below) once, interactively.
3. **Create dashboard artifact** with `db` capability (load `artifact-capabilities`). Collections: `products`, `prices`, `verdicts`.
4. **One cloud routine for all products** via `/schedule`, using this project's GitHub repo: daily 08:00 Europe/Amsterdam, prompt = "Run tracking-prices-nl daily check for dashboard <url>". If a routine already exists, just add the product to `products`. Store routine id in `products`.
5. Tell the user: dashboard link, routine time, how to stop.

## Daily check
**Check ALL sources in the table, every run.** Not reached = record `status: unreachable`, don't skip silently.

| Type | Sources |
|------|---------|
| History (fetch first) | tweakers.net/pricewatch (price history graph), kieskeurig.nl, beslist.nl |
| Major shops | coolblue.nl, bol.com, mediamarkt.nl, ep.nl, expert.nl, wehkamp.nl, amazon.nl |
| Appliance specialists | bemmelenenkroon.nl, deschouwwitgoed.nl, keukenloods.nl, + any shop Tweakers lists |

Per shop record: `date, sku, shop, price_incl_btw, delivery_cost, total_delivered, stock, condition, promo_end, url, source=fetched`.
- `total_delivered` = price + delivery. Compare **only** this. Pickup discounts go in a note.
- New only; skip refurbished / B-keuze / outlet unless user opts in.
- **Find the product page, don't guess URLs.** Use links from Tweakers/Kieskeurig listings, or search `site:<shop> <SKU>`. Save each found product URL in `products` and reuse it next run. Search results pages are not price sources.
- **Blocked (403/405/503, bot wall) → Playwright is mandatory**, not optional. Only then `unreachable`.
- **Cookie walls:** click "Weigeren" / "Alleen noodzakelijk" if offered, else accept. Never log in or enter personal data.
- Aggregator-only price (shop page unreachable): record with `source=aggregator`. It may be shown, but never be named "cheapest" unless shop-confirmed.
- Cashback / free gifts: record in a note; subtract cashback only if confirmed on the product page; never value gifts.
- Prices on NL consumer sites are incl. BTW unless stated otherwise.
- One run per day: if today's rows exist, update don't duplicate.

## Verdict
Use own log + Tweakers history (fills early days).
- **BUY**: today's lowest `total_delivered` ≤ historical low +3%, OR ≥10% below 90-day median.
- **WAIT**: above 90-day median AND a sales event within 21 days (Black Friday = last Fri of Nov, Cyber Monday, Jan sales), or a clear downward trend.
- **NEUTRAL**: otherwise.
- No history at all → say so; verdict max confidence "low". If a sales event is within 21 days, verdict NEUTRAL + note "promo prices run until <date>; waiting costs nothing / check again on <event>". Confidence rises as own log reaches 7 / 30 days.

Output: verdict, cheapest shop + link + total, vs low/median, reason in one line. On change to BUY → `PushNotification` if available; always also set an `alert` field on today's `verdicts` row so the dashboard shows a banner.

## Stop tracking
"Stop tracking X" → mark product `stopped` in `products`, keep history. If no `active` products remain → delete the routine (`/schedule`). "Stop everything" → stop all, delete routine. Confirm to user.

## Common mistakes
| Mistake | Fix |
|---|---|
| Quoting snippet price (€625 vs real €629) | Fetch the page |
| Checking 3 shops, calling it done | Every row of the source table |
| Comparing pickup price to delivered price | `total_delivered` only |
| Mixing YB/C variants | Confirmed SKU only |
| Seasonal guess as "data" | State history depth + confidence |
| Skipping Playwright "to save effort" | Required for every blocked source |
| Guessed shop URL → 404 → "not found" | Find URL via aggregator link / site search |
| Reading half a search page → "not found" | Use the product page, not search results |
