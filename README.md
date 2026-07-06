# weeks

Your whole life as a scrollable grid of weeks — one dot per week. Enter your birthday
and see how many you've lived, and how many might be left.

Live at **[weeks.danbing.app](https://weeks.danbing.app)** · a [danbing.app](https://danbing.app) toy.

## Run locally

Pure static site, no build step:

```bash
python3 -m http.server 8092 --directory src
# open http://localhost:8092
```

## How it works

Everything is client-side. Your birthday and lifespan are encoded in the URL hash, so a
link fully reconstructs the view — **nothing is ever sent to a server.** Deployed as
static files from `src/` via Cloudflare Pages.

Inspired by Tim Urban's "Your Life in Weeks."
