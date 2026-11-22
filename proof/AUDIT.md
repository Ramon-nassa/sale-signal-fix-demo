# Tracking audit: findings, fixes and evidence

Audited system: [funil-aberto](https://github.com/thiagopatzdorf/funil-aberto) (open source, MIT, Genesis Innovation),
upstream commit `7d85b53`. Run on 2026-11-21. Every buyer, e-mail, phone and click id is fictional test data.

## 1. How it was tested

1. An audit script starts the real funnel HTTP server and a **fake Graph API** that records every Conversions API call.
   The only patch is the Graph base URL. Each finding is an assertion on what actually left the server, not a code
   reading. Results: [evidence-before.json](evidence-before.json) (upstream code) and
   [evidence-after.json](evidence-after.json) (with the fix).
2. A live run sends one clean journey (InitiateCheckout + Purchase) to a **real Meta test pixel**, always with a
   Test Events code. Result: [live-baseline.json](live-baseline.json) (token, pixel id and test code not logged).
3. The upstream test suite still passes, plus 6 tests for the fixes: **70/70**.

## 2. Findings (7/7 reproduced on the upstream code)

| # | Finding | Evidence from the run | Business impact |
|---|---|---|---|
| F1 | No browser Pixel. Meta only gets `InitiateCheckout` and `Purchase` from the server | No `fbq` / `fbevents.js` in the pages; 6 events tracked internally, 2 sent to Meta; 0 browser events share the `event_id` | Meta optimizes on 2 signals; nothing to deduplicate, so Events Manager can never show "Deduplicated" |
| F2 | Tracker snippet only works on pages served by the funnel | JS posts to a relative `/t`; `OPTIONS /t` → 501; no CORS header | "One line on your site" fails on any other domain |
| F3 | First touch is never updated | Sale came from a new ad; Meta received the click id of the old one | Wrong click sent to Meta; retargeting gets no credit |
| F4 | Dashboard stops at campaign level | `utm_content` stored but never shown | Cannot answer "which ad sold" |
| F5 | Meta send has no queue or retry and runs inside the buyer's request | Graph returned 500 → 1 attempt, 0 resends; checkout took **3.01 s** when Graph took 3 s | Every Meta outage silently loses sales; a slow Meta slows the checkout |
| F6 | Low match quality | `user_data` keys sent: `em, ph, fbc, client_ip_address, client_user_agent`. Missing: `fbp, external_id, fn, ln, country` | Fewer sales matched to a person |
| F7 | `client_ip_address` taken from an unchecked `X-Forwarded-For` | Header `203.0.113.66` was sent to Meta as the buyer's IP | Anyone can poison the IP sent to Meta |

## 3. What was fixed

| # | Status | What changed | After (same script) |
|---|---|---|---|
| F1 | Fixed | Browser Pixel with PageView, ViewContent, InitiateCheckout and Purchase. Order events use the server's `event_id` (`<order>-<event>`), so Meta deduplicates. The page security policy opens only Meta's two domains, and only when a Pixel is set | 4 browser events, same id as the server |
| F5 | Fixed | Outbox table: the send runs off the buyer's request; failures retry at 30 s, 2 min, 10 min, 1 h, 6 h, then stop with the error kept | Checkout 3.01 s → 0.01 s when Meta is slow; 2 events resent after the outage, 0 lost |
| F6 | Fixed | Server reads the Pixel's `_fbp` / `_fbc` cookies; adds `external_id` (same hash in the Pixel), `fn`, `ln`, `country`, `order_id`; Pixel gets advanced matching on the order page | `user_data` keys 5 → 10 |
| F3 | Fixed on the Meta side | `fbc` now comes from the Pixel cookie, which always holds the latest ad click | — |
| F2, F4, F7 | Out of scope | Hardening and reporting; they do not change what Meta learns | — |

## 4. Proof on Meta's own screens (test pixel, 2026-11-21)

1. **Test Events:** InitiateCheckout and Purchase shown as **Deduplicated** (browser + server, one event).
2. **User data keys:** 9 shown by Meta (was 4): country, e-mail, external id, click id, browser id, first name,
   last name, phone, user agent. The IP was dropped because the test ran on a private network.
3. **Custom data:** 5 parameters (was 3).
4. **Event Match Quality:** 7.7 for InitiateCheckout on the test dataset overview.

Screenshots are shown on a call; the test pixel and business ids stay private.
