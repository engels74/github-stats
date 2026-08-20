# AGENTS.md

This file provides guidance to AI coding agents when working with code in this
repository.

One Zig executable that pulls GitHub user statistics and renders them into two
SVG images. No third-party dependencies.

## Zig version

Zig **0.16.0** exactly (`minimum_zig_version` in `build.zig.zon`; both workflows
pin `0.16.0` via `mlugg/setup-zig`). The code targets the 0.16 `std.Io` API and
will not compile on earlier versions:

- `pub fn main(init: std.process.Init) !void`, not `pub fn main() !void`
- `std.Io.File` / `std.Io.Dir`, not `std.fs.File` / `std.fs.Dir`
- `std.Io.Writer`, not `std.io.getStdOut().writer()`
- `io: std.Io` is threaded explicitly through every function that does I/O

Match the surrounding call signatures rather than recalling pre-0.16 idioms.

## Commands

| Command | What it does |
| --- | --- |
| `zig build` | Builds `zig-out/bin/github-stats`; default optimize is `ReleaseSafe` |
| `zig build run -- <args>` | Builds and runs, forwarding args after `--` |
| `zig build test` | Runs the suite (root module `src/main.zig`) |
| `zig test src/glob.zig --test-filter match` | One file / one test |
| `zig build release` | Cross-compiles ~50 targets at `ReleaseFast`; tag workflow only |
| `zig fmt src build.zig` | Formats; no CI job checks this |

`build.zig` wires `b.args` only into the `run` step, so `zig build test` takes no
`--test-filter`; drop to bare `zig test <file>` for a single test. That works for
every file except `src/main.zig`, which imports the build-generated `options`
module. There is no separate lint or typecheck step.

Running the binary needs a token or a JSON dump:

```bash
zig build run -- --access-token "$TOKEN" --json-output-file stats.json --debug
zig build run -- --json-input-file stats.json   # replay from disk, no network
```

Use `--json-input-file` when iterating on aggregation or templates — it skips
every API call.

## Where things live

- `src/main.zig` — arg struct, aggregation loop, template invocation.
- `src/statistics.zig` — every GraphQL/REST call, JSON shape, retry schedule,
  and the git-clone fallback. All API work belongs here.
- `src/http_client.zig` — `std.http.Client` wrapper; `graphql()` and `rest()`
  each retry 8 times and rebuild the client around a Zig keep-alive bug.
- `src/git.zig` — shells out to the `git` CLI; used by `build.zig` for the
  version string and by `statistics.zig` for the lines-changed fallback.
- `src/argparse.zig` — comptime-reflective parser with no knowledge of this
  app's options. `src/glob.zig`, `src/template.zig` — self-contained helpers.

## Adding a CLI option

The `Args` struct in `src/main.zig` is the single source of truth. `argparse`
reflects over its fields at comptime, so one field with a default yields the
flag, the env var, and the `--help` line. Do not edit `argparse.zig`.

- Field `foo_bar` → `--foo-bar` and env `FOO_BAR` (both matched
  case-insensitively).
- Precedence is CLI > environment > struct default.
- From the environment, a `bool` is true for any non-empty value except
  `false`.
- Only `?[]const u8`, `bool`, and integer types are supported; anything else is
  a `@compileError`.

## Adding a statistic to the overview image

`src/templates/*.svg` are `@embedFile`d, so template edits need a rebuild.
Placeholders resolve against the struct passed to `templateFill`, and an unknown
placeholder is a runtime `error.InvalidField`, not a compile error.

1. Add the field to the `aggregate_stats` struct literal in `src/main.zig`.
2. Accumulate it in the `for (stats.repositories)` loop; if it comes from the
   API, add it to `Repository` and its JSON parsing in `src/statistics.zig`.
3. Reference it as `{{ field_name }}` in `src/templates/overview.svg`.
4. Verify with `zig build && zig build run -- --json-input-file stats.json`.

Only unsigned integers and `[]const u8` render; unsigned integers get thousands
separators via `decimalToString`. `languages.svg` is different — `main.zig`'s
`languages()` pre-renders the `{{ progress }}` and `{{ lang_list }}` fragments
in Zig, so per-language markup lives in `main.zig`, not the SVG.

## Conventions

- Hard-wrap Zig source at 80 columns. The only exception in the tree is an
  unbreakable docs URL at `src/statistics.zig:759`.
- HTTP response bodies are allocated from the gpa, not the arena: free them with
  `client.allocator.free(response.body)`, as every call site in
  `statistics.zig` does. Everything else there is arena-allocated scratch.
- Data owned by `Statistics` comes from the caller's gpa and is released by the
  matching `deinit`.
- Parse JSON with `parseFromSliceLeaky(..., .{ .ignore_unknown_fields = true })`
  into an arena — GitHub adds response fields.
- Log through `std.log`. Verbosity is gated by the custom `logFn` and
  `log_level` in `main.zig`, so `std.debug.print` would bypass `--silent`.

## Gotchas

- **The environment scan is broad.** `argparse` walks the whole environment
  looking for a case-insensitive match against any `Args` field name. A stray
  `VERSION`, `DEBUG`, or `SILENT` in your shell changes behavior —
  `VERSION=1` makes the binary print its version and exit. Pass explicit CLI
  flags when running locally.
- **`owned_repos_only` changes what "contributions" means.** This repo's
  workflow sets `OWNED_REPOS_ONLY: "true"`. That path (`getOwnedRepos`) fills
  only `commit_contributions`, leaving the repo/issue/PR/review counters at
  zero, so the overview total is commits-only. The year-by-year path
  (`getReposByYear`) fills all five.
- **The lines-changed fallback clones repositories.** Once
  `/stats/contributors` returns 202/403/429 more times than `--max-retries`
  allows, `statistics.zig` calls `git.getLinesChanged`, which bare-clones the
  repo with the access token embedded in the URL. `main.yml` sets
  `MAX_RETRIES: 5` (the built-in default is 25) to reach that fallback quickly.
- **CI never runs the tests.** `.github/workflows/main.yml` runs `zig build`
  then the binary. Run `zig build test` yourself before pushing.
- **Generated SVGs do not belong on `master`.** The workflow checks out the
  `generated` branch before running the binary and commits `overview.svg` and
  `languages.svg` there. `src/templates/*.svg` are the sources.
- This is a copy of the upstream template `jstrieb/github-stats`. `README.md`
  and the `--version` banner in `src/main.zig` intentionally point upstream —
  leave that attribution in place.

## Reference

- `README.md` — end-user setup path (token scopes, repository secrets, `jq`
  analysis recipes) and the documented caveats about GitHub statistics
  accuracy. Read it when a question is about *using* the tool or about why the
  numbers look wrong; it says nothing about developing the Zig code.
