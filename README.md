# wizscan

A `wizcli` wrapper shell function for scanning directories and container images, with clean human-readable output.

## Requirements

- [`wizcli`](https://docs.wiz.io/wiz-docs/docs/wizcli-overview) installed and in `PATH`
- Python 3 (for output parsing)
- Docker (only for `image` mode — tested with Colima)

## Installation

Copy the function below into your `~/.zshrc` (or `~/.bashrc`), then reload:

```bash
source ~/.zshrc
```

## Usage

```bash
wizscan dir                      # scan current directory
wizscan dir /path/to/project     # scan a specific directory
wizscan image                    # scan <current-dir>:local image
wizscan image foo:tag            # scan a specific image tag
```

Add `--high` anywhere to also show high vulnerabilities (hidden by default):

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
[ wizcli ] SUCCESS: ...

Result             : FAIL — CRITICAL+ vulnerabilities found
Policy             : Default Policy (threshold: CRITICAL)
Libraries scanned  : 12
Critical           : 2

--- CRITICAL (policy violations) ---
Name                 Version    Fixed      Path
----------------------------------------------------------------------------------------------------
fast-xml-parser      4.2.7      4.5.4      /yarn.lock
lodash               4.17.15    4.17.21    /yarn.lock
```

## Function

```zsh
# --- Wiz Scan ---
# Usage:
#   wizscan dir                    → scan current directory
#   wizscan dir /path/to/dir       → scan a custom directory
#   wizscan image                  → scan <current-dir>:local
#   wizscan image foo:tag          → scan explicit image tag
#   Add --high anywhere to also show high vulnerabilities
wizscan() {
  local target name wiz_cmd
  local proj="$(basename $(pwd))"
  local show_high=0
  local args=()

  # Strip --high flag from args
  for arg in "$@"; do
    if [[ "$arg" == "--high" ]]; then
      show_high=1
    else
      args+=("$arg")
    fi
  done

  local mode="${args[1]:-}"

  # Check wizcli is installed
  if ! command -v wizcli &>/dev/null; then
    echo "ERROR: wizcli is not installed or not in PATH."
    return 1
  fi

  if [[ -z "$mode" ]]; then
    echo "Usage: wizscan [dir|image] [path|image:tag] [--high]"
    return 1
  fi

  case "$mode" in
    dir)
      target="${args[2]:-.}"
      if [[ ! -d "$target" ]]; then
        echo "ERROR: '$target' is not a valid directory."
        return 1
      fi
      name="$(basename $(realpath $target))"
      wiz_cmd="wizcli scan dir $target --name=\"$name\" --stdout=json"
      echo "Mode   : dir scan"
      echo "Target : $(realpath $target)"
      ;;
    image)
      # Check Docker is running
      if ! docker info &>/dev/null; then
        echo "ERROR: Docker daemon is not running."
        echo "Start it with: colima start"
        return 1
      fi
      target="${args[2]:-${proj}:local}"
      name="${proj}"
      if ! docker image inspect "$target" &>/dev/null; then
        echo "ERROR: Image '$target' not found locally."
        echo "Build it first with: docker build --no-cache -t $target ."
        return 1
      fi
      wiz_cmd="wizcli scan container-image $target --name=\"$name\" --stdout=json"
      echo "Mode   : image scan"
      echo "Image  : $target"
      ;;
    *)
      echo "ERROR: Unknown mode '$mode'. Usage: wizscan [dir|image] [path|image:tag] [--high]"
      return 1
      ;;
  esac
  echo "Name   : $name"
  echo ""

  eval "$wiz_cmd" 2>&1 | tee /tmp/wizscan_raw.txt | while IFS= read -r line; do
    [[ "$line" =~ ^(Connecting|Initiating|Scanning|Uploading|Waiting|SUCCESS:|ERROR:|FAILED:) ]] || continue
    echo "[ wizcli ] $line"
  done
  echo ""

  SHOW_HIGH=$show_high python3 -c "
import json, sys, os

