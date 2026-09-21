# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning is [SemVer](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-09-21

First release of the Codex port. Forked from
[claude-keepwarm 1.0.0](https://github.com/mamuncseru/claude-keepwarm), whose
window model, self-correcting hourly tick, command surface and test approach
are carried over unchanged.

### Changed — the port itself

- The keepalive is now `codex exec`, run with `--sandbox read-only`,
  `--skip-git-repo-check`, `--ephemeral`, `--color never` and `--json`, plus
  `--ignore-user-config` unless `IGNORE_USER_CONFIG=0`.
- Success is decided by the arrival of a `turn.completed` event, not by an exit
  status. `codex exec` also emits `error` events it recovers from, so neither
  signal is trustworthy on its own.
- Binary detection looks for `codex`: `PATH`, the VS Code / Cursor bundles, the
  standalone installer's versioned directory under `~/.codex/packages`, then
  the usual package-manager locations. `CLAUDE_BIN` is now `CODEX_BIN`.
- The activity check reads `~/.codex/sessions`. keepwarm's own pings use
  `--ephemeral` and write no rollout file, so they can never be mistaken for
  your own activity.
- Rate-limit and authentication detection match what the Codex CLI actually
  prints (`You've hit your usage limit`, `401 Unauthorized`, and friends), and
  the auth advice points at `codex login`.
- `MODEL` now defaults to empty, meaning the CLI's own default. The window is
  account-wide, so the model only changes what the ping is billed at.
- `PING_SYSTEM_PROMPT` is gone. Codex has no replaceable system prompt and no
  way to drop tool schemas from the request, so the flags that made the Claude
  version cost a few hundred tokens have no equivalent here.

### Added

- `PING_TIMEOUT` (default 120s), and the machinery to enforce it on both ports.
  A Codex that cannot authenticate retries its connection for around 30
  seconds; a wedged one would otherwise hold the lock until something killed
  it. The Unix port backgrounds the ping and polls, because `timeout(1)` is
  GNU-only; the Windows port uses `Start-Process` and `WaitForExit`.
- `IGNORE_USER_CONFIG`, so anyone pointing Codex at their own model provider
  can stop the ping from bypassing it.
- stdin is closed for every ping. `codex exec` reads its prompt from stdin
  whenever stdin is not a TTY, which under cron and Task Scheduler is always.
- Three bash tests and one PowerShell test covering the timeout, lock release
  after a kill, and the `turn.completed` success rule (28 and 17 in total).

### Notes

Pings come out of the usage your ChatGPT plan already includes; there is no
separate bill. The exception is an `OPENAI_API_KEY` instead of a ChatGPT login,
which bills normally.

The part worth reading before installing this: ChatGPT plans enforce a
**weekly** limit alongside the 5-hour window, and tiling cannot do anything
about it. Measured on Plus, the weekly allowance is worth roughly three full
5-hour windows, and keepwarm's own pings spend about 3% of it per week. See
[How it works](docs/how-it-works.md#two-limits-not-one).

The rendered social PNGs from upstream were removed rather than shipped with
the wrong branding; `docs/assets/social/card.svg` is the source.

[1.0.0]: https://github.com/alanrliu/codex-keepwarm/releases/tag/v1.0.0
