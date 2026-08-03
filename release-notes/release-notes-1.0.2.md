# Release Notes — Amazon::Signature4::Lite 1.0.2

**Released:** Mon Aug 3 2026  
**Distribution:** `Amazon-Signature4-Lite-1.0.2`

---

## Bug Fixes

### S3 Path Encoding Fix (`Amazon::Signature4::Lite::sign`)

The most important change in this release is a correctness fix for AWS
Signature Version 4 signing of S3 requests involving object keys that
contain reserved or special characters (e.g. `#`, `%`, `+`).

**The problem:** AWS SigV4 requires path segments to be
percent-encoded *twice* for all services except S3, which requires
encoding *once*. The `sign` method was unconditionally applying a
second encoding pass via `_encode_path`, causing already-encoded S3
paths (e.g. `%23`) to be re-encoded to `%2523`. This produced a
`SignatureDoesNotMatch` error from S3 whenever object keys contained
reserved characters.

**The fix:** The canonical request path is now computed conditionally:

- **S3** (`service eq 's3'`): the path is used verbatim (single
  encoding, as the AWS spec requires).
- **All other services**: `_encode_path` applies the second encoding
  pass as before.

```perl
my $canon_path
  = ( $self->{service} eq 's3' ) ? $path : _encode_path($path);
```

---

## New Tests

- **`t/02-s3-path-encoding.t`** — new test suite covering S3 path
  encoding edge cases, including keys with `#`, `%`, spaces, and other
  reserved characters that previously produced incorrect signatures.

---

## Dependency Changes

- **Removed:** `Digest::SHA 6.04` from `requires` and
  `cpanfile`. `Digest::SHA` is a Perl core module since 5.10 and does
  not need to be declared as an explicit runtime prerequisite.
- **Retained:** `URI::Escape 5.34`

---

## Build System Updates

This release includes a significant overhaul of the
`CPAN::Maker::Bootstrapper`-managed build infrastructure. These
changes do not affect the installed module but improve developer
tooling reliability and correctness.

### `Makefile`

- Default goal changed from `all` to `$(TARBALL)`; `all` is now a
  phony alias.
- `cpanfile` generation refactored: now produces separate intermediate
  files (`cpanfile.requires`, `cpanfile.suggests`,
  `cpanfile.recommends`) and merges them, with support for the new
  `recommends` and `suggests` dependency tiers.
- New targets: `recommends`, `suggests` — dependency files for
  soft/optional dependencies.
- New `deps.mk` generation: `deps.mk` now depends on `.pm.in`/`.pl.in`
  source files directly (not built `.pm`/`.pl` targets), eliminating a
  chicken-and-egg problem during `make clean`.
- New `test` target: runs `prove -I lib -v t/`.
- New `check` target: syntax-checks and builds from `.in` sources.
- New `package` target: runs `clean`, then `LINT=on SCAN=on`.
- `SCANDEPS`, `CPAN_MAKER`, and `MD_UTILS` tool names updated to
  reflect renamed executables (`scandeps-static`, `cpan-maker`,
  `markdown-render`).
- `build-ci` now mounts the project directory into the container,
  records build duration, and symlinks `build.log`.
- `GIT_NAME`, `GIT_EMAIL`, `GITHUB_USER` lookups now suppress `git` errors gracefully.
- Added `config.mk` include for user-local overrides.
- Added `CMB_UPDATE_CHECK` and `CMB_VERSION_DRIFT` configuration variables.

### `.includes/perl.mk`

- Replaced `podextract` with `podchecker` (`Pod::Checker`) for routine
  compile-time POD validation. Both `.pm` and `.pl` syntax-check rules
  now also run `podchecker` and fail the build on POD errors.
- `perlcritic` and `perltidy` checks are now skipped entirely (rather
  than erroring) when the respective tools are not installed.
- Added `PERLCRITIC_SEVERITY` (default: `5`) and `PERLCRITIC_THEME`
  (default: `pbp`) variables.
- `perlcritic` rules now use `tee` to write output to the sentinel
  file, and pass `--theme` and `--severity` flags.
- Compile-skip list now also reads from a `compile.skip` file in the
  project root (in addition to `PERLWC_SKIP`).
- `%.pl` build rule now initialises `local_cleanfiles` and installs a
  `trap` for cleanup.
- Fixed a duplicate `2>/dev/null` in `diff -q` invocation in the
  `perltidy` sentinel rule.
- Added `check-syntax` as a `.PHONY` convenience alias for building
  all modules and scripts with syntax checking.
- Added `-include deps.mk` and `-include project.mk` with explanatory
  comments.
- Added `run_podextract` guard: emits a clear error if `Pod::Extract`
  is not installed when `POD=extract` or `POD=remove`.

### `.includes/update.mk`

- `post-update` now sets managed include files read-only after
  copying, and merges missing entries from the bootstrapper's
  `gitignore` into the project `.gitignore`.
- `update` now handles the optional `builder` script, and sets
  `Makefile` and includes back to read-only after updating.
- `update-available` now supports configurable `CMB_UPDATE_CHECK` and
  `CMB_VERSION_DRIFT` modes (`fail` / `warn` / `ignore`), and checks
  local file integrity via `md5sum` against the installed
  bootstrapper's checksum manifest.

### `.includes/git.mk`

- `git` target now supports `NO_COMMIT=1` to stage without committing.
- `git init` output suppressed.
- Shell commands properly chained with `;` and `\`.

### `.includes/release-notes.mk`

- Replaced the previous multi-step shell script (producing `.diffs`,
  `.lst`, `.tar.gz` files) with a single `cmb release-notes`
  invocation that generates
  `release-notes/release-notes-{version}.md`.

### `.gitignore`

- Added `**/*.checked`, `**/*.raw`, and `buildspec.yml.current` to ignore patterns.

---

## Upgrade Notes

- If you use S3 with object keys containing `#`, `%`, spaces, or other
  URL-reserved characters, upgrading to 1.0.2 is strongly
  recommended. Previous versions would generate invalid signatures for
  such keys.
- No changes to the public API. Existing code requires no
  modification.
- `Digest::SHA` no longer appears in `cpanfile` or `requires`; it
  remains a dependency but is satisfied by Perl core (5.10+).
