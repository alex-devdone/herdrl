# herdrl — resilient wrapper for remote [herdr](https://herdr.dev) sessions

`herdr --remote <host>` gives you a full herdr client against a herdr server on
another machine. `herdrl` wraps it so the client survives laptop sleep and
network drops, and — when you run it from inside a herdr pane — mirrors the
remote session's agents into your local sidebar.

```bash
herdrl --remote myhost --session my-session
```

Every argument passes through to `herdr` untouched. Without `--remote` it just
`exec`s `herdr`, so it's safe to alias.

## What it adds

- **Auto-reconnect.** Client dies from a dropped link → reconnect with 2→10s
  backoff. The remote server and its panes are untouched, so you land back
  exactly where you were.
- **Terminal reset on every disconnect.** Mouse-tracking and bracketed-paste
  modes off, leave the alt screen, restore cursor/keypad/SGR, `stty sane`.
  Without this, a client killed mid-session leaves the local terminal emitting
  escape junk on every click and paste.
- **Agent mirroring** (optional, see below).

## Install

```bash
git clone https://github.com/alex-devdone/herdrl.git ~/work/herdrl
ln -sf ~/work/herdrl/herdrl ~/.local/bin/herdrl     # anywhere on your PATH
```

Requires `herdr` ≥ 0.7.0 locally and on the remote host, and passwordless ssh
to that host.

## Agent mirroring

Run `herdrl` inside a herdr pane and it bridges the two herdr instances: the
remote session's agents (name + idle/working/blocked) show up in your *local*
agents sidebar, so a nested remote herdr isn't a black box.

This needs the
[Remote Agent Watch](https://github.com/alex-devdone/herdr-remote-agent-watch)
plugin — `herdrl` calls its `watch.sh` in bridge mode. Set `HERDR_WATCH` if you
cloned it somewhere other than `~/work/herdr-remote-agent-watch`. Not
installed? The mirroring is skipped and everything else still works.

The remote socket is discovered in the background via
`ssh <host> herdr session list`, retried for ~2 minutes: `herdr --remote`
*creates* the session on first attach, so at launch time the host often doesn't
know about it yet. The mirror keeps updating even while the client is
reconnecting, and is torn down when `herdrl` exits.

## Does herdr already do this?

Partly — checked against **herdr 0.7.5**:

| this wrapper | upstream herdr |
|---|---|
| reconnect after a drop | No. Network drops "disconnect only the client instead of killing remote panes" — the server survives, the client dies and you restart it by hand. [#1779](https://github.com/ogulcancelik/herdr/issues/1779) (mosh transport, to survive sleep/roaming) was closed as not planned. |
| terminal reset on disconnect | Attempted in [#939](https://github.com/ogulcancelik/herdr/issues/939), still reported on 0.7.3 — including `herdr --remote` on macOS, where mouse movement turns to garbage after a drop. |
| remote agents in the local sidebar | Not yet — [#1170](https://github.com/ogulcancelik/herdr/issues/1170) is open. |

herdr *does* handle **keepalive** itself: `[remote] manage_ssh_config = true`
(the default) generates an ssh config adding `ServerAliveInterval` /
`ServerAliveCountMax` when you haven't set them, plus ControlMaster
multiplexing. That's complementary — this wrapper passes no ssh options of its
own, so faster upstream drop detection just means the reconnect loop starts
sooner.

## Quit vs drop

| exit | meaning | wrapper does |
|---|---|---|
| 0 | you quit/detached the remote client cleanly | exits — you're done |
| 130 | Ctrl-C | exits |
| anything else | connection lost | resets the terminal, reconnects |

Ctrl-C while it's counting down to reconnect aborts the wrapper.

See also [sshl](https://github.com/alex-devdone/sshl), the same idea for plain
ssh into a remote tmux session.

## License

MIT — see [LICENSE](LICENSE).
