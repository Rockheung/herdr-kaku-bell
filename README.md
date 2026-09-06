# herdr-kaku-bell

English · [한국어](README.ko.md)

Lights up a [kaku](https://github.com/tw93/Kaku) tab when an agent is waiting on you.

**This is a [herdr](https://github.com/herdrdev/herdr) plugin.** Nothing is installed into
kaku. But kaku is what draws the indicator, so this only makes sense when kaku is your
outer terminal.

## What it needs

| | | |
|---|---|---|
| herdr | 0.8.0 or newer | where the plugin is installed |
| kaku | — | where the indicator is drawn; must be the outer terminal |
| macOS | — | relies on `ps` output and writing to `/dev/ttysNNN` |

On the kaku side there is nothing to install, only two settings worth checking. Details in
[Recommended kaku settings](#recommended-kaku-settings).

- `bell_tab_indicator` — the tab dot. **On by default.** Nothing to do unless you have
  explicitly set it to `false`.
- `bell_dock_badge` — the Dock badge. **Off by default.** Worth turning on if you keep
  kaku behind other apps.

To also get herdr's notifications, see
[Recommended herdr settings](#recommended-herdr-settings). On kaku that takes more than one
config line: herdr has to be told what the terminal is.

## Why it is needed

kaku draws an orange dot on a tab that receives a BEL, and clears it when you open that tab.
The feature exists to be used as a completion signal.

herdr, however, only sends a toast when an agent turns `blocked` or `done` — it does not
emit a BEL. The `TerminalBell` that herdr forwards to the outer terminal is produced only
when a program inside a pane actually writes a BEL, and even then only the focused pane's
bell gets through ([herdrdev/herdr#3095](https://github.com/herdrdev/herdr/issues/3095)).
The background tabs — the ones that actually need the indicator — stay silent.

The problem grows with `herdr --remote` across several servers. Nothing in the tab bar tells
you which server has an agent standing still, waiting for an answer.

This plugin writes the BEL from outside. It finds the tty each herdr client occupies via
`ps` and writes `\a` into it directly.

## Install

```sh
herdr plugin install Rockheung/herdr-kaku-bell
```

The `[[startup]]` hook launches the watcher when the herdr server starts. To use it now
without restarting the server, run it yourself:

```sh
bin/kaku-bell watch --daemon
```

For local development, link the working directory instead:

```sh
herdr plugin link /path/to/herdr-kaku-bell
```

## How it works

- `bin/kaku-bell ring [--local|<ssh-target>]` — writes a BEL to the kaku tab that hosts the
  given herdr session.
- `bin/kaku-bell watch [--daemon]` — watches local and remote sessions and rings the tab
  when `blocked` or `done` **newly** appears.

Things the design is careful about:

- **Identifying targets**: the child `ssh` process shares the same tty and also carries
  `ssh://target` on its command line. Only processes whose `argv[0]` is `herdr` and whose
  `argv[1]` is `--remote` are counted. The target value is passed through untouched, since
  herdr hands it straight to `ssh` — config aliases, `user@host`, `ssh://` URIs and ports
  all arrive here.
- **JSON**: the `agent list` response is read with a parser. Agent titles contain whatever
  prompt the user typed, so scraping braces and quotes as text breaks.
- **Failed lookups**: not treated as "nothing there". The previous state is kept, so a
  network drop and reconnect does not produce a phantom alert.
- **First cycle**: records state without ringing. Starting the watcher does not set off
  every agent that was already waiting.
- **Parallel lookups**: one thread per session. One slow host timing out does not delay the
  cycle.
- **Resilience**: a failed cycle is logged and the loop continues. The watcher dying
  silently would mean alerts simply stop arriving with no way to notice.

The target list is rebuilt from `ps` every cycle, so it follows reconnects that change the
tty and sessions that come and go. Remote lookups reuse connections through ssh
`ControlMaster`.

## Recommended kaku settings

The tab dot needs nothing — `bell_tab_indicator` is on by default. The **Dock badge is off
by default**, though. To see how many are pending while kaku sits behind other apps, add one
line to `~/.config/kaku/kaku.lua`:

```lua
config.bell_dock_badge = true
```

It is easy to miss because it is not in kaku's documentation. The badge clears when the kaku
window takes focus or you switch tabs.

## Recommended herdr settings

This plugin only handles the tab indicator, which tells you *which* session. To learn *what*
happened, turn on herdr's notifications as well. They distinguish `blocked` as
`claude needs attention` from `done` as `claude finished`.

```toml
[ui.toast]
delivery = "terminal"
```

**On kaku this alone is not enough.** herdr picks the notification sequence by inspecting
`TERM_PROGRAM` and `TERM`. kaku is a WezTerm fork but announces itself under its own name
(`TERM_PROGRAM=Kaku`, `TERM=xterm-256color`), so it matches no branch. herdr then emits no
sequence at all while still returning `shown: true` — do not take the response as proof that
anything was displayed.
([herdrdev/herdr#2513](https://github.com/herdrdev/herdr/issues/2513))

kaku does read OSC 9, so telling herdr it belongs to the WezTerm family matches its actual
capability. Scope it to herdr so nothing else is affected:

```sh
# ~/.zshrc.local
herdr() { TERM_PROGRAM=WezTerm command herdr "$@"; }
```

Environment variables are read at process start, so clients that are already running must be
restarted. With `herdr --remote` across several servers, that means every tab.

The trade-off: notifications now carry the kaku icon, but OSC 9 takes a single body string,
so title and body are joined into one line. The notification title becomes `Kaku` and the
body reads `claude needs attention: ~ · 1`. If you would rather read the state straight from
the title, use `delivery = "system"` — that path goes through `osascript`, so notifications
appear under the name "Script Editor".

## Configuration

| Environment variable | Default | Meaning |
|---|---|---|
| `HERDR_KAKU_BELL_INTERVAL` | `1` | polling interval in seconds; fractions accepted |

Querying seven sessions in parallel takes about 0.2 seconds, so a shorter interval costs
little — `0.5` works. With many servers or a slow link the lookup can exceed the interval;
the next cycle is simply delayed, nothing breaks.

State and logs live in `/tmp/herdr-kaku-bell/`. ssh errors accumulate in `watch.log`.

## Limitations

- It polls. herdr 0.8.2 does not actually invoke the plugin event hook
  (`pane.agent_status_changed`) — the manifest declares it anyway, so on a version where the
  hook fires it will respond first.
- Multiple herdr sessions (`--session`) are not supported. Only the default session and
  `--remote` are counted.
- `done` means "unseen background work finished", so herdr flips it to `idle` once you open
  the tab. When that races the polling cycle, the ring can arrive a beat late.
- macOS only. It relies on `ps -axo tty=` output and writing to `/dev/ttysNNN`.
- No tab indicator outside kaku. How a terminal treats a BEL varies; other WezTerm-family
  terminals may behave similarly, but this has not been verified.

## License

MIT
