# AGENTS.md

## Cursor Cloud specific instructions

Colibrì is a local LLM inference engine written in pure C (`c/colibri.c`) that streams
Mixture-of-Experts weights from disk, plus Python launcher/API tooling (`c/coli`,
`c/openai_server.py`), a React/Vite web dashboard (`web/`), and a Tauri desktop shell
(`desktop/`). There is **no database or external service**; runtime state lives in
file sidecars next to the model.

### What the update script already does
On startup the update script runs `npm --prefix web ci` to refresh the web dashboard's
`node_modules`. Nothing else is auto-installed — the C engine and Python tooling are
dependency-free (stdlib only), so no build or pip step is needed to start working.

### Toolchain (preinstalled in this environment)
`gcc` (OpenMP/libgomp works), `make`, `python3` (3.12), `node`/`npm` (v22), `cargo`.

### Building and testing (standard commands — see `CONTRIBUTING.md`, `Makefile`, `web/package.json`)
- Engine + full dependency-free gate (clean + portable CPU build + C unit tests +
  Python stdlib tests): `make -C c check` (or `make check` from root). This is the
  canonical CI gate (`.github/workflows/check.yml`); it does **not** need a model or CUDA.
  Takes a couple of minutes; ~220 tests, some skipped.
- Engine only: `make -C c colibri` (or `make -C c colibri inkling`).
- Web dashboard: from `web/` run `npm run build` (tsc + vite) and `npm test` (vitest;
  the tests mock the backend, so they need no engine or model).

### Running the app in development
- Web dashboard dev server: `npm run dev` in `web/` serves on port **5173** and is
  bound to `localhost` only (Vite `host: false`), so reach it via `http://localhost:5173`
  — `http://127.0.0.1:5173` may refuse the connection. The dashboard is a pure
  OpenAI-API client that talks to a backend at `http://127.0.0.1:8000/v1` by default;
  use the sidebar "Probe server" button to connect.
- The real inference server (`./c/coli serve|web|chat`) requires the ~372 GB GLM-5.2
  int4 model on fast local NVMe (see `README.md`). **This model is not present in the
  cloud environment**, so `coli serve/web/chat` cannot run end-to-end here. The web
  dashboard, unit/integration tests, and the engine build are the practical targets.

### Real engine inference without the 372 GB model (optional)
The compiled engine can be exercised end-to-end against a tiny random-init model using
the CI-blessed inkling oracle. It needs CPU torch + transformers (heavy, so it is NOT in
the update script): `python3 -m pip install -r c/tools/oracle-requirements.txt`, then
`cd c && make inkling && python3 tools/make_tiny_inkling.py tiny_inkling &&
SNAP=tiny_inkling ./inkling 8 0 tiny_inkling/ref_inkling.json` — the C engine must
reproduce the transformers reference token-for-token (expect `24/24` matching tokens).

### Gotchas
- Contributions target the `dev` branch, not `main` (see `CONTRIBUTING.md`).
- The Makefile defaults to `ARCH=native`; `make portable` / `make check` build a portable
  x86-64-v3 baseline. Flag changes are tracked via `.build-config`, so a plain rebuild
  after changing `CUDA`/`ARCH`/etc. relinks correctly.
- Build artifacts (`c/colibri`, `c/inkling`, `c/tiny_inkling/`, `web/node_modules`,
  `web/dist`) are gitignored.
- The desktop shell (`desktop/`, Tauri v2) additionally needs system WebKitGTK/webview
  libraries and the `tauri-cli`; it is optional and not required for engine or web work.
