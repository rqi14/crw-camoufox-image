# crw-camoufox-image

Builds [`us/crw`](https://github.com/us/crw) (fastCRW) with the `camoufox`
Cargo feature enabled, which the official `ghcr.io/us/crw` image does not
ship (it's compiled with `--features cdp` only). This feature is required
for `crw-server` to talk to a [`camofox-browser`](https://github.com/redf0x1/camofox-browser)
sidecar (`[renderer.camoufox]` in `config.toml`), which is the only renderer
tier that can pass Cloudflare-style fingerprint/bot challenges.

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
