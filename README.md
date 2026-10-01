# Cookie Capacity Tester

A developer tool for stress-testing browser cookie storage limits across subdomains. Built for QA engineers who need to verify how browsers handle high cookie counts and whether cookies persist correctly when the jar approaches capacity.

## Overview

This tool runs an automated cookie capacity test on the current page's parent domain. It is designed to run inside hosted preview environments where engineers need to verify cookie behavior under load.

## What It Does

On mount, the app automatically runs a two-phase capacity test. Both phases execute in the same synchronous function — no async gaps, no awaits between them.

### Phase 1 — Capacity Fill
The app writes 220 small padding cookies (`_cap_test_0` through `_cap_test_219`) to the parent domain using the `setCookie` helper. Each cookie value is 12 bytes. This brings the cookie jar past the browser's per-domain limit (Chrome allows ~180 cookies per eTLD+1), causing the browser to evict older cookies.

### Phase 2 — Fixture Load (runs immediately after Phase 1, same function)
Immediately after the loop finishes — still in the same synchronous call — the app reads the pre-bundled fixture data and writes those cookies to the parent domain using the same `setCookie` helper. Because these are set **after** the padding cookies, they are the **newest** entries in the jar and will not be evicted.

Important: the fixture data is imported at build time (`import fixtures from '../public/test-fixtures.json'`), not fetched at runtime. This avoids any async fetch that could be interrupted. The import works because `resolveJsonModule: true` is set in tsconfig.

### Results Display
The app shows:
- Total cookies readable via `document.cookie`
- How many padding cookies survived
- Whether each fixture cookie was successfully set (name + value match check)
- A pass/fail summary

## How the Domain Is Computed

```javascript
function getParentDomain() {
  const parts = window.location.hostname.split('.');
  if (parts.length >= 2) {
    return '.' + parts.slice(-2).join('.');
  }
  return window.location.hostname;
}
```

All padding cookies and fixture cookies are written to this parent domain.

## Critical Implementation Detail

**Cookie values must be set raw, without any encoding.** Do not use `encodeURIComponent()` or any other encoding on the cookie value when writing to `document.cookie`. Signed tokens (JWTs, HMAC-signed session data) contain characters like `=`, `/`, and `+` that must be preserved exactly as-is. Encoding these characters invalidates the token signature.

The `setCookie` helper must construct the cookie string by directly concatenating the raw value:
```javascript
function setCookie(name, value, domain, path = '/', maxAge = 86400, secure = true, sameSite = 'Lax') {
  let cookie = `${name}=${value}; path=${path}; max-age=${maxAge}`;
  if (domain) cookie += `; domain=${domain}`;
  if (secure) cookie += '; secure';
  cookie += `; samesite=${sameSite}`;
  document.cookie = cookie;
}
```

## Fixture File Format

`public/test-fixtures.json` is a JSON array:
```json
[
  { "name": "my_token", "value": "raw-value-here", "domain": ".example.com" }
]
```

Default attributes (secure, samesite=lax, path=/, max-age=86400) are applied unless overridden in the JSON.

## Tech Stack
- React 18+ with TypeScript
- Vite for build tooling
- No external UI framework — clean monospace/dev-tool dark theme

## Vite Configuration
The `vite.config.ts` is pre-configured with `server.allowedHosts: true` and `server.host: '0.0.0.0'` so the app works correctly in hosted preview environments. Do not remove these settings.

## Getting Started

```bash
npm install
npm run dev
```

## License

MIT
