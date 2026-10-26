# who-killed

Names the process that sent a signal to another process — including `SIGKILL`,
which leaves no trace on the victim's side.

```
18:55:21 SIGNAL 15 -> chrome pid=3228318 | sender pkill sproc=pkill pid=3237890 uid=1000 ppid=3237873 pcomm=sh ppid2=3113688 pcomm2=delta-app | si_code=0
```

> `pkill` (3237890) sent SIGTERM to Chrome (3228318). Its parent is `sh`, and
> that shell's parent is `delta-app`.

## Why it exists

When an application disappears with no crash dump, no coredump and nothing in
journald, everything on the victim's side is blind:

- `SIGKILL` cannot be caught, so the victim logs nothing;
- `coredumpctl` records crashes, not kills;
- the kernel OOM killer logs, and its cgroup counters (`memory.events:
  oom_kill`) record it;
- `systemd-oomd` logs every kill it makes.

What is left is the sender's side. The kernel's `signal:signal_generate`
tracepoint fires inside the *sending* task, before delivery. This arms a probe
there and reads it back.

## Use

```sh
who-killed chrome          # arm for 1h (prompts for sudo)
who-killed --report        # who was it?
who-killed --status        # what is armed
who-killed --stop chrome   # stop early
```

Options: `--seconds N`, `--signals 9,15`, `--report --log FILE`, `--all`,
`--lines N`, `--replace`.

Arming needs root, because attaching kernel tracepoints does. `--report` and
`--status` do not. The tool re-execs itself through `sudo` using its own
absolute path, since `sudo`'s `secure_path` would not find a script in
`~/.local/bin`.

## What it records

Two tracepoints, filtered to processes named like the target:

| Probe | Fires | Records |
| --- | --- | --- |
| `signal:signal_generate` | any signal being enqueued | signal number, target, and the sender's process name, pid, uid, two levels of parent chain, and `si_code` |
| `sched:sched_process_exit` | a thread exiting | kills only: the signal that ended it |

The sender is printed twice on purpose. `comm` is the **thread's** name, so a
browser's own thread pool appears as `ThreadPoolSingl` and looks like a
stranger; `sproc` is the **process's** name via `group_leader->comm`, which is
what "did it signal itself?" needs.

`si_code` separates the cases: `0` is `SI_USER`, an ordinary `kill(2)` or
`pkill`; negative values are kernel-generated.

## Where things go

Logs land in `~/.local/state/who-killed/<target>-<UTC>.log`, owned by whoever
invoked `sudo`. Each probe is a transient systemd unit with `RuntimeMaxSec`, so
it stops itself.

## Requirements

- Linux with a BTF-enabled kernel and `bpftrace`
- systemd, for transient units via `systemd-run`
- Tested on Garuda/Arch, kernel 6.x, bpftrace 0.27

Both are system packages. They stay that way on purpose: conda-forge has no
`bpftrace`, and the probe runs it as root, outside any pixi environment.

## Limits

- Matches the 15-byte process name (`comm`), not the command line.
- If a kernel field it needs is missing, it falls back to a probe that has only
  thread names, and says so in the log.
- A window ended by `RuntimeMaxSec` shows in systemd as `failed (timeout)`;
  `--status` reports it as `finished (window elapsed)`.
- One probe per target: a second arm is refused unless you pass `--replace`.