show_high = os.environ.get('SHOW_HIGH') == '1'
raw = open('/tmp/wizscan_raw.txt').read()

error_lines = [l for l in raw.splitlines() if l.startswith('ERROR:') or l.startswith('FAILED:')]
json_line = next((l for l in raw.splitlines() if l.strip().startswith('{')), None)

if not json_line:
    print('Scan failed — no results returned by wizcli.')
    if error_lines:
        print()
        for l in error_lines:
            print(f'  {l}')
    print()
    print('Full output saved to /tmp/wizscan_raw.txt')
    sys.exit(1)

try:
    data, _ = json.JSONDecoder().raw_decode(json_line.strip())
except json.JSONDecodeError as e:
    print(f'ERROR: Failed to parse wizcli JSON output: {e}')
    print('Full output saved to /tmp/wizscan_raw.txt')
    sys.exit(1)

result = data.get('result', {}) or {}
libs = result.get('libraries') or []
scan_state = data.get('status', {}).get('state', 'UNKNOWN')

policies = data.get('policies') or []
vuln_policy = next((p for p in policies if p.get('type') == 'VULNERABILITIES'), None)
policy_threshold = vuln_policy.get('params', {}).get('severity', 'UNKNOWN') if vuln_policy else 'UNKNOWN'
policy_name = vuln_policy.get('name', '') if vuln_policy else ''

analytics = result.get('analytics', {}).get('vulnerabilities', {})
cnt_critical = analytics.get('criticalCount', 0)
cnt_high     = analytics.get('highCount', 0)
cnt_medium   = analytics.get('mediumCount', 0)
cnt_low      = analytics.get('lowCount', 0)

rows = [
    (r.get('name',''), r.get('version',''), v.get('severity',''), v.get('fixedVersion',''), r.get('path',''))
    for r in libs
    for v in r.get('vulnerabilities', [])
]

if scan_state == 'ERROR':
    print('Result             : SCAN ERROR — wizcli reported a scan failure')
    print('Full output saved to /tmp/wizscan_raw.txt')
    sys.exit(1)
elif scan_state != 'SUCCESS':
    result_label = f'SCAN ERROR (state={scan_state})'
elif rows:
    result_label = f'FAIL — {policy_threshold}+ vulnerabilities found'
else:
    result_label = f'PASS — no {policy_threshold}+ vulnerabilities found'

print(f'Result             : {result_label}')
print(f'Policy             : {policy_name} (threshold: {policy_threshold})')
print(f'Libraries scanned  : {len(libs)}')
if cnt_critical: print(f'Critical           : {cnt_critical}')
if show_high and cnt_high: print(f'High               : {cnt_high}')
if cnt_medium:   print(f'Medium             : {cnt_medium}')
if cnt_low:      print(f'Low                : {cnt_low}')
print()

if not rows:
    sys.exit(0)

print(f'--- CRITICAL (policy violations) ---')
print(f'%-20s %-10s %-10s %s' % ('Name','Version','Fixed','Path'))
print('-'*100)
for row in rows:
    name, version, severity, fixed, path = row
    print(f'%-20s %-10s %-10s %s' % (name, version, fixed, path))

if show_high:
    sbom = result.get('vulnerableSBOMArtifactsByNameVersion') or []
    high_rows = [
        (
            s.get('name',''),
            s.get('version',''),
            s.get('vulnerabilityFindings',{}).get('severities',{}).get('highCount', 0),
            s.get('vulnerabilityFindings',{}).get('fixedVersion',''),
            s.get('filePath',''),
        )
        for s in sbom
        if s.get('vulnerabilityFindings',{}).get('severities',{}).get('highCount', 0) > 0
    ]
    if high_rows:
        print()
        print(f'--- HIGH (informational) ---')
        print(f'%-20s %-10s %-6s %-10s %s' % ('Name','Version','Count','Fixed','Path'))
        print('-'*100)
        for row in high_rows:
            print(f'%-20s %-10s %-6s %-10s %s' % row)
"
}
```
