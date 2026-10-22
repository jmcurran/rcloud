# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

RCloud (version in `VERSION`) is a collaborative notebook platform. It has an R back end that runs inside Rserve and a jQuery/Ace browser client. This is a fork with a reproducible Zig-based build aimed at Debian 12. The main branch is `develop`.

## Build and run

```sh
nix develop                  # optional: dev shell with R, Zig 0.14.0, node, redis, system libs
zig build                    # full build into zig-out/ (~3 min): R packages + deps, htdocs, conf
cd zig-out && conf/start2    # run (conf/start2 -d for debug); conf/stop to stop
```

- Zig 0.14.0 is required. Without Nix, `zig/download.sh 0.14.0` (run from the repo root) installs it into `zig/`.
- `zig build update` re-scans the `DESCRIPTION` files under `packages/`, `rcloud.client/`, `rcloud.packages/` and `rcloud.support/`, resolves CRAN/R dependencies, and rewrites `build-aux/config.json` and `build-aux/generated/build.zig`. Don't hand-edit the generated build file. New entries in `config.json` start with an empty hash, which the next `zig build` fills in.
- `zig build dist` makes a thin source tarball. `dist-fat` (with `-Dassets=zig-out/assets` and `--global-cache-dir zig/cache`) makes a tarball that builds offline. See `README-dev.md`.
- `-Dassets=<dir>` bypasses downloading **and hash verification** of R dependency tarballs. Only use it for development.
- If the build behaves strangely, the cause is usually stale caching: delete `.zig-cache/`, and if needed `~/.cache/zig`. Deleting `zig-out/assets` alone does not force a re-download.
- Docker: `docker/dev/compose.yaml` (local dev) and `docker/stage/` (PAM auth, anonymous users, gitgist), built on `docker/common/Dockerfile`. Run `docker compose build && docker compose up` from the config's directory. Code changes need a rebuild of the image.

### Front end

`zig build` runs `npm ci` and Grunt itself. To iterate on the JS/CSS alone, run `npm ci` and then `npx grunt` at the repo root. The Grunt pipeline is sass → concat → uglify → compress.

- `htdocs/js/rcloud_bundle.js` is a **concatenation** of the files listed in `bundleFiles` in `Gruntfile.js`, in that order. A new JS file must be added to that list or it won't load. The `_*_begin.js` files set up the namespaces that later files fill in, so the order matters.
- The bundles (`rcloud_bundle*`, `merger_bundle*`) and `htdocs/css/*.css` are generated from `htdocs/js` and `htdocs/sass`, so edit the sources rather than the outputs. `htdocs/lib/` is vendored third-party code, including a committed `ace_bundle.js`.
- Lint with `npx eslint htdocs/js` (config in `.eslintrc.js`; the globals are `RCloud`, `rcloud`, `shell`, `Notebook` and others).

### Tests

The repo has no automated test suite (no testthat, no JS tests, no CI). To verify a change, build it and exercise it in a running instance (`zig-out` or Docker).

## Architecture

The big picture is in `doc/RCloud-architecture.pdf`. The most important points:

- **Rserve is the server.** `conf/run_rcloud.R` starts Rserve with `conf/rserve.conf`, which calls `rcloud.support:::configure.rcloud()`. Rserve serves `htdocs/` over HTTP, runs the `htdocs/*.R` scripts (`login.R`, `notebook.R`, `api.R`, `view.R`, …) as FastRWeb-style endpoints, and forks one R process per WebSocket connection. That process can also fork a separate "compute" process, and can switch to the logged-in Unix user.
- **The client talks to the server through object capabilities (OCAPs) over a WebSocket.** `rcloud.support/R/ocaps.R` builds the capability tree: `unauthenticated.ocaps()` → `authenticated.ocaps()` after login, and `compute.ocaps()` for the compute process. Each function is wrapped with `make.oc()`. On the JS side, `htdocs/js/rcloud.js` (`RCloud.create`) promisifies those paths into the `rcloud.*` API (`fooAsync` → promise). The server pushes asynchronous output and messages back as OOB messages (`.rc.oobSend`, `self.oobMessage`). To expose a new server function to the client, add it in `ocaps.R` **and** register its path in `rcloud.js`. Put it in the unauthenticated tree only if anonymous clients should have it.
- **`rcloud.support/`** is the core R package. It covers session setup (`setup.R`, `session*.R`), configuration from `rcloud.conf` (`conf.R`), auth (`auth.R`, `password.R`, `crypt.R`), notebook execution and output (`notebook.execution.R`, `output.R`, `plots.R`, `wplot.R`), and RCS. RCS is the metadata key/value store (stars, interests, settings), with flat-file and Redis back ends in `rcs.ff.R` and `rcs.redis.R`. `src/` has a few C helpers.
- **Notebook storage is pluggable** through the `gist` interface in `packages/gist`. The implementations are `githubgist` (GitHub/GHE) and `gitgist` (local git repositories). The back end is chosen in `rcloud.conf`.
- **`rcloud.packages/`** are RCloud-dependent R packages that ship by default. They are language/cell handlers (`rcloud.r`, `rcloud.python`, `rcloud.sh`, `rcloud.rmarkdown`, `rcloud.jupyter`) and UI extensions (`rcloud.viewer`, `rcloud.enviewer`, `rcloud.notebook.info`, `rcloud.web`, `rcloud.lux`). Each one registers itself through the language and extension mechanisms (`languages.R`, `htdocs/js/language.js`, `extension.js`) and usually carries JS in `inst/`.
- **`rcloud.client/`** is an R client for talking to RCloud through Rserve's QAP protocol (`RSclient`).
- **Front-end entry points**: `edit.html`/`edit.js` (the main IDE), `view.html` (read-only), `mini.html` (dashboards and "mini" notebooks), and `discover.html`. The notebook UI in `htdocs/js/notebook/` uses a model/view/controller split for notebooks, cells and assets.
- **Configuration**: `conf/rcloud.conf.samp` and `conf/rcloud.conf.dist` are templates, and `doc/CONFIGURATION.md` documents the keys. `services/` and `conf/start`/`start-qap` cover multi-process deployments (proxy, script server, SKS auth, Solr search).

## Notes

- `review/rcloud-review.qmd` (git-ignored) is a static code and security review at commit `9dc56614`. It lists known security issues and bugs, including an unauthenticated config read, missing ownership checks in `gitgist`, and XSS through assets. It also has notes on removing the Rserve dependency and pruning old APIs. Check it before changing auth, the OCAPs or storage.
- R code style in this repo is the original upstream style (`<-`, dotted names such as `rcloud.foo.bar`). Match the surrounding code rather than the workspace R-package conventions.

## NEWS.md and README.md

Keep both up to date as part of the work, not as an afterthought:

- **Every commit updates `NEWS.md`** in the same commit, saying what changed from a user's or deployer's point of view. Add the entry under the top `## RCloud <version>` section that matches `VERSION`, starting a new section if `VERSION` has moved past it. Use the existing `### Features` / `### Improvements` / `### Bugfixes` subsections and ` * ` bullets, with issue/PR numbers in parentheses where there are any. Purely internal changes (refactors, tests) still get a short entry, under `### Improvements` if nothing else fits.
- **Update `README.md`** whenever a change affects what it describes: installation, the Zig or Docker quick starts, required Zig version, or how to run and stop RCloud. Maintainer-only build detail goes in `README-dev.md`, and the release tarball's instructions in `README-release.md`.
