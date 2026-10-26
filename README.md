# utils

Small, single-file tools for a Linux dev box. Each lives in its own directory
with its own README.

| Tool | What it does |
| --- | --- |
| [who-killed](who-killed/) | Name the process that sent a signal to another process, including `SIGKILL`. |

## Install

```sh
pixi run install     # symlinks every tool onto ~/.local/bin
```

That task is one symlink per tool:

```sh
ln -sfn "$PWD/who-killed/who-killed" ~/.local/bin/who-killed
```

## Development

The repo is managed with [pixi](https://pixi.sh); `pixi.lock` pins the
environment.

| Task | What it does |
| --- | --- |
| `pixi run who-killed …` | run a tool straight from the checkout |
| `pixi run lint` | `ruff check` over the tools |
| `pixi run install` | symlink every tool onto your `PATH` |

Two things pixi deliberately does not cover:

- **The tools run under the system `python3`, not pixi's.** They are
  stdlib-only, and `who-killed` re-execs itself through `sudo`, where pixi's
  environment is not on the path. pixi supplies the tooling around them.
- **`bpftrace` and `systemd` stay system packages.** Neither is on
  conda-forge, and the probe runs them as root.

Adding a tool takes three entries: `extend-include` in `ruff.toml` (the tools
have no `.py` extension, so Ruff cannot discover them on its own), the
`who-killed`-style tasks in `pixi.toml`, and the table above.

## Licence

MIT, unless a tool's README says otherwise.
