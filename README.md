# kitten3d Bazel registry

A small supplementary [Bazel registry](https://bazel.build/external/registry)
holding kitten3d's shared third-party module pins. It hosts **metadata only** —
the module archives themselves live as release assets in the repos that produce
them (e.g. the `dawn` module's tarball is a release asset in
[kitten3d/dawn-builds](https://github.com/kitten3d/dawn-builds)).

Registries compose, so this is not a fork of the [BCR](https://bcr.bazel.build)
— it is checked *first*, with the BCR as fallback.

## Consuming it

Add both `--registry` lines (order matters; specifying `--registry` replaces
the default list, so the BCR line is mandatory) to your `.bazelrc`:

```
common --registry=https://raw.githubusercontent.com/kitten3d/bazel-registry/main
common --registry=https://bcr.bazel.build
```

Then depend on a module normally:

```starlark
bazel_dep(name = "dawn", version = "20260214.164635")
```

## Modules

- **`dawn`** — platform-neutral wrapper over prebuilt Dawn (WebGPU) static
  libraries; Wayland+X11-enabled Linux binary from kitten3d/dawn-builds,
  upstream macOS/Windows binaries. See
  [modules/dawn/metadata.json](modules/dawn/metadata.json).

## Layout

```
bazel_registry.json                 # registry config (mirror slot)
modules/
└── dawn/
    ├── metadata.json               # homepage, maintainers, versions
    └── <version>/
        ├── MODULE.bazel            # byte-identical copy of the module's MODULE.bazel
        └── source.json             # archive url + SRI integrity + strip_prefix
```

**Byte-identical MODULE.bazel rule:** each `modules/<name>/<version>/MODULE.bazel`
must be a byte-for-byte copy of the `MODULE.bazel` inside that version's source
archive. Bazel verifies this and fails the build on any mismatch — treat the
error as "re-copy the file", never as "hand-edit to match".
