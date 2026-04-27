# kongctl-ext-debug

A small, script-based [`kongctl`](https://github.com/Kong/kongctl) extension
that prints diagnostic information about how the extension was invoked, dumps
the runtime context that `kongctl` passes to it, and demonstrates a reentrant
call back into the host CLI (`kongctl get me`).

It exists for two reasons:

1. **Exercise the repository-based install path.** `kongctl install extension
   Kong/kongctl-ext-debug` should fetch a release archive, validate the
   manifest, and link the runtime — this repository is the simplest possible
   target to verify that flow end to end.
2. **Be a working example.** The repo layout, manifest, script, and release
   workflow are intentionally minimal and copyable. Use it as a starting
   template for your own extension.

> **Status — extension support in `kongctl`.** Extension capabilities
> (`kongctl install extension`, `kongctl link extension`, the
> `KONGCTL_EXTENSION_CONTEXT` runtime contract, etc.) are **not yet merged into
> `main`** of [`Kong/kongctl`](https://github.com/Kong/kongctl). To try this
> extension you currently need a `kongctl` build from the branch where that
> work lives. Once the feature lands, the install commands below will work
> against a stock release.

## What it prints

When you run `kongctl get debug` (or `kongctl debug`), the script writes:

- `context_path` — the path to the JSON context file `kongctl` generated for
  this invocation
- `args` — any remaining arguments passed after the matched command path
- `context_json` — the full contents of that context file (matched command
  path, profile, base URL, output settings, host `kongctl_path`, etc.)
- `reentrant_kongctl_get_me` — the result of calling back into the host
  `kongctl` to fetch the authenticated user, using the `kongctl_path` recorded
  in the context

If you are not authenticated, the reentrant call will fail and the script
prints a hint to stderr — stdout still contains the rest of the diagnostic
output.

## Install

Once `kongctl`'s extension support is available in your build:

```sh
# Latest published release
kongctl install extension Kong/kongctl-ext-debug

# A specific tag
kongctl install extension Kong/kongctl-ext-debug@v0.1.0

# A branch (source clone fallback; release archive is preferred when present)
kongctl install extension Kong/kongctl-ext-debug --ref main

kongctl list extensions
kongctl get extension kong/debug
```

To uninstall:

```sh
kongctl uninstall extension kong/debug
```

## Use

Both command paths invoke the same script:

```sh
kongctl get debug
kongctl debug

# Pass arbitrary args to see them echoed back as `args=...`
kongctl debug -- --foo bar baz
```

`--` is useful when you want to pass a token that `kongctl` would otherwise try
to interpret as a host flag.

## Run it locally without releasing

You can iterate on the script without cutting a release by linking the working
tree directly:

```sh
git clone https://github.com/Kong/kongctl-ext-debug
cd kongctl-ext-debug
chmod +x bin/kongctl-ext-debug

kongctl link extension .
kongctl get extension kong/debug
kongctl debug
```

`link` points at the on-disk directory, so edits to `bin/kongctl-ext-debug`
take effect on the next invocation. To exercise the full install path
(`kongctl` copies the package into its managed location), use:

```sh
kongctl install extension .
kongctl uninstall extension kong/debug
```

## Repo layout

```
extension.yaml                 # the manifest kongctl reads
bin/kongctl-ext-debug          # runtime — the file referenced by runtime.command
README.md                      # this file (also bundled into release archives)
.github/workflows/release.yml  # workflow_dispatch release pipeline
```

The release archive extracts to the same shape, with `extension.yaml` at the
archive root and the executable at `bin/kongctl-ext-debug`.

## Releases

Release archives are built and published by the
[`release` workflow](.github/workflows/release.yml). It is triggered manually
via **Actions → release → Run workflow**:

1. Pick the branch/commit to release from.
2. Provide a tag, e.g. `v0.1.0` (must match `vMAJOR.MINOR.PATCH` with an
   optional pre-release suffix).
3. The workflow builds `dist/kongctl-ext-debug-universal.tar.gz` and creates a
   GitHub Release at the chosen commit with that tag and the archive attached.

Because this is a shell script extension with no compiled component, a single
universal `.tar.gz` is published per release. `kongctl install extension`
accepts `.tar.gz`, `.tgz`, and `.zip` assets.

To reproduce the archive locally:

```sh
mkdir -p dist/package/bin
cp extension.yaml README.md dist/package/
cp bin/kongctl-ext-debug dist/package/bin/
chmod +x dist/package/bin/kongctl-ext-debug
tar -C dist/package -czf dist/kongctl-ext-debug-universal.tar.gz \
  extension.yaml README.md bin/kongctl-ext-debug
```

## Using this repo as a template for your own extension

The pieces you need for a minimal `kongctl` extension are all here. To fork
this into your own:

1. **Rename the manifest.** Edit `extension.yaml`:
   - `publisher` and `name` form the extension ID (`publisher/name`). Use
     lowercase, path-safe segments.
   - `runtime.command` is a path *relative to the extension root*, and the
     file at that path must already be executable. `kongctl` does not compile
     anything during install.
   - `command_paths` is a list. Each entry contributes a path into the
     `kongctl` command tree. v1 extensions can either contribute below the
     built-in `get` / `list` verbs, or define a custom root verb that does not
     collide with a built-in. Built-in roots like `get` and `list` cannot
     declare aliases; leaf names can.

2. **Replace the runtime.** Rename `bin/kongctl-ext-debug` to whatever you set
   in `runtime.command`. For richer parsing or JSON handling you may prefer a
   compiled Go binary instead of a shell script — in that case add a build
   step to the release workflow and a build matrix per platform (see notes
   below).

3. **Rely on the runtime context.** When `kongctl` invokes your extension it
   sets the `KONGCTL_EXTENSION_CONTEXT` env var to a generated `context.json`
   file. Read that file for:
   - the matched command path
   - remaining extension arguments
   - the selected profile and resolved base URL
   - output and log settings
   - the extension data directory
   - the host `kongctl` path and version

   Do **not** expect secrets in `context.json`. If your extension needs
   authenticated Konnect access, shell out to the host `kongctl` (`kongctl
   api`, `kongctl get …`, etc.) using the `kongctl_path` from the context.
   Child `kongctl` invocations inherit the parent extension's profile, output,
   and base URL unless you explicitly override them.

4. **Be careful with stdout.** If your extension is meant to be piped or
   parsed, write diagnostics to stderr so stdout stays clean for whatever
   structured output you produce.

5. **Avoid importing `kongctl/internal/...`.** Extensions run as separate
   processes. They are not Go plugins and should not depend on `kongctl`'s
   internal packages — talk to the host CLI as a subprocess instead.

6. **Ship a release archive.** `kongctl install extension <owner>/<repo>`
   prefers a release archive whose root contains `extension.yaml` and the
   runtime referenced by `runtime.command`. Publish a single archive per
   release for script extensions, or platform-specific archives whose names
   include `GOOS-GOARCH` for compiled extensions (e.g.
   `myext-linux-amd64.tar.gz`, `myext-darwin-arm64.tar.gz`). Use `.tar.gz`
   for compiled binaries — it preserves the executable bit reliably across
   platforms; `.zip` does not.

   When no compatible archive is found, `kongctl install extension` falls
   back to cloning the repo. That fallback only works if the repo root
   already contains `extension.yaml` and an already-executable runtime.

### If you switch to a Go runtime

Replace the script with a Go `main` package and add a build matrix to the
release workflow. A starting point:

```yaml
strategy:
  matrix:
    include:
      - { goos: linux,  goarch: amd64 }
      - { goos: linux,  goarch: arm64 }
      - { goos: darwin, goarch: amd64 }
      - { goos: darwin, goarch: arm64 }
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-go@v5
    with:
      go-version-file: go.mod
  - run: |
      mkdir -p dist/package/bin
      CGO_ENABLED=0 GOOS=${{ matrix.goos }} GOARCH=${{ matrix.goarch }} \
        go build -o dist/package/bin/<your-binary> ./cmd/<your-binary>
      cp extension.yaml README.md dist/package/
      chmod +x dist/package/bin/<your-binary>
      tar -C dist/package -czf \
        dist/<your-ext>-${{ matrix.goos }}-${{ matrix.goarch }}.tar.gz \
        extension.yaml README.md bin/<your-binary>
```

Set `runtime.command: bin/<your-binary>` in the manifest.

## Related

- [`Kong/kongctl`](https://github.com/Kong/kongctl) — the host CLI. (Note:
  extension support is not yet on `main`; see **Status** above.)
