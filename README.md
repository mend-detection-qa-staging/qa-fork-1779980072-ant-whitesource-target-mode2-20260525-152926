# ant-whitesource-target-mode2

## Feature exercised

Forces the Mend bolt-4 SCM scanner's Ant resolver into **Mode 2 (WhitesourceXmlAntTask path)**
by including a `<whitesource>` target inside `build.xml`, covering catalog gap **G6.a**.

## Coverage gap

**G6.a**: Mode 2 of the Ant resolver had never been exercised by a probe. Project #1
(`ant-conflict-managers`) confirmed Mode 2 was attempted (log line:
`Ant-Resolver - scanWhitesourceAntModules - START`) but fell through because no
`<whitesource>` target existed. This probe adds that target.

## How Mode 2 works

Per `whitesource/unified-agent` `.claude/knowledge/resolvers/java-jvm.md` §3:

> **Mode 2: Whitesource XML Modules** (if `<whitesource>` target exists in build.xml):
> 1. Inject custom `WhitesourceXmlAntTask` into Ant project
> 2. Execute `<whitesource>` target
> 3. Parse `<module>` elements with `<path>` references
> 4. Each module becomes separate project with its own file-based dependencies

The `build.xml` in this probe defines:
- Two `<path>` elements (`module-a.path`, `module-b.path`) pointing at `lib/module-a/` and `lib/module-b/`.
- A `<whitesource>` target containing two `<module>` children referencing those paths.

### Why `ant whitesource` fails locally

`WhitesourceXmlAntTask` is not distributed as a standard Ant library. It is injected
into the running Ant process by the Mend scanner at scan time. Running `ant whitesource`
locally produces:

```
Problem: failed to create task or type whitesource
Cause: The name is undefined.
```

This failure is expected and harmless. The Mend scanner handles it via programmatic
task injection before parsing the target, not via Ant's own class loader.

## Repo structure

```
build.xml                    Mode-2 entry point: <whitesource> target + <path> defs
lib/
  module-a/
    commons-io-2.11.0.jar    SHA1: a2503f302b11ebde7ebc3df41daebe0e4eea3689
    commons-lang3-3.12.0.jar SHA1: c6842c86792ff03b9f1d1fe2aab8dc23aa6c6f0e
    commons-logging-1.2.jar  SHA1: 4bfc12adfe4842bf07b657f0369c4cb522955686
  module-b/
    hamcrest-core-1.3.jar    SHA1: 42a25dc3219429f0e5d060061f71acb49bf010a0
    junit-4.13.2.jar         SHA1: 8ac9e16d933b6fb43bc7f576336b8f4d7eb5ba12
    slf4j-api-1.7.36.jar     SHA1: 6c62681a2f655b49963a5983b8b0950a6120ae14
.whitesource                 Bucket A config: configMode LOCAL, java pinned
whitesource.config           ant.resolveDependencies=true, ivyResolveDependencies=false
expected-tree.json           schema 1.1, monorepo root, two subtrees
```

## Expected dependency tree

**Mode 2 fires (primary expectation):**

Two separate Mend projects:

- `module-a` (3 direct deps, flat — no transitives in Mode 2):
  - `commons-io-2.11.0.jar`
  - `commons-lang3-3.12.0.jar`
  - `commons-logging-1.2.jar`

- `module-b` (3 direct deps, flat):
  - `hamcrest-core-1.3.jar`
  - `junit-4.13.2.jar`
  - `slf4j-api-1.7.36.jar`

**Mode 3 fallback (if Mode 2 does not fire):**

A single flat Mend project containing all 6 JARs as direct deps. The comparator
can distinguish the two outcomes by checking `subtrees[]` length in the dep tree:
- 2 subtrees = Mode 2 fired correctly.
- 0 subtrees + flat root with 6 deps = Mode 3 fallback.

All deps are `source: "local"` — Mend uses SHA1 file fingerprinting, not Maven
coordinate resolution, for file-based JARs.

## Mend config

**Bucket A — default-emit** with `java` pinned to `17.0.19+10`.

Ant itself is NOT pinnable via `install-tool` — only the Java runtime is pinnable
via `scanSettings.versioning`. The Ant runtime version comes from whatever the
operator installs out-of-band. This is a known partial reproducibility limitation:
if the operator upgrades Ant, the scan behavior could differ. Flag this in any
downstream comparator run.

Additional dimension: `whitesource.config` at repo root sets:
- `ant.resolveDependencies=true` — required to enter the mode-evaluation chain.
- `ant.ivyResolveDependencies=false` — no `ivy.xml` present; skip Mode 1.
- `ant.pathIdIncludes=` (empty) — Mode-3 fallback uses default extension list.

## Probe metadata

| Field | Value |
|---|---|
| Pattern | G6.a (WhitesourceXmlAntTask / Mode 2) |
| Probe ID | ant-whitesource-target-mode2-20260525-152926 |
| Schema version | 1.1 |
| PM | ant (monorepo root + 2 subtrees) |
| PM version tested | ant-1.10.x |
| Java pinned | 17.0.19+10 |
| Generated | 2026-05-25 |
