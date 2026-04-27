# kongctl-ext-debug

A small, script-based [`kongctl`](https://github.com/Kong/kongctl) extension
that prints diagnostic information about how the extension was invoked, dumps
the runtime context that `kongctl` passes to it, and demonstrates a reentrant
call back into the host CLI (`kongctl get me`).

It's a working example you can install, read, and copy when starting your own
extension.

## Install

```sh
# Latest release
kongctl install extension Kong/kongctl-ext-debug

# A specific tag
kongctl install extension Kong/kongctl-ext-debug@v0.1.0
```

## Use

Both command paths invoke the same script:

```sh
kongctl get debug
kongctl debug

# Pass arbitrary args to see them echoed back
kongctl debug -- --foo bar baz
```

The script prints the path to the runtime context file, the args it received,
the full `context.json` contents, and the result of a reentrant
`kongctl get me` call. If you aren't authenticated, the reentrant call fails
and a hint is written to stderr — the rest of the output still appears on
stdout.

## Build your own extension

The fastest path is the **`kongctl-extension-builder` skill**, usable from any
coding agent that supports skills. It scaffolds a valid `extension.yaml`,
runtime, and release workflow from a short prompt:

```
/kongctl-extension-builder I want to build a kongctl extension that ...
```

The skill encodes the manifest rules, runtime contract, and release-archive
shape that `kongctl` expects, so you don't have to keep them in your head.

If you'd rather read by example, this repo is intentionally tiny — clone it,
rename the publisher/name in `extension.yaml`, swap in your own runtime, and
adjust the release workflow.

## Repo layout

```
extension.yaml                 # manifest
bin/kongctl-ext-debug          # runtime (the file referenced by runtime.command)
.github/workflows/release.yml  # workflow_dispatch release pipeline
```

Release archives extract to the same shape, with `extension.yaml` at the root.

## Releasing

The [`release` workflow](.github/workflows/release.yml) is triggered manually
via **Actions → release → Run workflow**. Provide a tag like `v0.1.0`; the
workflow builds `kongctl-ext-debug-universal.tar.gz` and creates a GitHub
Release at the chosen commit with the archive attached.
