# syscode — code definitions (v1 / SYS1)

`syscode` prints a single line summarizing the health of a Linux machine.
Give the code together with this document to any AI assistant and it can
interpret it: what is fine, what to watch, and what needs action.

## Format

```
SYS1-SEG-SEG-SEG...
```

`SYS1` marks format version 1. Segments always appear in this fixed order:

| # | Segment | Example  | What it is |
|---|---------|----------|------------|
| 1 | OS      | `FED43`  | distro + major version |
| 2 | DISK    | `D23`    | worst mounted filesystem, used % |
| 3 | MEM     | `M29`    | RAM used % |
| 4 | SWAP    | `SW02`   | swap used % (or `SWX` = none) |
| 5 | LOAD    | `LD006`  | 1-min load average as % of CPU cores |
| 6 | SVC     | `S0`     | failed systemd units |
| 7 | REB     | `K0`     | reboot pending? (newer kernel installed) |
| 8 | UP      | `U038`   | uptime in whole days |
| 9 | JRN     | `J12`    | journal error lines this boot |
| 10 | CT     | `C0/13`  | unhealthy / running containers |
| 11 | NET    | `NT+`    | internet reachable |

## Segment definitions and thresholds

### OS — `FED43`
First 3 letters of the distro ID (uppercased) + major version.
`FED43` = Fedora 43, `UBU24` = Ubuntu 24.x, `DEB12` = Debian 12,
`ARC00` = Arch (rolling). Informational, never a warning.

### DISK — `D23`
Used percentage of the fullest mounted real filesystem
(`/`, `/boot`, `/home`, `/var`, `/opt`, `/srv`, `/mnt/*`, `/media/*`).
tmpfs, EFI, squashfs and overlay mounts are excluded.

- `D00-D79` OK
- `D80-D89` WATCH — plan a cleanup
- `D90-D100` ALERT — act now; full disks cause data loss and service failures

### MEM — `M29`
RAM used %, computed as `(total - available) / total`. "Available" already
counts reclaimable cache, so this is the honest number.

- `M00-M79` OK
- `M80-M92` WATCH
- `M93-M100` ALERT — OOM killer territory

`MNA` = unknown.

### SWAP — `SW02` / `SWX`
Swap used %. `SWX` = no swap configured (normal on many desktops; the
tradeoff is OOM kills instead of swapping).

- `SW00-SW49` OK — some swap use is normal, not a problem by itself
- `SW50-SW79` WATCH
- `SW80+` ALERT

### LOAD — `LD006`
1-minute load average expressed as a percentage of CPU cores
(e.g. load 0.26 on 4 cores = `LD006`; load 0.26 on 1 core = `LD026`).

- `LD000-LD069` OK
- `LD070-LD100` WATCH — sustained saturation
- `LD101+` ALERT — more demand than cores

Single sample, not an average. Short spikes are normal.

### SVC — `S0`
Count of failed systemd units since boot (`systemctl --failed`).

- `S0` OK
- `S1-S2` WATCH — the follow-up question is "which unit?"
- `S3+` ALERT

`SNA` = systemd not present/usable.

### REB — `K0` / `K1`
- `K0` running kernel is the newest installed one
- `K1` a newer kernel is installed but not booted — reboot pending (WATCH)
- `KNA` unknown (non-rpm distro)

### UP — `U038`
Uptime in whole days. Informational on its own; combined with `K1` it tells
you how long a reboot has been overdue. `U38` style values above `U099`
just use more digits (`U190` = 190 days).

### JRN — `J12`
Error-severity journal lines since boot (`journalctl -p 3 -b`).
Noisy hardware/drivers can inflate this; trend matters more than raw count.

- `J0-J19` OK
- `J20-J100` WATCH — worth a look at what is repeating
- `J101+` ALERT

Counts reflect the invoking user's journal privileges; an unprivileged run
may undercount.

### CT — `C0/13` / `CNA`
Unhealthy-or-restarting containers / total running containers
(docker or podman, whichever is present).

- `C0/n` OK
- `Cx/n` with x>0 WATCH — container health checks are failing
- `CNA` no container runtime, or the invoking user lacks permission

Exited containers are NOT counted (intentionally stopped containers are normal).

### NET — `NT+` / `NT-` / `NTX`
One ICMP ping to 1.1.1.1 (2s timeout).

- `NT+` reachable
- `NT-` no reply — offline, or ICMP is blocked (WATCH, not always a real problem)
- `NTX` check skipped (`--no-net`)

## Worked example

```
SYS1-FED43-D87-M41-SW00-LD012-S1-K1-U190-J44-C2/13-NT+
```

Interpretation: Fedora 43. One filesystem at 87% (WATCH, clean it soon).
RAM healthy at 41%, swap idle, load light. One failed unit (find it with
`systemctl --failed`). A kernel update is waiting for a reboot, and the box
has been up 190 days — reboot when convenient. 44 journal errors this boot
(WATCH, see what repeats: `journalctl -p 3 -b`). 2 of 13 running containers
are unhealthy (WATCH). Internet is fine. Nothing is on fire; three things
deserve a look.

## Notes for AI consumers

- All checks are read-only. syscode never changes system state.
- Exit code: 0 = clean, 1 = at least one WATCH, 2 = at least one ALERT.
- If a code contains segments you do not recognize (newer format version),
  say so instead of guessing.
