# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

A single Zig executable that pulls GitHub user statistics and renders them into two SVG images. No external dependencies (`build.zig.zon` has `.dependencies = .{}`).

## Essential Commands

Zig **0.16.0** is required (`minimum_zig_version` in `build.zig.zon`; both workflows install `0.16.0` via `mlugg/setup-zig`). The code uses the 0.16 `std.Io` API — `pub fn main(init: std.process.Init)`, `std.Io.File`, `std.Io.Writer` — and will not compile on earlier versions.

| Command | What it does |
| --- | --- |
| `zig build` | Builds `zig-out/bin/github-stats`; default optimize mode is `ReleaseSafe` |
| `zig build run -- <args>` | Builds and runs, forwarding args after `--` |
| `zig build test` | Runs all tests (root module is `src/main.zig`) |
| `zig build release` | Cross-compiles ~50 targets at `ReleaseFast`; used only by the tag-triggered release workflow |
| `zig test src/glob.zig` | Tests one file directly — works only for files that import just `std` |
| `zig fmt src build.zig` | Formats sources; no CI job checks this |

`build.zig` exposes no test-filter option, so `zig build test` always runs the whole suite. `src/main.zig` cannot be run through bare `zig test` because it imports the build-generated `options` module. There is no separate lint or type-check step.

Running the binary requires either a token or a JSON input file:

```bash
zig build run -- --access-token "$TOKEN" --json-output-file stats.json --debug
zig build run -- --json-input-file stats.json   # replay from disk, no network
```

Use `--json-input-file` when iterating on aggregation or templates — it skips every API call.

## Architecture Overview

`main()` in `src/main.zig` drives four stages:

1. `argparse.parse` populates the `Args` struct from CLI args, then environment, then defaults.
2. Statistics come from `Statistics.initFromJson` (`--json-input-file`) or `Statistics.init` (GitHub API through `HttpClient`).
3. `main.zig` folds `stats.repositories` into a local anonymous `aggregate_stats` struct, applying `--exclude-repos` / `--exclude-langs` glob filters and `--exclude-private`.
4. `template.fill` substitutes `{{ field }}` placeholders in the SVG templates against `aggregate_stats`, writing `overview.svg` and `languages.svg`.

Module responsibilities:

- `src/argparse.zig` — comptime-reflective parser; contains no knowledge of this app's options.
- `src/statistics.zig` — all GraphQL/REST calls, JSON shapes, retry scheduling, and the git-clone fallback. Largest file; API work belongs here.
- `src/http_client.zig` — thin `std.http.Client` wrapper. `graphql()` and `rest()` each retry 8 times and rebuild the client to work around a Zig keep-alive bug.
- `src/git.zig` — shells out to the `git` CLI, used both by `build.zig` for the version string and by `statistics.zig` for the lines-changed fallback.
- `src/glob.zig`, `src/template.zig` — self-contained helpers; the only unit-tested modules.

## Configuration Derives From One Struct

The `Args` struct in `src/main.zig` is the single source of truth. `argparse.parse` reflects over its fields at comptime, so every field automatically produces:

- a CLI flag `--field-name` (underscores become dashes, matched case-insensitively), and
- an environment variable matched case-insensitively against the field name (`ACCESS_TOKEN` → `access_token`).

Precedence is CLI > environment > struct default: `setFromCli` marks fields as seen and later passes skip them.  From the environment, a `bool` is true for any non-empty value other than `false`.

To add an option, add one field with a default to `Args`. Do not edit `argparse.zig` — flag parsing, env binding, and `--help` output all derive from the field. Only `?[]const u8`, `bool`, and integer types are supported; anything else is a `@compileError`.

## Adding a Statistic to the Overview Image

`src/templates/*.svg` are `@embedFile`d at compile time, so template edits require a rebuild. Placeholders resolve against the struct passed to `templateFill`, and an unknown placeholder is a runtime `error.InvalidField`, not a compile error.

1. Add the field to the `aggregate_stats` struct literal in `src/main.zig`.
2. Accumulate it in the `for (stats.repositories)` loop; if it comes from the API, add the source field to `Repository` and its JSON parsing in `src/statistics.zig`.
3. Reference it as `{{ field_name }}` in `src/templates/overview.svg`.
4. Verify with `zig build && zig build run -- --json-input-file stats.json`.

Only unsigned integers and `[]const u8` are renderable; unsigned integers get thousands separators via `decimalToString`. `languages.svg` works differently — `main.zig`'s `languages()` pre-renders the `{{ progress }}` and `{{ lang_list }}` HTML fragments in Zig, so per-language markup lives in `main.zig`, not the SVG.

## Repository Conventions

- Hard-wrap Zig source at 80 columns. Every file obeys this; the sole exception is an unbreakable documentation URL in a comment at `src/statistics.zig:759`.
- Keep allocator roles separate. Data owned by `Statistics` is allocated from the caller's gpa and released by a matching `deinit`; per-request scratch comes from the `ArenaAllocator` threaded through as `arena`. Freeing arena memory with the gpa is the failure mode to avoid.
- Call `std.json.parseFromSliceLeaky` with `.{ .ignore_unknown_fields = true }` into an arena, as every existing call site does — GitHub adds response fields.
- Log through `std.log`. Verbosity is gated by the custom `logFn` and `log_level` in `main.zig`, not by build mode, so `std.debug.print` would bypass `--silent`.

## Critical Gotchas

- **Environment matching is broad.** `argparse` scans the entire environment for a case-insensitive match against any `Args` field name. A stray `VERSION`, `DEBUG`, or `SILENT` in your shell changes behavior — `VERSION=1` makes the binary print its version and exit. Pass explicit CLI flags when debugging locally.
- **`owned_repos_only` changes what "contributions" counts.** This repository's workflow sets `OWNED_REPOS_ONLY: "true"`. That path (`getOwnedRepos`) fills only `commit_contributions`, leaving `repo_`, `issue_`, `pr_`, and `review_contributions` at zero, so the overview total is commits-only. The year-by-year path (`getReposByYear`) fills all five.
- **The lines-changed fallback clones repositories.** Once `/stats/contributors` returns 202/403/429 more times than `--max-retries` allows, `statistics.zig` calls `git.getLinesChanged`, which bare-clones the repository with the access token embedded in the URL. The workflow deliberately sets `MAX_RETRIES: 5` (the built-in default is 25) to reach this fallback quickly.
- **CI never runs the tests.** `.github/workflows/main.yml` runs `zig build` and then the binary. Run `zig build test` yourself before pushing.
- **Generated SVGs do not belong on `master`.** The workflow checks out the `generated` branch before running the binary and commits `overview.svg` and `languages.svg` there. `src/templates/*.svg` are the sources; repo-root SVGs are output.
- This is a copy of the upstream template `jstrieb/github-stats`. `README.md` and the `--version` banner in `src/main.zig` intentionally point upstream; leave that attribution in place.

## Additional Documentation

- `README.md` — Read for the end-user setup path (token scopes, repository secrets, `jq` analysis recipes) and the documented caveats about GitHub statistics accuracy. It targets people copying the template, not people developing the Zig code.
