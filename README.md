# utils

Small, single-file tools for a Linux dev box. Each lives in its own directory
with its own README.

| Tool | What it does |
| --- | --- |
| [who-killed](who-killed/) | Name the process that sent a signal to another process, including `SIGKILL`. |

## Install

Each tool is a standalone executable. Symlink it onto your `PATH`:

```sh
ln -s "$PWD/who-killed/who-killed" ~/.local/bin/who-killed
```

## Licence

MIT, unless a tool's README says otherwise.
