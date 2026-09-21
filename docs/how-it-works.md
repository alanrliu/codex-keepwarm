# How it works

## The window model

keepwarm assumes:

```
window_end = floor_to_hour(first_message) + 5h
```

A first message at 09:37 puts you in a window that started at **09:00** and ends
at **14:00** — so starting mid-hour quietly costs you the part of the hour you
already spent. That's the model the tool encodes, and the reason `ping` is worth
running at the top of an hour.

## Why more windows, not more quota

This is the part worth being precise about, because it's easy to over-claim.

keepwarm does **not** raise your per-window quota. What it changes is *how many
windows exist in a day*. A five-hour window means a ceiling of 24 ÷ 5 ≈ 4.8
windows per day — but you only collect one when a window actually opens, and a
window only opens when a message is sent.

Without keepwarm, you collect windows only while you're awake and working. Every
idle stretch is a window that never existed. With keepwarm, the idle stretch
opens and expires a window on its own, so by the time you sit down you're either
inside a fresh one or minutes away from the next.

<figure class="kw-figure" markdown>
![The same timeline with keepwarm running: windows tile continuously at 04:00, 09:00 and 14:00, opened by small keepalive pings, with work starting at 08:30 and never blocked.](assets/solution.svg)
<figcaption>Windows tile whether or not you're at the keyboard.</figcaption>
</figure>

The worst case is that you start work just after a boundary and get a full fresh
window. The best case is that you start just before one and get the tail of the
running window *plus* a full fresh one. Neither case involves waiting.

## The hourly tick

The scheduled job runs **every hour**, not every five. Each tick compares now
against `window_end + SLACK_MINUTES` and exits immediately if it isn't past.

That's deliberate. A rigid five-hourly schedule drifts: the moment one run is
late, missed, or lands early, every subsequent run is out of phase with the real
boundary, and there's no mechanism to recover. An hourly tick with a recorded
window is self-correcting — a missed run just means the next tick notices the
window is overdue and acts.

It's also what makes liveness observable. Every tick logs a line even when it
does nothing, so the log is a heartbeat and `doctor` can tell you the schedule is
alive without spending a ping.

## What each tick decides

| Situation | What happens | Cost |
| --- | --- | --- |
| Window still live | Log `WAIT`, exit | none |
| Window expired | Ping, open a new window, log `OK` | one ping (see below) |
| You were active after the boundary | Log `SKIP`, record the window | none |
| Ping came back rate-limited | Log `LIMITED`, leave state alone, retry next tick | one ping |
| Transient local failure | Retry in-process, then give up until next tick | up to 3 pings |

### It skips redundant pings

If you were using Codex yourself after the boundary, your own messages already
opened the new window. keepwarm notices — it stamps a reference file at the
boundary time and looks for newer rollout files under `~/.codex/sessions` — and
records the window without spending anything.

Its own pings can never trigger this: they run with `--ephemeral`, which writes
no rollout file at all.

### It self-corrects

A `LIMITED` result means the old window hadn't really expired — or that the
weekly limit is exhausted — so the recorded state is left untouched and the next
tick tries again. This is why a wrong guess
about the boundary degrades into a retry rather than a drift.

Transient local failures are different: those are retried *within* the same run,
because otherwise a single blip costs a full hour of window. This isn't
hypothetical — it's why the retry exists. During development a ping failed with
zero API duration and zero tokens (it never reached the network), recovered an
hour later, and cost an hour of window in the process.

## Two limits, not one

This is the part that decides whether keepwarm is worth running for you.

ChatGPT plans meter Codex on two windows at once, and the CLI's own backend
reports both:

| Limit | Length | What keepwarm does about it |
| --- | --- | --- |
| Primary | 5 hours | Tiles it — this is the whole point of the tool |
| Secondary | 7 days | Nothing. It cannot be tiled, reset or worked around |

Tiling the 5-hour window only pays off while the weekly limit has room. On a
Plus plan it does not have much: measured against the primary window, the weekly
allowance is worth roughly **three full 5-hour windows**. A week holds about 34
five-hour windows, so you can fill only a small fraction of them no matter how
diligently they are opened.

So be honest with yourself about which problem you have:

- *"I sat down at 09:00 and my window hadn't started, so I lost the morning."*
  keepwarm fixes exactly this.
- *"I'm out of credit until Thursday."* keepwarm does not help, and its pings
  come out of the same weekly budget that ran out.

## What a ping costs

One ping is an ordinary `codex exec` turn. Codex has no equivalent of a
replaceable system prompt or a way to drop tool schemas from the request, so the
payload is whatever the CLI normally sends, whatever the prompt says:

```
input 13,629 (11,520 of it cached)  +  output 5
```

Measured against the backend's own usage figures on a Plus plan, that is about:

| Against | Per ping | At ~4.8 pings a day |
| --- | --- | --- |
| The 5-hour window it opens | ~0.3% | ~0.3% of each window |
| The weekly limit | ~0.1% | ~3% of the week |

So a ping costs well under one percent of the window it buys you — that part of
the trade is clearly worth it. The weekly figure is the one to weigh: a few
percent of your week spent on keepalives, in exchange for windows that are
already open when you sit down.

!!! note "How these were measured"

    `used_percent` from the backend's usage endpoint is an integer, so a single
    ping moves it by less than one point and cannot be read directly. The
    numbers above come from ten pings back to back: the primary window went
    92% → 95% and the weekly went 55% → 56%. That makes the primary figure solid
    and the weekly one coarse — one tick in ten pings is consistent with
    anything from about 0.05% to 0.15% each.

Flags do keep the ping from being worse than it has to be, even though they
don't touch the token count:

| Flag | Effect |
| --- | --- |
| `--ephemeral` | No rollout file under `~/.codex/sessions` |
| `--ignore-user-config` | No `config.toml`: no MCP servers, no profile |
| `--sandbox read-only` | The ping can't write to disk |
| `--skip-git-repo-check` | The state directory needn't be a repo |
| `--json` | A `turn.completed` event is the success signal |

!!! info "Why stdin is closed"

    `codex exec` reads its prompt from stdin whenever stdin isn't a TTY — which,
    under cron or Task Scheduler, is always. Without `</dev/null` the ping would
    block on the job's own stdin.

## Boundaries rotate

24 isn't divisible by 5, so a tiling schedule can't hold a fixed daily phase.
Boundaries at 07:00 / 12:00 / 17:00 / 22:00 today become 03:00 / 08:00 / 13:00 /
18:00 / 23:00 tomorrow, shifting an hour earlier each day.

This is inherent to a five-hour window, not something the tool can fix. You can
re-phase whenever you like by running `keepwarm ping` at the top of an hour you
want a boundary on — that opens a window there and every subsequent boundary
follows from it.

## The model is inferred, not documented

The hour-anchoring above matches observed behaviour, but OpenAI doesn't document
it. If the real anchoring differs, pings will occasionally land early — which
surfaces as a `LIMITED` line and a retry, not as breakage.

Codex's backend does report the real window state, including exactly when the
current one resets. keepwarm does not read it: that endpoint is undocumented and
reading it means handling the stored OAuth token, and the guess-and-retry design
inherited from claude-keepwarm degrades safely without either. It is the obvious
next improvement if the guessing ever proves annoying.

If you see `LIMITED` repeatedly, raise `SLACK_MINUTES`. That's the knob for
exactly this uncertainty.
