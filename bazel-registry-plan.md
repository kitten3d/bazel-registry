# kitten3d Bazel registry — design & bootstrap plan

> **Status:** RATIFIED direction, 2026-07-14 — Michael chose the registry
> ("full send on option 1") over release-URL-only and build-from-source
> alternatives. This document is the **baseline context for bootstrapping the
> registry in a new session/repo**: it is deliberately self-contained and
> repeats workspace facts a fresh session will not have. Origin: finding F1 of
> [plans/wayland-runtime-review-findings.md](wayland-runtime-review-findings.md).

---

## 1. Why this exists (context a new session needs)

### The workspace

`kitten-workspace` (github.com/mfield4/kitten3d-workspace) is a hybrid
monorepo: a Bazel 9.0.1 bzlmod root module stitching six C++23 engine
submodules (kernel, platform, gpu, core, std, debug — GitHub org **kitten3d**;
note repo names on GitHub don't always match module names, e.g. gpu's remote
is `kitten3d/sandbox`) together via `local_path_override`. Each submodule is
also a **standalone Bazel module** — it must build and test from its own repo
root with no workspace present, so each carries its own `MODULE.bazel`,
`.bazelrc`, and the full transitive sibling-override set. Both gpu and debug
have CI that builds standalone (`--config=ci` pins gcc-14 on Linux).

### How Dawn is consumed today

