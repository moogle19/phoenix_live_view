# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

Phoenix LiveView: an Elixir library (server, `lib/`) plus its JavaScript client (`assets/js/phoenix_live_view/`, published to npm as `phoenix_live_view`). Most behavior changes touch both sides, so check the counterpart when editing either.

## Commands

Setup: `mix deps.get && npm ci` (the JS tests need `deps/phoenix` from mix).

Elixir:
- `mix test` runs all Elixir tests; `mix test test/phoenix_live_view/diff_test.exs:42` runs one test.
- CI also runs `mix compile --warnings-as-errors`, `mix compile --no-optional-deps`, `mix format --check-formatted`, and `mix test --warnings-as-errors`.

JavaScript (TypeScript sources, type checked with `tsc`, unit tested with Jest + jsdom):
- `npm run js:test` (runs `tsc` first, which emits `.d.ts` files into the gitignored `assets/js/types/`).
- Single file or test: `npm run build && npx jest assets/test/view_test.ts -t "test name"`.
- `npm run typecheck:tests`, `npm run js:lint`, `npm run js:format` / `npm run js:format.check` (Prettier on `assets` and `test/e2e`).

End-to-end (Playwright, Chromium/Firefox/WebKit against a real LiveView server):
- `npm run e2e:test` builds assets via `mix assets.build`, then Playwright starts the server with `npm run e2e:server` (`MIX_ENV=e2e`, port 4004).
- Single test: `npm run e2e:test -- tests/streams.spec.js:9 --project chromium` (add `--headed`, `--debug`, or `--ui`).

Assets: `mix assets.build` / `mix assets.watch` bundle the client with esbuild (config in `config/config.exs`) into `priv/static/`. Those bundles are committed, but CI regenerates and commits them on push to main ("Update assets" commits), so do not hand-edit them.

## Architecture

**Server side (`lib/phoenix_live_view/`)**
- `Phoenix.LiveView` / `Phoenix.Component` / `Phoenix.LiveComponent` are the public APIs. `Phoenix.LiveView.Static` handles the initial HTTP (dead) render; `Phoenix.LiveView.Channel` is the GenServer that runs a connected LiveView over a Phoenix channel, dispatching events, `handle_info`, navigation, uploads, and live components.
- HEEx compilation: `TagEngine` (generic, in `tag_engine/`) + `HTMLEngine` parse templates and produce a `%Phoenix.LiveView.Rendered{}` that separates static parts from dynamic expressions with change tracking via `Engine`. `Phoenix.Component.Declarative` (`lib/phoenix_component/`) backs the `attr`/`slot` declarations and their compile-time validation. `html_formatter/` is the `mix format` plugin for HEEx.
- `Phoenix.LiveView.Diff` turns `Rendered` structs into the wire diff format and keeps the per-process fingerprint/component state. Wire keys are short atoms (`:s` statics, `:c` components, `:k` keyed comprehensions, `:p` shared templates, `:stream`); the JS client must agree with them.
- Colocated hooks/JS/CSS (`colocated_*.ex`, `macro_component.ex`) extract code from templates at compile time.
- `lib/phoenix_live_view/test/` is `Phoenix.LiveViewTest`: it runs LiveViews in-process with a `ClientProxy` that applies diffs to a DOM (via `lazy_html`) without a browser, so it mirrors client semantics in Elixir.
- `lib/phoenix_live_view/igniter/` and `lib/mix/` contain installer/upgrade tasks (optional `igniter` dep).

**Client side (`assets/js/phoenix_live_view/`)**
- `LiveSocket` (`live_socket.ts`) owns the Phoenix socket, the root views, navigation (live_patch / live_redirect), and global event binding.
- `View` (`view.ts`) joins one LiveView channel, pushes events, handles replies and diffs; `Rendered` (`rendered.js`) merges incoming diffs into the cached tree and renders it to HTML; `DOMPatch` (`dom_patch.ts`) applies that HTML with a vendored-fork `morphdom`, handling `phx-update`, streams, focus preservation, and portals.
- `ElementRef` (`element_ref.ts`) manages loading/lock refs on elements with pending events (`phx-*-loading` classes, disabled/readonly state) and undoes them when the server acknowledges. `dom_post_morph_restorer.ts` restores child order for `phx-update="append"/"prepend"` containers after morphing.
- `js.js` / `js_commands.ts` execute `Phoenix.LiveView.JS` commands encoded server-side by `lib/phoenix_live_view/js.ex`; `hooks.ts` / `view_hook.ts` implement client hooks; `live_uploader.js`, `upload_entry.js`, `entry_uploader.js` implement uploads (server counterpart: `upload*.ex`, `upload_channel.ex`).

**Tests**
- `test/support/` defines the endpoint, router, and many fixture LiveViews used by `test/phoenix_live_view/integrations/`.
- `assets/test/` holds Jest tests (`integration/` exercises `LiveSocket` + `View` together).
- E2E: LiveViews in `test/e2e/support/` are routed in `test/e2e/test_helper.exs`; specs live in `test/e2e/tests/`. Regression tests for GitHub issues follow the pattern `test/e2e/support/issues/issue_NNNN.ex` + `test/e2e/tests/issues/NNNN.spec.js`, routed under `/issues`.

## Conventions

- Add user-facing changes to `CHANGELOG.md`.
- Guides in `guides/` are part of the HexDocs; update them alongside behavior changes. JS API docs come from `assets/js/phoenix_live_view/README.md` and TypeDoc comments.
