# Web Serving Infrastructure

How `viret.ai` is hosted, served, and secured. Last updated 2026-07-05.

## Overview

`viret.ai` is a static single-page site. The request path is:

```
Visitor → Cloudflare (DNS + edge SSL + proxy) → GitHub Pages (origin) → index.html
```

| Layer | Provider | Role |
|-------|----------|------|
| Domain registrar | GoDaddy | Owns/renews the `viret.ai` and `veret.ai` domains |
| Authoritative DNS | Cloudflare (free plan) | Answers DNS queries; proxies + terminates TLS |
| Origin host | GitHub Pages | Serves the static site from this repo |
| Source of truth | This repo (`Subsovereign/viret`) | `main` branch = production |

## Repository & deployment

- **Repo:** `Subsovereign/viret` (public — GitHub Pages is not available for private repos on the current plan).
- **Deploy:** push to `main`. GitHub Pages rebuilds and publishes automatically. No CI/CD config required.
- **Site files:**
  - `index.html` — the entire site (single page, inline CSS/JS).
  - `CNAME` — contains `viret.ai`; tells GitHub Pages the custom domain.
  - `.nojekyll` — disables Jekyll processing so files are served as-is.

## DNS (Cloudflare)

Nameservers at GoDaddy point to Cloudflare: `pete.ns.cloudflare.com` / `rosalyn.ns.cloudflare.com`.

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| A | `@` | `185.199.108.153` | Proxied |
| A | `@` | `185.199.109.153` | Proxied |
| A | `@` | `185.199.110.153` | Proxied |
| A | `@` | `185.199.111.153` | Proxied |
| CNAME | `www` | `subsovereign.github.io` | Proxied |

The four A records are GitHub Pages' anycast IPs. Because they are **proxied** (Cloudflare orange-cloud), public DNS resolves `viret.ai` to Cloudflare's edge IPs, not GitHub's directly.

> Note: the apex (`@`) must use A records — DNS does not allow a CNAME on the root domain. Only subdomains like `www` can be CNAMEs.

## TLS / HTTPS

- **Certificate is issued and served by Cloudflare's edge** (Google Trust Services), not by GitHub.
- **SSL/TLS mode:** `Full`. (`Full (strict)` would fail — GitHub Pages has no valid `viret.ai` origin certificate. `Flexible` would cause a redirect loop.)
- **Always Use HTTPS:** enabled — all `http://` requests 301-redirect to `https://`.

### Why Cloudflare instead of GitHub's built-in HTTPS

GitHub Pages' own Let's Encrypt provisioning failed persistently with `bad_authz` despite verifiably correct DNS. Fronting the site with Cloudflare issues a valid certificate instantly and removes the dependency on GitHub's provisioning. **Do not fight GitHub's Let's Encrypt flow — Cloudflare is the certificate authority in this architecture.**

## Domain routing

- `viret.ai` and `www.viret.ai` → the site (www 301-redirects to the apex).
- `veret.ai` — separate typo-catch domain, kept at GoDaddy with a 301 forward to `https://viret.ai`. Not on Cloudflare.

## Privacy note

The contact email in `index.html` is assembled at runtime by JavaScript from reversed fragments, so the plaintext address never appears in the static HTML that scrapers read. Keep it that way — do not hardcode a plaintext `mailto:` address.

## Runbook

- **Update the site:** edit `index.html`, commit, push to `main`. Live within ~1–2 min.
- **Cache:** Cloudflare may cache assets; purge via the Cloudflare dashboard if a change doesn't appear.
- **Verify HTTPS end-to-end:**
  ```sh
  curl -sSL -o /dev/null -w "final=%{http_code} url=%{url_effective}\n" http://viret.ai
  # expect: final=200 url=https://viret.ai/
  ```
