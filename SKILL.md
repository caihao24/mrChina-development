---
name: mrChina-development
description: Use when developing, reviewing, or refactoring madrat source or calc functions, especially for download/read/convert/calc structure, readSource/calcOutput contracts, aggregate behavior, isocountries handling, full-data wrapper generators, and ISO-vs-cell output design.
---

# mrChina-development

## Overview
Use this skill when working on madrat-style data pipelines in `mrChina`-like packages.

The goal is to keep source functions, calc functions, wrapper generators, and file outputs aligned with how `madrat::readSource()` and `madrat::calcOutput()` actually work.

## Core Rules

### 1. Separate source and calc responsibilities
For standard source types:
- `download<Type>()`: download raw input files into the current source-type directory
- `read<Type>()`: read raw files and return a `magpie` object
- `convert<Type>()`: standardize, harmonize, or fill data
- `calc<Type>()`: derive outputs and call `readSource()` for upstream source data

Do not make `read<Type>()` call `downloadSource()`.
Do not bypass `readSource()` from `calc<Type>()` for standard source data.

### 2. `read()` must return `magpie` for standard source types
Raw input can be CSV, XLSX, raster, ZIP, or other formats.
That is fine.
But for a standard madrat source chain, `read<Type>()` itself must convert it into a `magpie` object.

### 3. `convert()` returning `magpie` triggers ISO validation
`madrat::readSource()` enforces ISO-country checks on `convert<Type>()` output if it is a `magpie`.
This means:
- ISO country source: `convert()` may return `magpie`
- non-ISO cell source: do not keep a `convert<Type>()` layer that returns cell-level `magpie`

For valid cell-level standard source chains, prefer:
- `download<Type>()`
- `read<Type>()` returning cell-level `magpie`
- no `convert<Type>()`
- `calc<Type>()` calling `readSource()`

### 4. Avoid `getConfig("sourcefolder")` inside source/calc wrappers
Inside `downloadSource()` / `readSource()` / `calcOutput()` paths:
- `download<Type>()` should write to `.`
- `read<Type>()` should read from `.`
- do not use `getConfig("sourcefolder")` as the primary path

If a fallback is needed, prefer relative directory inference.
Only use `getConfig("sourcefolder")` in special calc wrappers that are not standard source types and need to locate external model files.

### 5. `download<Type>()` should return metadata
Standard download functions should return a list with source metadata for madrat tracking.
Preferred fields:
- `url`
- `doi`
- `title`
- `description`
- `unit`
- `author`
- `release_date`
- `license`

If metadata is omitted, `downloadSource()` / `calcOutput()` may emit warnings even when the download succeeds.

### 6. `calc<Type>()` return structure
Standard return for `calcOutput()` should be:
- `x`
- `weight`
- `description`
- `unit`
- `isocountries`

If there is no weight, return `weight = NULL` explicitly.

## Aggregate Logic

### 7. `aggregate = TRUE` only works if calc output is still aggregatable
If `calc<Type>()` already aggregates ISO to region, default madrat aggregation will likely fail or be semantically wrong.

Use `aggregate = FALSE` for outputs that are already:
- region level
- region x region bilateral
- cell level
- special non-ISO structures

If a type should support normal madrat aggregation, then `calc<Type>()` should keep data at ISO or cell level and not pre-aggregate to region.

## `isocountries` Rule

### 8. Set `isocountries = TRUE` only for full ISO output
This flag is about the first dimension of `x`, not about the topical scope of the data.

Set `isocountries = TRUE` only if output is a full ISO country set.
That includes China-focused data only when it has been expanded to full ISO via patterns like:
- start with `CHN`
- add `HKG`, `MAC`, `TWN` if needed
- run `toolCountryFill(..., NA)`

Set `isocountries = FALSE` for:
- region outputs
- cell outputs
- `CHA`-only outputs
- bilateral region matrices
- special parameter objects

## Timestep Alignment

### 9. Use `toolAlignTimesteps()` only where a calc explicitly needs timestep alignment
If a `calc<Type>()` must align its output years to the MAgPIE timestep set, use `toolAlignTimesteps()` inside that function.

Do not expose timestep alignment choices as user-facing `calcOutput()` parameters by default.
The developer maintaining the specific `calc<Type>()` should choose the appropriate method and document the reason in that function.

Default pattern:
- `toolAlignTimesteps(x)` uses `findset("time")`
- `method = "exact"` keeps exact year matches and fills missing target timesteps with `NA`
- existing target years keep their original values

Explicit developer choices:
- `method = "near"` / `"nearest"` for stepwise nearest-year assumptions
- `method = "average"` with `window = ...` for local moving-window means
- `tie = "earlier"` / `"later"` only when nearest-year ties matter

Use this helper near the end of `calc<Type>()`, before returning the standard list.
Do not use it in `read<Type>()`: raw source years do not need to match model timesteps.
Do not silently align years in generic wrappers unless the wrapper is explicitly responsible for final MAgPIE input delivery.

