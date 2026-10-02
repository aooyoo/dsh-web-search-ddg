# dsh-web-search-ddg

[中文](README.zh.md) | English

Zero-token web search provider for the [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (DSH) web capability seam (`ctx.web`).

DSH's shipped search route (`deepseek-official`) performs every `web_search` as a **full billed model round trip** on `deepseek-v4-flash` — even when your session model is something else entirely. This plugin replaces that with two zero-cost engines tried in order, first success wins:

1. **Bing** — plain fetch against Bing's HTML endpoint (with a cookie bootstrap). No browser needed, sub-second when healthy.
2. **DuckDuckGo** — drives your local Chrome/Edge/Chromium headless against DuckDuckGo's HTML endpoint and parses the dumped DOM.

- **Zero model tokens** per search — no API key, no auxiliary model request
- **Zero dependencies** — Node builtins only; no Playwright/Puppeteer download
- **Engine fallback** — if one engine is blocked or its markup changes, the other answers; a failure is reported only when every engine misses
- **Keeps the shipped provider registered** — switching back is a one-line config change, not an uninstall

## Requirements

- A DSH host (≥ `0.1.0-rc`) providing the `ctx.web` seam
- For the DuckDuckGo engine: a local Chromium-family browser, detected automatically on macOS (Chrome, Edge, Chromium) and Linux (`/usr/bin/chromium`, `/usr/bin/google-chrome`); override with [`chromePath`](#configuration). The Bing engine needs no browser.
- Node.js ≥ 18 (`AbortSignal.any` used for fetch timeouts when available)

## Install

In your DSH profile directory (e.g. `~/.dsh/profiles/web`), install the package as an out-of-tree plugin:

```bash
pnpm add dsh-web-search-ddg
```

Declare it as a profile bundle so DSH loads it as a first-class layer (visible in the plugin manager, clean removal):

```json
// profile package.json → dsh.profile.bundles
"bundles": [..., "dsh-web-search-ddg"]
```

Then select the provider in the profile's `cordis.patch.yml`:

```yaml
# Select this provider for the model-facing web_search tool.
- id: web
  config:
    searchProvider: ddg-browser
```

> Note: a patch row **replaces the target row's whole config** (no deep merge), so the `web` row must restate every key — the shipped row owns only `searchProvider`. The `insert` of `web-search-ddg` itself comes from the package's `dsh.bundle` patch and needs no manual row.

The shipped `web-search-deepseek` row stays untouched: its provider remains registered and available, so switching back is one line (`searchProvider: deepseek-official`). DSH's selection is a single explicit id, **not** a priority chain — there is no silent fallback by design.

Verify the composed tree without starting the host:

```bash
dsh --profile web --dump-config | grep -A2 searchProvider
```

Restart the host to apply (host-side plugin rows do not hot-reload).

## Configuration

All keys optional; the row config goes to the `insert` entry above.

| Key | Default | Meaning |
| --- | --- | --- |
| `engines` | `["bing", "duckduckgo"]` | Engine execution order. Supported: `"bing"`, `"duckduckgo"`. First success answers. |
| `chromePath` | first detected browser | Absolute path to a Chromium-family executable (DuckDuckGo engine only). |
| `timeoutMs` | `20000` | Per-attempt budget for the DuckDuckGo engine. On timeout, buffered DOM output still counts as success (Chrome's `--dump-dom` process often lingers after printing). Two attempts run per search; keep `2 × timeoutMs` under `tool-web`'s `searchTimeoutMs` (DSH ships 60s). |
| `virtualTimeBudgetMs` | `8000` | Chrome's `--virtual-time-budget` — how long the page may settle before the DOM is dumped. |

## Behavior notes

- **Latency**: the Bing engine typically answers in well under a second; the DuckDuckGo engine takes ~10–20s (the DOM dump is ready quickly, but Chrome often fails to exit on its own, so results land at the timeout guard — buffered output is still accepted).
- **Bing quality needs cookies.** Without a bootstrapped cookie jar Bing serves degraded results that ignore most of the query. The plugin visits the Bing homepage once per session and replays those cookies; if results come back empty it re-bootstraps once.
- **Search engines rate-limit aggressive IPs.** Heavy automated searching from one machine can earn connection resets or challenge pages from every engine at once. When that happens the plugin fails with an aggregated message naming each engine's failure — switch networks or wait for the flag to decay (hours), or switch `searchProvider` back to `deepseek-official`. It never silently degrades.
- **The headless UA is overridden** with a plain desktop Chrome UA — DuckDuckGo keys on the `HeadlessChrome` marker and blocks it otherwise.
- Each browser attempt runs in a throwaway `--user-data-dir` under the OS temp dir, cleaned up best-effort after the process dies.

## How it works

**Bing engine** — `fetch` the SERP with desktop-Chrome headers and bootstrapped cookies; parse `<li class="b_algo">` blocks for title + snippet; unwrap result links (`/ck/a?…&u=a1<base64url>`) into real target URLs.

**DuckDuckGo engine** — spawn the browser headless (`--headless --dump-dom --virtual-time-budget`, UA overridden) against `https://html.duckduckgo.com/html/?q=<query>`; read the serialized DOM with a kill guard (buffered output at timeout still counts); parse `<a class="result__a">` titles and `<a class="result__snippet">` snippets, decoding the real URL from each redirect's `uddg` parameter and pairing on it.

Both engines return `{ sources: [{ url, title?, snippet? }], truncated: false }` through the seam — the model-facing `web_search` tool and result cards work unchanged.

## Troubleshooting

`ddg-browser: all search engines failed (…)` names every engine's last error:

| Message fragment | Meaning | Remedy |
| --- | --- | --- |
| `connection reset / rate-limited` | Your IP is temporarily flagged by the engine | Switch networks (e.g. phone hotspot) or wait hours; flags decay |
| `anomaly/challenge page` | DuckDuckGo anti-bot challenge served | Same as above; also check the UA string still matches a current Chrome |
| `no parsable results/links` | The engine's markup changed | Update this plugin, or file an issue |
| `browser did not finish within Nms` | Browser hung without producing DOM | Raise `timeoutMs`; check the browser launches at all |

## Development

```bash
npm test          # end-to-end: registers on a stub ctx.web and runs one real search
CHROME_PATH=/path/to/browser npm test
TEST_QUERY="something else" npm test
```

## License

[MIT](LICENSE)
