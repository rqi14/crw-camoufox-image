# crw-camoufox-image

Builds [`us/crw`](https://github.com/us/crw) (fastCRW) with the `camoufox`
Cargo feature enabled, which the official `ghcr.io/us/crw` image does not
ship (it's compiled with `--features cdp` only). This feature is required
for `crw-server` to talk to a [`camofox-browser`](https://github.com/redf0x1/camofox-browser)
sidecar (`[renderer.camoufox]` in `config.toml`), which is the only renderer
tier that can pass Cloudflare-style fingerprint/bot challenges.

## Patches

Every `*.patch` in the repo root is applied to the freshly-cloned upstream at
build time, in filename order.

| Patch | What it adds |
|---|---|
| `wait-for-challenge.patch` | `renderer.camoufox_challenge_wait_ms` — polls a Camoufox tab while a Cloudflare-style JS challenge clears instead of evaluating once and returning the interstitial. Env: `CRW_RENDERER__CAMOUFOX_CHALLENGE_WAIT_MS`. |
| `configurable-escalation.patch` | `extraction.lightpanda_escalation_renderer` — the renderer forced when a LightPanda-tier fetch returns thin content. Upstream hardcodes `"chrome"` (`crw-crawl/src/single.rs`), so a deployment without a Chrome CDP sidecar gets `requested renderer 'chrome' not in pool` and never reaches stronger tiers like camoufox. Defaults to `"chrome"` — unchanged behaviour unless set. Env: `CRW_EXTRACTION__LIGHTPANDA_ESCALATION_RENDERER`. |
| `reject-binary-bodies.patch` | Bug fix, no new config. `crw-renderer/src/http_only.rs` keys its PDF branch on `content_type == "application/pdf"` and treats **everything else** as text, so a non-HTML non-PDF body is UTF-8-lossy'd into the HTML extractor: a `.docx`/`.xlsx`/`.pptx` returns `markdown` starting `PK...[Content_Types].xml` under `success: true`, indistinguishable from a real scrape. Now a `%PDF-` magic sniff relabels mislabelled PDFs (so the crw-crawl PDF branch engages), and a NUL-byte-in-first-1KB check turns other binary bodies into an `HttpError` instead of garbage. Proposed upstream. |

No source is forked or modified — the workflow clones upstream `us/crw` at
build time and rebuilds their own `Dockerfile` with one extra `--build-arg`
(`CARGO_PKGS=-p crw-server --features cdp,camoufox -p crw-mcp -p crw-cli`).

Runs on GitHub-hosted runners rather than locally: upstream's own Dockerfile
notes this build previously OOM'd under fat-LTO (their issue #90) — not worth
risking on a small self-hosted box.

## Usage

```
docker pull ghcr.io/<owner>/crw-camoufox:latest
```

Then point `[renderer.camoufox]` / `CRW_RENDERER__CAMOUFOX__*` at a running
`camofox-browser` sidecar as usual.

## Triggering a build

Manual: Actions tab → "build-camoufox-image" → Run workflow.
Also runs weekly (Monday 03:17 UTC) to pick up upstream changes.
