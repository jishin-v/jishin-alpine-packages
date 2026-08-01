# AGENTS.md

Notes for coding agents working in this repo.

## Source of truth, mirror, and CI logs

- The **canonical** repo is the self-hosted Gitea remote `origin`
  (`gitea@git.emunest.net:jishin/alpine_packages.git`). Push branches here —
  do **not** push to GitHub directly.
- Gitea **mirrors** to GitHub at
  [`jishin-v/jishin-alpine-packages`](https://github.com/jishin-v/jishin-alpine-packages).
  Pushing to `origin` triggers the mirror, and the GitHub Actions CI
  (`.github/workflows/ci.yml`) runs on the mirrored repo.
- **Read CI logs with the `gh` CLI** against the GitHub mirror, e.g.:

  ```sh
  gh run list -R jishin-v/jishin-alpine-packages --limit 5
  gh run view -R jishin-v/jishin-alpine-packages <run-id> --log-failed
  gh run watch -R jishin-v/jishin-alpine-packages <run-id>
  ```

  There is a short delay after `git push origin` while Gitea mirrors to GitHub
  and the workflow kicks off; if `gh run list` shows nothing yet, wait ~30–60 s
  and retry. Grep full logs for `>>> ERROR:` / `Create …apk` — during
  iteration with a `/keep-going` trailer the job can report success while
  individual packages fail.

## Working branch & commits

- Day-to-day work happens on **`staging`**. (Other long-lived branches exist
  for older Alpine releases: `v3.<n>`, plus `main`/`master`/`old`/`template`.)
- New aports go under `incubating/<pkgname>/APKBUILD` (one subdir per aport).
- Commit messages follow `<repo>/<pkgname>: <summary>`, e.g.
  `incubating/dirge: Add aport v0.18.7 (Rust coding agent)`. The
  `prepare-commit-msg` hook auto-prepends this prefix when you commit without
  `-m`; supply it explicitly when using `-m`.
- The `/keep-going` commit trailer makes `buildrepo` attempt every package in a
  run even if one fails (and flips the job conclusion to `success`), useful
  while iterating.

## APKBUILD conventions in this repo

- **DIY repo** — standards can be relaxed: vendoring blobs in-repo is fine, no
  need for perfect subpackage splits, etc.
- `incubating/` is the active aport dir. Read sibling APKBUILDs (dirge, kumo,
  charon, aevum) before writing a new one — they encode the repo's idioms
  (detailed header comment, `so:`-autoscan + only the *functional* `depends=`,
  `$builddir`-absolute `prepare()` patches, etc.).
- `.editorconfig`: APKBUILDs use **tab** indentation, 4-col otherwise.
- The `pre-commit` hook validates local source files + checksums and rejects
  files >256 KiB. Remote (`http(s)://…`) sources are skipped, so point
  `source=` at fetchable URLs rather than committing large tarballs (the rare
  exception is the `../alpine_packages_bulk` LFS mirror scheme used by
  `wpewebkit-kumo` / `kumo` — see PLAN.md).
- `-dbg`/`-doc` subpackages and signature verification of sources are generally
  deferred unless requested.
- `PLAN.md` records detailed notes (gotchas, decisions, CI iteration history)
  for the heavier aports — consult it, and add to it for non-trivial work.

## CI specifics

- 2-way arch matrix: `x86_64 -> ubuntu-latest`, `aarch64 -> ubuntu-24.04-arm`,
  each pulling the matching `alpine:3.24` image so `$CARCH` matches the host
  (native builds). `fail-fast: false`.
- `actions/checkout` is **not** used (it doesn't run inside Alpine/musl
  containers on aarch64); the workflow does a plain `git clone` instead.
- abuild resolves build-deps from the local `$REPODEST`, so a aport can depend
  on another in this same repo (e.g. `kumo` → `wpewebkit-kumo-dev`) within one
  arch run.
- `buildrepo` skips a package whose apk is already published in `REPODEST`;
  bump `pkgrel` to force a rebuild.

## Local environment caveat

The dev host may not be Alpine (it can be NixOS), so `apk`/`abuild` may be
unavailable locally. Verify dependency package names against
`https://pkgs.alpinelinux.org/` (package pages return 404 for missing pkgs;
use the `/contents?file=…` endpoint to find which apk ships a given file or
pkg-config `.pc`). Actual builds run in CI.
