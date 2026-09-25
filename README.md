# force-ignore-platform

Probe for pnpm 11.28 features: `forceIgnoresPlatform` setting
(`tree_structure` category) and `pnpm install` execution-semantics
changes for TLS certificate verification (`install_command` category).

## Feature exercised

### 1. `forceIgnoresPlatform` (tree_structure)

pnpm 11.28 introduces the `force-ignore-platform` setting in `.npmrc`
(or `forceIgnoresPlatform` in `pnpm-workspace.yaml`). When `true`,
pnpm installs optional dependencies regardless of whether the current
host OS/CPU matches the package's `os`, `cpu`, or `libc` constraints.

This probe uses `esbuild@0.24.2` as the root dependency. esbuild
ships 25 platform-specific optional packages (e.g.
`@esbuild/linux-x64`, `@esbuild/darwin-arm64`, `@esbuild/win32-arm64`,
etc.), each gated by `os` and `cpu` fields. Without
`force-ignore-platform=true`, only the platform-matching variant is
installed. With it enabled, **all 25 variants appear in the lockfile
and are installed unconditionally**.

The key test: Mend reads the lockfile (not the installed
`node_modules/`), so it should report all platform variants as long as
they appear in `pnpm-lock.yaml`. The `force-ignore-platform` setting
changes what pnpm writes to the lockfile — Mend is passive. This probe
verifies Mend does NOT silently filter platform-constrained optionals
at scan time.

### 2. `strict-ssl=false` (install_command)

pnpm 11.28 changes how `strict-ssl` and `cafile` in `.npmrc` interact
during install. The flag now applies to both registry fetches and
git+SSH dependency cloning. `.npmrc` in this probe includes
`strict-ssl=false` to exercise this execution-semantics path. This
setting does not affect the dependency tree shape — it is a scan-time
install hint for the Mend pre-step runner.

## Project layout

```
force-ignore-platform-20260925-110242/
├── .npmrc                   # force-ignore-platform=true, strict-ssl=false
├── .whitesource             # Bucket A: versioning pins
├── README.md
├── expected-tree.json
├── package.json             # esbuild@0.24.2
└── pnpm-lock.yaml           # lockfileVersion 9.0, all 25 esbuild variants
```

## Expected dependency tree

Root depends on:
- `esbuild@0.24.2` (registry, main group)

`esbuild@0.24.2` has 25 optional transitive dependencies — all
platform-variant packages at version `0.24.2`. Each is:
- `optional: true`
- `source: "registry"`
- `group: "main"` (they appear in the main install group)
- No further transitive deps of their own

Platform packages included (all at 0.24.2):
- `@esbuild/aix-ppc64`
- `@esbuild/android-arm`
- `@esbuild/android-arm64`
- `@esbuild/android-x64`
- `@esbuild/darwin-arm64`
- `@esbuild/darwin-x64`
- `@esbuild/freebsd-arm64`
- `@esbuild/freebsd-x64`
- `@esbuild/linux-arm`
- `@esbuild/linux-arm64`
- `@esbuild/linux-ia32`
- `@esbuild/linux-loong64`
- `@esbuild/linux-mips64el`
- `@esbuild/linux-ppc64`
- `@esbuild/linux-riscv64`
- `@esbuild/linux-s390x`
- `@esbuild/linux-x64`
- `@esbuild/netbsd-arm64`
- `@esbuild/netbsd-x64`
- `@esbuild/openbsd-arm64`
- `@esbuild/openbsd-x64`
- `@esbuild/sunos-x64`
- `@esbuild/win32-arm64`
- `@esbuild/win32-ia32`
- `@esbuild/win32-x64`

## Mend config

**Bucket A** — pnpm has no dynamic version detection from the manifest.
This probe emits `.whitesource` with `scanSettings.versioning` pinning:
- `pnpm: "11.28.0"` (the version under test)
- `node: "20.19.2"` (LTS at time of probe generation)

`configMode` is `"AUTO"` (no `whitesource.config` ships with this probe).

## Mend failure modes targeted

1. Platform-constrained optional deps silently dropped — Mend should
   report all 25 `@esbuild/*` variants because they appear in the
   lockfile, regardless of the scan host's OS/CPU.
2. `optional: true` flag absent on platform variant entries.
3. `force-ignore-platform` `.npmrc` key misread as an unknown lockfile
   field, causing a YAML parse error in the pnpm resolver.
4. Only the locally-matching platform variant reported (Mend incorrectly
   applies platform filtering on top of lockfile reading).

## Resolver note

Mend's `PnpmLockCollector` is lockfile-driven. It reads `pnpm-lock.yaml`
and builds the tree from the `packages` + `snapshots` sections. The
`force-ignore-platform` setting is a pnpm install-time knob — Mend does
not read `.npmrc` during lockfile parsing. The expected tree therefore
reflects the lockfile contents unconditionally: all 25 optional platform
variants appear.

The resolver file (`javascript.md`, section 6) does not explicitly
document `force-ignore-platform` behavior — this probe is exploratory
for that dimension. The `optional` tracking behavior (tracked separately
in lock file, per the resolver notes) is regression-bound.

## Source

https://github.com/pnpm/pnpm/releases/tag/v11.28.0
