---
hide:
  - navigation
---

<div class="kw-hero" markdown>

# Stop waiting out your 5-hour window

<p class="kw-tagline">
The Codex CLI's usage window only starts when <em>you</em> do. Idle all night,
and those hours buy you nothing. <strong>keepwarm</strong> keeps windows rolling
around the clock, using the ChatGPT subscription you already pay for.
</p>

<div class="kw-cta" markdown>
[Install it](install.md){ .md-button .md-button--primary }
[How it works](how-it-works.md){ .md-button }
</div>

</div>

## The problem

Your 5-hour window starts at the **top of the hour containing your first
message**. It does not tick while you're away.

<figure class="kw-figure" markdown>
![Timeline showing seven idle hours during which no usage window runs, a window opening at 09:00 when the first message is sent, quota exhausted by 10:00, and four hours blocked with no access.](assets/problem.svg)
<figcaption>You were away for seven hours and still had to wait four more.</figcaption>
</figure>

The hours you spent asleep were never converted into anything. The window only
began when you sat down — and then you burned it in an hour and sat blocked for
the rest.

## The fix

keepwarm sends one tiny prompt just after each window expires, so windows tile
back to back whether you're at the keyboard or not.

!!! danger "There is a second limit, and it's the one that binds"

    ChatGPT plans meter Codex on **two** limits: the 5-hour window this tool
    tiles, and a **weekly** limit it cannot do anything about. Measured on a
    Plus plan, the weekly allowance is worth only about three full 5-hour
    windows — so you run out of week long before you run out of windows.

    keepwarm helps when your problem is *"my window hadn't started yet"*. It
    does nothing for *"I'm out of weekly credit"*, and its own pings draw on
    that same weekly budget. See
    [Two limits, not one](how-it-works.md#two-limits-not-one).

<figure class="kw-figure" markdown>
![The same timeline with keepwarm running: windows tile continuously at 04:00, 09:00 and 14:00, opened by small keepalive pings. Work starting at 08:30 uses the tail of the running window and then a fresh full-quota window at 09:00, with no blocked period.](assets/solution.svg)
<figcaption>Idle time now burns windows, so your working time doesn't have to.</figcaption>
</figure>

You do **not** get more quota per window. You get **more windows per day** — up
to the plan's natural ceiling of 24 ÷ 5 ≈ 4.8 — regardless of when you sit down.

## Quickstart

=== "macOS / Linux"

    ```sh
    git clone https://github.com/alanrliu/codex-keepwarm.git
    cd codex-keepwarm

    ./keepwarm install   # hourly cron job
    ./keepwarm ping      # open the first window
    ./keepwarm doctor    # confirm it's running
    ```

=== "Windows"

    ```powershell
    git clone https://github.com/alanrliu/codex-keepwarm.git
    cd codex-keepwarm

    .\keepwarm.ps1 install   # hourly scheduled task
    .\keepwarm.ps1 ping      # open the first window
    .\keepwarm.ps1 doctor    # confirm it's running
    ```

Needs a logged-in [Codex CLI](https://developers.openai.com/codex/cli). Full
notes on the [Install](install.md) page.

## Is it running?

You never have to wait for a ping to find out. Every hourly tick writes a log
line even when nothing is due, so the log doubles as a heartbeat.

<div class="kw-term" markdown>

```
$ ./keepwarm doctor

  [ok]   codex binary           /home/you/.local/bin/codex
  [ok]   cron job               installed (fires at :02 every hour)
  [ok]   cron daemon            running
  [ok]   last tick              0h 08m ago — cron is firing
  [ok]   window tracked         2026-09-21 13:00 -> 2026-09-21 18:00

  next cron tick   2026-09-21 17:02 (in 0h 47m)
  that tick will   log a WAIT line and exit (no API call)
  first ping tick  2026-09-21 18:02 (in 1h 47m)

  5 ok, 0 warning(s), 0 failing
```

</div>

`doctor` exits non-zero when something is wrong, so it works as a health probe
too.

## Honest notes

!!! success "No extra charge - it uses the subscription you already have"

    Pings come out of the usage your ChatGPT plan already includes. There is
    **no separate bill and nothing to enable**.

    They are not free, though: a ping is a normal `codex exec` turn and spends a
    slice of both limits. [What a ping costs](how-it-works.md#what-a-ping-costs)
    has the measured numbers.

    The one exception: if you've pointed Codex at an `OPENAI_API_KEY` instead of
    a ChatGPT login, those tokens are billed the usual way.

!!! note "It doesn't raise your limits"

    keepwarm uses your own subscription within your plan's existing limits. It
    doesn't increase them, bypass them, or share credentials, and it reads no
    secrets — authentication is whatever the Codex CLI already has. It does
    generate automated background usage on your account, so make sure that's
    consistent with your plan's terms.

!!! warning "Boundaries rotate"

    24 isn't divisible by 5. Boundaries at 07:00 / 12:00 / 17:00 / 22:00 today
    become 03:00 / 08:00 / 13:00 / 18:00 / 23:00 tomorrow. That's inherent to a
    five-hour window — see [How it works](how-it-works.md#boundaries-rotate).

A port of [claude-keepwarm](https://github.com/mamuncseru/claude-keepwarm) to
the Codex CLI. An unofficial community tool, not affiliated with OpenAI.
MIT licensed.