## Cell-Level Pattern

### 10. Valid cell-level source pattern
When source data is raster-like but should still be a standard madrat source:
- `read<Type>()` converts raster to cell-level `magpie`
- cell ids may be filtered by magclass, for example decimal points in coordinates may become `_`
- `calc<Type>()` should parse the actual stored cell ids, not assumed coordinate strings
- if needed, rebuild rasters from cell coordinates inside `calc<Type>()`

Do not assume the first-dimension labels survive unchanged after `as.magpie()`.

## Full Wrapper Pattern

### 11. `full<Type>()` wrappers should be thin orchestrators
Wrapper generators like `fullFERTILIZER()` or `fullMAGPIECHINA()` should:
- accept one explicit output directory from the user
- call standardized `calcOutput()` entry points
- pass `aggregate = FALSE` for already-aggregated, cell-level, or special outputs
- write final `.cs2` / `.cs3` files with stable names
- return invisibly, optionally with generated file paths

Avoid embedding source-specific path logic in the wrapper.

### 12. Archive only newly generated files
If a wrapper also creates a tarball:
- track exactly which files were generated in that run
- compress only those files
- do not tar the whole output directory unless that directory is guaranteed to contain only newly generated files

This avoids accidentally archiving large unrelated data trees.

### 13. File headers should carry provenance
When writing final `.cs2` / `.cs3` files for delivery, add comment header lines for:
- human-readable description
- unit
- origin call, for example `calcOutput(type = "CDGTargets", subtype = "old", aggregate = FALSE, file = "f15_intake_EATLancet_iso.cs3")`
- creation date
- package versions when useful, for example `(madrat X | mrChina Y)`

This makes generated inputs traceable without external logs.

### 14. Avoid `setwd()` in full wrappers
Do not change the process working directory just to create archives.
Prefer command forms such as:
- `system2("tar", c("-czf", archive, "-C", outputdir, basename(files)))`

This avoids side effects and keeps wrapper behavior predictable inside larger workflows.

## Source Sync and Download Policy

### 15. Distinguish public downloads from OSS-only downloads
If the team uses a local `inputdata` mirror synchronized from OSS, for example via an external `oss_sync.sh` script:
- keep `download<Type>()` for public internet sources that other users may need on fresh machines
- remove or deprecate package-internal download helpers that only wrap Aliyun OSS access
- make `read<Type>()` read from the synced local source tree instead of calling OSS commands

Do not keep package-level `download<Type>()` functions whose only purpose is OSS synchronization if that workflow has been replaced by an external sync script.

### 16. Support subtype subdirectories in synced source trees
For source types that are naturally organized by subtype, prefer a local layout such as:
- `sources/CDGTargets/old/<file>`
- `sources/CDGTargets/new/<file>`
- `sources/AfforestPotentialCluster/c200/<file>`
- `sources/AfforestPotentialCluster/c400w4/<file>`

`read<Type>()` should first look for `file.path(subtype, filename)` and optionally keep compatibility with older flat layouts.

## Naming and Compatibility

### 17. Rename source types decisively when the old name should disappear
If a source type is renamed, for example `Land` to `LandNBS`:
- rename the reader and source type consistently
- update documentation and exports
- remove the old compatibility wrapper if the old name should no longer be used

Do not keep legacy wrappers unless there is a real migration need.

## Common Pitfalls
- Returning raster or list objects from `read<Type>()` for a standard source chain
- Calling `read<Type>()` directly from `calc<Type>()` when the type should be a normal source
- Keeping `convert<Type>()` for cell-level source data and then hitting ISO validation errors
- Using `getConfig("sourcefolder")` inside `readSource()` or `downloadSource()` paths
- Marking China-only outputs as `isocountries = TRUE` without expanding to full ISO
- Aligning timesteps in `read<Type>()` instead of in the specific `calc<Type>()` that needs model timestep compatibility
- Exposing timestep alignment method choices to ordinary users when the method should be a developer decision
- Omitting `weight = NULL` or `isocountries` from calc return lists
- Using `aggregate = TRUE` on outputs already aggregated to region level
- Writing wrapper archives from the whole output directory instead of the generated file list
- Leaving `download<Type>()` without metadata and then chasing avoidable wrapper warnings
- Keeping OSS-only package download helpers after adopting a separate local sync workflow
- Using `setwd()` in wrappers just to build archives

## Quick Decisions
- Table/ISO source data: standard `download/read/convert/calc`
- Cell/raster source data: standard `download/read/calc`, usually no `convert`
- Pure generated parameter: `calc<Type>()` only, no source chain required
- Calc output must match MAgPIE timesteps: use `toolAlignTimesteps()` inside that calc, with the method chosen by the developer
- Special external model file input: may remain special calc if no clean source abstraction exists
- Delivery wrapper with archive: write files first, track them, tar only those files
- Team uses local OSS-synced `inputdata`: remove OSS-only package download helpers, keep public download helpers
