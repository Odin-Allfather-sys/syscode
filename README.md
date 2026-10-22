# syscode

One-line system health codes for Linux, built to be handed to an AI.

`syscode` runs a quick, read-only scan of your machine and prints a compact
code like:

```
SYS1-FED43-D23-M29-SW02-LD006-S0-K0-U038-J12-C0/13-NT+
```

Paste that code together with [CODES.md](CODES.md) into any AI assistant and
it can tell you what is healthy, what to watch, and what needs action. No
agent access to your machine required, nothing leaves your box but 60
characters.

## Install

```bash
git clone https://github.com/Odin-Allfather-sys/syscode
sudo install -m 0755 syscode/syscode /usr/local/bin/syscode
```

## Usage

| Command | What you get |
|---|---|
| `syscode` | the code + quick hints |
| `syscode --code` | just the code (pipe-friendly) |
| `syscode --explain` | human-readable table with OK/WATCH/ALERT per segment |
| `syscode --no-net` | skip the internet check |
| `syscode --version` | version |

Exit codes: `0` clean, `1` at least one WATCH, `2` at least one ALERT.
Handy for cron:

```bash
syscode --code || notify-send "syscode flagged your machine"
```

## What it checks

- distro and version
- disk usage of every real mounted filesystem (worst one reported)
- RAM and swap usage
- load average vs CPU count
- failed systemd units
- pending reboot (newer kernel installed than running)
- uptime
- journal error count this boot
- unhealthy containers (docker or podman, optional)
- internet reachability (optional)

Everything is read-only. No dependencies beyond bash, coreutils, and
whatever systemd/container tools your system already has.

## Using it with an AI

```text
Here is a system health code from a tool called syscode:

SYS1-FED43-D23-M29-SW02-LD006-S0-K0-U038-J12-C0/13-NT+

Here are the definitions:
<paste the contents of CODES.md>

Tell me what is fine, what to watch, and what needs action.
```

Not near an AI? `syscode --explain` gives you the same verdict as a plain
table.

## License

MIT
