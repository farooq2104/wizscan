# wizscan

A `wizcli` wrapper shell function for scanning directories and container images, with clean human-readable output.

## Requirements

- [`wizcli`](https://docs.wiz.io/wiz-docs/docs/wizcli-overview) installed and in `PATH`
- Python 3 (for output parsing)
- Docker (only for `image` mode — tested with Colima)

## Installation

Copy the contents of [`wizscan.sh`](./wizscan.sh) into your `~/.zshrc` (or `~/.bashrc`), then reload:

```bash
source ~/.zshrc
```

## Commands

| Command | Description |
|---|---|
| `wizscan dir` | Scan current directory |
| `wizscan dir /path/to/project` | Scan a specific directory |
| `wizscan image` | Scan `<current-dir>:local` image |
| `wizscan image foo:tag` | Scan a specific image tag |

### Flags

| Flag | Description |
|---|---|
| `--high` | Also show high vulnerabilities (hidden by default) |

The `--high` flag can be placed anywhere in the command:

```bash
wizscan dir --high
wizscan --high image foo:tag
```

## Output

```
Mode   : dir scan
Target : /Users/you/workspace/my-service
Name   : my-service

[ wizcli ] Connecting to Wiz
[ wizcli ] SUCCESS: Connected to Wiz
[ wizcli ] SUCCESS: Local scanning complete

Result             : FAIL — CRITICAL+ vulnerabilities found
Policy             : Default vulnerabilities policy (threshold: CRITICAL)
Libraries scanned  : 12
Critical           : 2

--- CRITICAL (policy violations) ---
Name                 Version    Fixed      Path
----------------------------------------------------------------------------------------------------
fast-xml-parser      4.2.7      4.5.4      /yarn.lock
lodash               4.17.15    4.17.21    /yarn.lock
```