Dawn (Google's WebGPU implementation) is consumed as **prebuilt static
tarballs** from Google's official nightly GitHub releases — never built from
source. Two modules consume it, each with an identical private copy of the
machinery:

- **gpu** (the renderer): `third_party/dawn_prebuilt.bzl` (a `repository_rule`
  used via `use_repo_rule`) + `third_party/dawn.BUILD` (overlay) + a
  `dawn_prebuilt(...)` block in `MODULE.bazel`. Targets reference `@dawn`
  (= `@dawn//:dawn`).
- **debug** (ImGui overlay shim): the same three files, byte-identical `.bzl`,
  because its vendored ImGui's `imgui_impl_wgpu.cpp` compiles against
  `<webgpu/webgpu.h>` — `third_party/imgui.BUILD` deps on `@dawn`.

Both static libs link into one game binary — a **latent ODR oddity** held
safe only by keeping the two declarations identical (workspace review finding
F4: an invariant maintained by discipline, no drift pin).

The repo rule's mechanics (preserve these exactly in the wrapper module):

- Platform keys from `ctx.os`: `macos_arm64`, `macos_x86_64`, `linux`,
  `windows`.
- Suffix map: `macos-latest-Release`, `macos-15-intel-Release`,
  `ubuntu-latest-Release`, `windows-latest-Release`.
- URL: `https://github.com/google/dawn/releases/download/<tag>/Dawn-<commit>-<suffix>.tar.gz`
- `stripPrefix = "Dawn-<commit>-<suffix>"`. Tarball layout after strip:
  `include/` (webgpu/, dawn/) + the static lib at **`lib64/libwebgpu_dawn.a`
  on Linux and Windows, `lib/` on macOS** — `dawn.BUILD`'s `cc_import`
  `select()`s on that, so packaging must preserve `lib64/`.
- Current pin: tag **`v20260214.164635`**, commit
  **`1a3afc99a7ef7dacaab73b71d44575c4f1bf2dd7`**. Upstream sha256s:
  - macos_arm64 `536c7ae9e2e679224797880afe6a3a6ba072e6986d5bc9b7cce18c2d730aa578`
  - macos_x86_64 `50439db37abd602ad7f46342b3200d11eaa955e6482c43d7daad72735cfd608a`
  - linux (upstream, X11-only) `9bee0a25a445f8a649cbebde305bdb4c93a1c9b6e4f89f6bf32d7f054a87eeaf`
  - windows `3abbab979ea196c0cc9e171be30a8c14850257ab77fd8e38a1a9473727bf5319`
- `dawn.BUILD` also carries the link surface: macOS frameworks (Metal,
  QuartzCore, IOKit, IOSurface, Foundation, CoreGraphics, CoreFoundation,
  Cocoa), Linux `-ldl -lpthread`.

### The problem that forced this

The upstream Linux release is built `DAWN_USE_X11=ON, DAWN_USE_WAYLAND=OFF`
(verified in Dawn's CMake and in the shipped binary: it has
`vkCreateXlibSurfaceKHR`, no `vkCreateWaylandSurfaceKHR`). A native
`wl_surface` fails Dawn validation with `Unsupported sType
(SType::SurfaceSourceWaylandSurface)`. The engine gained runtime
Wayland/X11 dispatch (the `wayland-runtime` worktrees on platform/gpu/debug —
see [logs/2026-07-13.md](../logs/2026-07-13.md)), which required a
Wayland-enabled Dawn. Interim state in those worktrees: a locally-built
tarball at `file:///home/michael/…/.dawn-build/dawn-wayland-linux.tar.gz`
(sha `91d6561ca4d572769cbd5eacb7362406f5ae5567c64242b1e2df04dca689071e`)
wired through a new `url_overrides` attr on `dawn_prebuilt`. **Review finding
F1: that machine-local path must not merge** — it breaks Linux builds on any
other machine or checkout path. The local tarball has a second, subtler flaw:
it was built with GCC 15.3 on openSUSE Tumbleweed, so it references newer
glibc/libstdc++ symbols than the gcc-14/ubuntu baseline gpu's CI pins —
CI-building on `ubuntu-latest` restores upstream's compatibility floor.

### The decision (alternatives weighed 2026-07-14)

1. **Own Bazel registry** ← **chosen**. Not a *fork* of the BCR — registries
   compose (`--registry` flags, checked in order), so this is a small
   supplementary registry holding only kitten3d modules.
2. Release-asset repo only (CI builds Wayland Dawn, `url_overrides` points at
   the https URL). Rejected as the endpoint but **subsumed**: a registry hosts
   metadata, not bytes, so the CI-built release asset is a *component* of
   option 1.
3. Build Dawn from source under Bazel. Rejected — historically caused
   constant issues, and the pain is structural: Dawn ships no usable Bazel
   build (its root `BUILD.bazel` is a header-glob shim only, no
   `MODULE.bazel`), and its deps are a 57-entry Chromium-style `DEPS` fetch.
   Also rows against the ratified house pattern (module-local third-party =
   sha-pinned prebuilt archives).

**What the registry uniquely buys** (the reason for full-send): gpu and debug
both declare `bazel_dep(name = "dawn", …)` and bzlmod resolves them to **one
shared module instance**. The duplicated machinery (2× `.bzl`, 2× overlay,
2× sha table), the must-match-sha invariant (F4), and the dual-static-link
ODR oddity all **cease to exist structurally**. It also becomes the home for
future shared third-party pins.

---

## 2. Target architecture — three pieces

### Piece A: `dawn-builds` repo (CI artifact factory)

A kitten3d repo whose GitHub Actions workflow produces the Wayland-enabled
Linux Dawn and publishes release assets at stable URLs.

- **Trigger:** `workflow_dispatch` with inputs `dawn_tag` (e.g.
  `v20260214.164635`) and `dawn_commit`; publish under a release tagged to
  match upstream (one release per Dawn version).
- **Runner:** `ubuntu-latest` (matches upstream's build environment and gpu
  CI's gcc-14 glibc/libstdc++ baseline — this is load-bearing, see above).
- **Build recipe** (proven locally 2026-07-13, GCC 15.3, 1051/1051 targets
  clean; keep verbatim, it is ABI-matched to the shipped `webgpu.h` by
  building at the exact upstream release commit):

  ```bash
  # apt: cmake ninja-build libwayland-dev wayland-protocols libxkbcommon-dev \
  #      libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev
  #      (expect to iterate; DAWN_FETCH_DEPENDENCIES covers the rest)
  git clone https://dawn.googlesource.com/dawn && cd dawn
  git checkout 1a3afc99a7ef7dacaab73b71d44575c4f1bf2dd7
  cmake -S . -B out/wayland -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DDAWN_FETCH_DEPENDENCIES=ON -DDAWN_ENABLE_INSTALL=ON \
    -DDAWN_BUILD_MONOLITHIC_LIBRARY=STATIC \
    -DDAWN_USE_WAYLAND=ON -DDAWN_USE_X11=ON -DDAWN_ENABLE_VULKAN=ON \
    -DDAWN_BUILD_SAMPLES=OFF -DDAWN_BUILD_TESTS=OFF \
    -DTINT_BUILD_TESTS=OFF -DTINT_BUILD_CMD_TOOLS=OFF \
    -DCMAKE_INSTALL_PREFIX=install
  cmake --build out/wayland --target webgpu_dawn -j
  cmake --install out/wayland
  ```

- **Packaging:** stage `install/` into
  `Dawn-<commit>-ubuntu-latest-Release/{include/,lib64/libwebgpu_dawn.a}` —
  **identical prefix and layout to upstream** (the consumer's `stripPrefix`
  and `dawn.BUILD` select depend on `lib64/`), then tar.gz.
- **CI acceptance gates** (encode the review checks):
  `nm -g` shows **both** `vkCreateWaylandSurfaceKHR` and
  `vkCreateXlibSurfaceKHR`; lib size sanity (~37 MB); layout matches upstream
  (`tar tf` prefix check). Emit the sha256 into the release notes.
- **Assets per release:** the Linux tarball **and the wrapper-module source
  tarball** (Piece B's archive — same repo, one release, two assets).
  **Always use uploaded release assets, never GitHub's autogenerated
  tag archives** — those are not byte-stable and will eventually break the
  integrity pin.

### Piece B: the `dawn` wrapper module (what the registry serves)

Dawn's per-platform prebuilts can't be a registry module directly — a
registry version has exactly **one** source archive, and we need four
platform tarballs (three still upstream, one ours). So the module is a thin,
platform-neutral wrapper: today's `dawn_prebuilt.bzl` + `dawn.BUILD` promoted
into a standalone module that fetches the right binary at repo-fetch time.

Shape (source lives in `dawn-builds`, e.g. under `module/`):

```
module/
├── MODULE.bazel        # module(name = "dawn", version = "20260214.164635")
│                       # bazel_dep: rules_cc, platforms
│                       # ext = use_extension("//:extensions.bzl", "dawn_binaries")
│                       # use_repo(ext, "dawn_bin")
├── extensions.bzl      # module extension wrapping today's dawn_prebuilt logic
│                       # verbatim: ctx.os → platform key → suffix/sha/url;
│                       # linux URL = the dawn-builds release asset (https);
│                       # mac/win URLs = upstream google/dawn releases
├── dawn_bin.BUILD      # today's dawn.BUILD, applied to the @dawn_bin repo
│                       # (cc_import lib64|lib select + cc_library w/ linkopts)
└── BUILD.bazel         # alias(name = "dawn", actual = "@dawn_bin//:dawn",
                        #       visibility = public)
```

Notes:
- The alias preserves the consumer label **`@dawn`** (= `@dawn//:dawn`)
  exactly — gpu's `BUILD.bazel` (2 uses) and debug's `imgui.BUILD` (1 use)
  need **zero label changes**. debug's `imgui.BUILD` keeps resolving `@dawn`
  because the http_archive it overlays is declared in debug's MODULE.bazel,
  so it sees debug's repo mapping, and `bazel_dep(name = "dawn")` provides
  that mapping just as `use_repo_rule(name = "dawn")` did.
- All four sha256s and both URL bases move into `extensions.bzl` — the
  **single source of truth** the old design duplicated across two repos.
  `url_overrides` dies with the old rule; it was the interim mechanism.
- **Versioning:** module version = upstream tag minus the `v`
  (`20260214.164635` — valid bzlmod). Wrapper-only fixes at the same Dawn
  version append a segment BCR-style (cf. `glfw 3.4.0.bcr.1`):
  `20260214.164635.1`.

### Piece C: the registry repo (metadata only)

A kitten3d repo (name to pick, e.g. `registry`), served as flat files via
`https://raw.githubusercontent.com/kitten3d/<repo>/main`:

```
bazel_registry.json                     # {} is valid; add mirrors later if wanted
modules/
└── dawn/
    ├── metadata.json                   # homepage, maintainers,
    │                                   # "versions": ["20260214.164635"],
    │                                   # "yanked_versions": {}
    └── 20260214.164635/
        ├── MODULE.bazel                # byte-identical copy of the wrapper's
        │                               # MODULE.bazel — Bazel VERIFIES they match
        └── source.json                 # {"url": "<dawn-builds release asset:
                                        #    wrapper-module tarball>",
                                        #  "integrity": "sha256-<BASE64>",
                                        #  "strip_prefix": "<if any>"}
```

Gotchas a fresh session must know:
- `integrity` is **SRI format** — base64 of the raw digest, *not* the hex
  from `sha256sum`: `openssl dgst -sha256 -binary f | base64`.
- The registry copy of `MODULE.bazel` must stay byte-identical to the one in
  the archive; Bazel checks and fails on mismatch.
- Keep both repos **public** — private registries/assets drag in `~/.netrc`
  auth for every machine and CI job.

### Consumer wiring

In **gpu** and **debug** `MODULE.bazel`: delete the whole
`use_repo_rule`/`dawn_prebuilt(...)` block; delete
`third_party/dawn_prebuilt.bzl` and `third_party/dawn.BUILD`; add

```starlark
bazel_dep(name = "dawn", version = "20260214.164635")
```

In **three** `.bazelrc`s — workspace root, gpu, debug (standalone builds
resolve registries per invocation; platform/kernel/etc. don't consume dawn
and need nothing):

```
# Custom registry first, BCR fallback. Specifying --registry REPLACES the
# default list, so the BCR line is mandatory.
common --registry=https://raw.githubusercontent.com/kitten3d/<registry-repo>/main
common --registry=https://bcr.bazel.build
```

Lockfiles (`MODULE.bazel.lock`, committed in workspace and submodules) record
registry URLs and hashes — regenerate and commit them wherever touched.

---

## 3. Migration plan (phased, each phase independently verifiable)

**Phase 0 — artifact factory.** Create `dawn-builds`; port the recipe into a
workflow; run it at commit `1a3afc9…`; publish release `v20260214.164635`
with the Linux tarball. Verify: download, `sha256sum`, `nm` gates, `tar tf`
prefix == `Dawn-1a3afc99a7ef7dacaab73b71d44575c4f1bf2dd7-ubuntu-latest-Release`.
(Compare against the known-good local tarball while it still exists.)

**Phase 1 — wrapper module.** Author `module/` in `dawn-builds` (lift
`dawn_prebuilt.bzl` + `dawn.BUILD` from gpu — they're byte-identical to
debug's). Point the linux URL at the Phase-0 asset; keep mac/win at upstream
with their existing shas. Package the wrapper tarball, attach it to the same
release. Smoke it *before* the registry exists via a bare test consumer using
`archive_override` (root-module overrides skip registry lookup).

**Phase 2 — registry.** Create the registry repo with the `dawn` entry
(SRI-converted integrity). Verify from a scratch module with only the
`--registry` flags: `bazel build @dawn` fetches wrapper → binary → links.

**Phase 3 — consumers.** The `.bazelrc` + `MODULE.bazel` edits above in
workspace, gpu, debug; delete the four dead third_party files; regenerate
lockfiles. Gates: workspace `bazel test @platform//:all @gpu//:all
@core//:all --test_tag_filters=-integration` green and `//games/voxel_sandbox
--define=debug=on` builds; **standalone** gpu and debug build/test green;
`bazel mod graph | grep dawn` shows **one** dawn instance (the structural F4
fix — record it); voxel_sandbox runs natively on Wayland with no
`Unsupported sType`.

**Phase 4 — converge the wayland-runtime worktrees.** Those worktrees carry
two kinds of changes: engine runtime-dispatch code (platform
window/handle/platform.hpp, gpu surface.cpp, debug imgui_overlay.cpp) — **keep,
merge as planned** — and the interim Dawn plumbing (both MODULE.bazel
url_overrides blocks + sha edits, both dawn_prebuilt.bzl edits) — **drop,
superseded by Phases 0–3**. Cleanest sequencing: land the registry first,
then strip the interim plumbing from the worktrees before merge. After
merge + verification, delete `.dawn-build/` (the recipe now lives in CI; the
`.gitignore` entry can stay).

**Phase 5 — docs close-out** (workspace definition of done):
- CLAUDE.md: build-system section gains the registry (what it is, where,
  how deps resolve); gpu/debug blurbs if they mention Dawn sourcing.
- Memory: revise `dawn-prebuilt-x11-only.md` — the "override both copies"
  guidance becomes "the registry's dawn module ships Wayland-enabled Linux
  builds from kitten3d/dawn-builds".
- REVIEW.md §7: the sha-pinned-http_archive pattern gains its registry
  sibling; retire F4's watchpoint with a dated note.
- No skill currently mentions `dawn_prebuilt` (verified 2026-07-14), but
  `/bazel` should gain a "registry deps" paragraph.
- Findings doc: mark F1 resolved (disposition: registry), move toward
  `plans/complete/` with the rest.

**Dawn version bump procedure (document in the registry README):** dispatch
the workflow at the new upstream tag/commit → new release assets → add
`modules/dawn/<new-version>/` + metadata.json entry (new upstream mac/win
shas from the upstream release, new linux sha from CI, new integrity for the
wrapper) → bump `bazel_dep` version in gpu + debug → lockfiles → suite.

---

## 4. Risks & watchpoints

- **Registry/module drift:** the registry `MODULE.bazel` copy must match the
  archived one byte-for-byte (Bazel enforces — treat the error as "re-copy").
- **Autogenerated archives:** never point `source.json` or `extensions.bzl`
  at `codeload`/tag tarballs; uploaded assets only (byte-stability).
- **raw.githubusercontent availability:** fine at this scale; lockfile +
  `--repository_cache` mean it's only consulted on changed deps. Mirrors slot
  exists in `bazel_registry.json` if it ever matters.
- **Toolchain floor:** the CI artifact inherits `ubuntu-latest`'s glibc floor
  — same as upstream today. If gpu CI ever pins an *older* image, rebuild
  there.
- **Two Dawn copies during transition:** between Phases 3 and 4 the
  worktrees still reference the old rule — fine, they're overridden modules;
  just don't half-migrate one consumer on main.
- **Named escalation (not scope now):** migrating other third_party deps
  (imgui, glfw overlay needs, FastNoiseLite) into the registry. Trigger: the
  next time one needs a patch or is duplicated across modules.

## 5. Open decisions for the bootstrap session

1. Repo names (`kitten3d/registry`? `kitten3d/dawn-builds`? one combined
   repo is possible but mixes metadata-on-main with binary releases —
   two repos recommended).
2. Whether the wrapper exposes extra targets (e.g. `:includes` headers-only)
   — current consumers need only `@dawn`; start minimal.
3. Whether workspace CI (if/when it exists) gets a registry-health job
   (fetch-from-scratch weekly). Cheap insurance; optional.
