# Repository notes for automated agents

## What this app is

`Index.html` is the entire application: a Scratch project exported with the
TurboWarp packager. The project data (`project.json` plus assets) is embedded in
the page as compressed `data` attributes, so there is **no build step, no
backend, no database and no external service credentials**. Any hosting story is
"serve this one file".

Do not run a bundler, a framework dev server or `npm install` here; there is no
package manifest and nothing to compile.

## Running it

```sh
docker compose -f docker-compose.base44.yml up -d   # http://localhost:3000
```

nginx serves the repo file directly from a read-only bind mount, so an edit to
`Index.html` is live on reload (no rebuild).

## Non-obvious things that will bite you

1. **Only `Index.html` (and the shell files) are mounted, per file.** The
   sandbox checkout's top directory is not traversable by nginx's unprivileged
   worker user (`nginx`, uid 101), so mounting `./` yields `403 Forbidden`. If
   the app ever gains another static file, add a mount for it.

2. **The preview harness rewrites app HTML.** It injects a script after *every*
   literal `<head>` string in the HTML source, not just the real document head.
   This game's bundled JavaScript contains two literal `<head>` substrings (a
   DOMPurify XHTML wrapper and a hidden iframe's `document.write`), so the
   injection splits the bundle and the page dies with
   `SyntaxError: Invalid or unexpected token` followed by
   `ReferenceError: Scaffolding is not defined`. Reproduced at commit c906b0b
   by serving `Index.html` directly at `/`; absent through the shell below.

   When `BASE44_PREVIEW_MODE=1`, `/` therefore serves `.base44/preview.html`,
   which fetches `/game.raw` (`Index.html` as `application/octet-stream`) and
   boots it in a same-origin blob-URL iframe, keeping the game's source away
   from that rewriting. With the flag unset the app is served exactly as it
   ships: `Index.html` at `/`. The shell is preview plumbing only — it holds no
   game logic and can be dropped on any host without the harness.

## Verifying a change

Loading `/` should end with the loading screen hidden, no `#error-message`
visible, a `#app canvas` present, and `vm.runtime` reporting sprites
(`Bula`, `Sprite1`, …). The stage is WebGL, so copying the canvas into a 2D
context reads back blank — judge the rendering from a screenshot, not pixels.
