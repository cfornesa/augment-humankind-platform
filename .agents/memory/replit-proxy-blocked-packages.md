---
name: Replit package proxy blocked packages
description: How to fix es5-ext (protestware block) and npm-run-path (proxy URL rewrite issue) during npm install on Replit.
---

# Replit Package Proxy: Blocked/Broken Packages

## es5-ext — Blocked as Protestware

Replit's Socket Security Policy blocks `es5-ext` versions from 0.10.57 onwards (and some earlier ones). Version 0.10.56 is the highest confirmed-available version. It's pulled in transitively by `drizzle-kit` → `json-diff` → `cli-color`.

**Fix:**
1. Add to root `package.json` overrides: `"es5-ext": "0.10.56"`
2. Patch `package-lock.json` to set the entry to `0.10.56` with a direct proxy URL:
   `http://package-firewall.replit.local/npm/es5-ext/-/es5-ext-0.10.56.tgz`

**Why:** The Replit proxy blocks `es5-ext >= 0.10.57` as protestware. The override alone isn't enough — the lockfile resolved URL must also be patched to use the direct proxy URL format (with `/npm/` prefix), otherwise npm fetches via the wrong rewritten path.

## npm-run-path — Proxy URL Rewriting Bug

`npm-run-path@6.0.0` returns 404 from the Replit proxy when fetched via a lockfile resolved URL (`https://registry.npmjs.org/npm-run-path/-/npm-run-path-6.0.0.tgz`). The proxy rewrites `https://registry.npmjs.org/PACKAGE/-/...` to `http://package-firewall.replit.local/PACKAGE/-/...` (missing the `/npm/` prefix), which gives 404.

**Fix:** Patch the lockfile entry to use the direct proxy URL:
```
"resolved": "http://package-firewall.replit.local/npm/npm-run-path/-/npm-run-path-6.0.0.tgz"
```
And also add the nested `path-key@4` entry with a direct proxy URL:
```
"node_modules/npm-run-path/node_modules/path-key": {
  "version": "4.0.0",
  "resolved": "http://package-firewall.replit.local/npm/path-key/-/path-key-4.0.0.tgz"
}
```

**Why:** The proxy correctly serves the package at the `/npm/` prefixed path, but only when the lockfile's resolved URL uses the direct proxy format. The automatic rewrite from `registry.npmjs.org` URLs strips the `/npm/` prefix for non-scoped packages.

## General Pattern

When any package gives 404 or 403 during `npm install`:
1. Check with `curl -s -o /dev/null -w "%{http_code}" "http://package-firewall.replit.local/npm/PKGNAME/-/PKGNAME-VERSION.tgz"`
2. If 200 at `/npm/` path but 404 without it → patch lockfile resolved URL to use direct proxy path
3. If 403 (protestware) → find highest available version with the same curl test, add override + patch lockfile
4. Remove the lockfile entry entirely and let npm re-resolve if patching is too complex

## How to apply

Any time `npm install` fails with 404/403 on Replit — check the proxy URL format and whether the package is blocked.
