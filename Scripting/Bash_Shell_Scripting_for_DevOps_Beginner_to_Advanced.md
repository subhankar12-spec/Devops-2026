# Bash & Shell Scripting for DevOps Engineers: Zero to Production Automation

**Self-contained handbook | Beginner → Intermediate → Advanced → Production**  
**Audience:** Linux administrators, DevOps engineers, SREs, cloud and CI/CD engineers.  
**Environment:** Linux (Debian/Ubuntu/RHEL) or WSL2 with Bash 5.x.  
**Learning method:** Concept → command → output → why it matters → hands-on lab.

> **Safety:** Practice in a disposable directory or VM. Never run destructive commands against production; inspect targets before `rm`, `find -delete`, `kubectl delete`, or cloud mutation. Avoid `curl | bash`, never commit secrets, and never print credentials in CI logs. Commands labeled **read-only** are safer starting points.

## Table of contents

1. [Part I — Shell fundamentals](#part-i--shell-fundamentals)
2. [Part II — Practical Bash programming](#part-ii--practical-bash-programming)
3. [Part III — Robust production scripting](#part-iii--robust-production-scripting)
4. [Part IV — DevOps integrations](#part-iv--devops-integrations)
5. [Part V — Guided labs](#part-v--guided-labs)
6. [Part VI — Capstone project](#part-vi--capstone-project)
7. [Part VII — Reference, debugging, assessment](#part-vii--reference-debugging-assessment)

### Suggested 6-week study plan

| Week | Focus | Deliverable |
|---|---|---|
| 1 | Shell, quoting, variables, paths, exit codes | Portable system inventory |
| 2 | Conditions, loops, functions, arrays | Host health report |
| 3 | Streams, text tools, regex, JSON | Structured log parser |
| 4 | Strict mode, traps, retries, locking, security | Reliable backup tool |
| 5 | Git, GitHub Actions, Docker, Kubernetes, AWS | Read-only pipeline checks |
| 6 | Testing, ShellCheck, capstone | Tested release verification CLI |

### Setup

```bash
bash --version
command -v bash git awk sed grep jq curl shellcheck bats || true
mkdir -p "$HOME/bash-devops-lab"
cd "$HOME/bash-devops-lab"
printf '#!/usr/bin/env bash\nprintf "Hello, DevOps!\n"\n' > hello.sh
bash hello.sh
chmod +x hello.sh
./hello.sh
```

Install optional tools on Debian/Ubuntu: `sudo apt-get update && sudo apt-get install -y shellcheck jq bats` (requires privileges and network). Prefer distro packages or approved tooling in managed environments.

# Part I — Shell fundamentals

## 1. Terminal, shell, Bash, and script

A **terminal** is an interface; a **shell** interprets commands; **Bash** is one shell implementation. A script is a file containing commands. Other shells include `sh` (often dash), zsh, and fish. **Bash features are not necessarily POSIX sh features**: arrays, `[[ ... ]]`, `=~`, and `pipefail` require Bash (or compatible shells). Use `#!/usr/bin/env bash` for a Bash script; use `#!/bin/sh` only when writing POSIX-compatible syntax.

```bash
printf '%s\n' "$SHELL"  # configured login shell, not necessarily current interpreter
printf '%s\n' "$BASH_VERSION"
ps -p $$ -o comm=
type -a printf ls
command -v git
help test
man bash
```

`type` distinguishes builtins, aliases, functions and executables. `command -v` is useful for dependency checks. `source file.sh` executes in the **current shell** (and can change its variables and directory); `bash file.sh` launches a child process.

## 2. Commands, arguments, options, and exit codes

A command has a name and arguments. Exit code **0** means success; nonzero generally means failure (exact meanings depend on command).

```bash
ls -lah /tmp
true; echo "$?"     # 0
false; echo "$?"    # 1
if grep -q 'root' /etc/passwd; then
  echo 'Found'
fi
```

`$?` holds the exit status of the **most recent** command, so save it immediately if needed. `cmd1 && cmd2` runs the second on success; `cmd1 || cmd2` runs it on failure. Avoid relying on an exit code after another command has run.

## 3. Paths, working directory, permissions

```bash
pwd
cd /tmp
cd -
ls -ld .
stat /etc/passwd
realpath .
readlink -f /etc/hosts
```

Absolute paths start with `/`; relative paths are resolved from the current directory. `.` is current directory, `..` is parent, `~` is home. `chmod 755 script.sh` sets permissions; `chmod +x script.sh` adds executable permission. For sensitive files, prefer `chmod 600`. `umask 077` makes newly created files/directories private by default (subject to tool behavior).

## 4. Variables, environment, and scope

```bash
service='catalogue'       # no spaces around =
replicas=3
printf 'Service: %s, replicas: %s\n' "$service" "$replicas"
export ENVIRONMENT=dev    # available to child processes
readonly REGION=us-east-1
unset replicas
```

Shell variables are strings by default. `export` makes a variable available in a child process. `local` limits variables to a function. Environment variables inherited from a parent can be overridden. Do not store secrets in shell history or print them with `env` in shared logs.

## 5. Quoting and expansion — the #1 Bash skill

```bash
name='commerce frontend'
printf '%s\n' "$name"   # one argument
printf '%s\n' '$name'   # literal $name
printf '%s\n' "${name}" # explicit boundary
file='report with spaces.txt'
printf 'hello\n' > "$file"
cat -- "$file"
```

**Double-quote variable expansions** (`"$var"`, `"${array[@]}"`) unless you deliberately want splitting or globbing. Unquoted expansion can cause whitespace splitting and filename expansion. Single quotes prevent expansion; double quotes permit variables and command substitution. `--` terminates options for supporting commands, preventing filenames starting with `-` from being interpreted as flags.

```bash
prefix='release'
printf '%s\n' "${prefix}_id"    # release_id
printf '%s\n' "${undefined:-fallback}" # fallback if unset or empty
printf '%s\n' "${undefined-default}"  # fallback only if unset
```

## 6. Command substitution, arithmetic, and braces

```bash
current_commit=$(git rev-parse --short HEAD 2>/dev/null || printf 'unknown')
count=3
((count += 1))
printf '%s\n' "$count"           # 4
printf '%s\n' "$((count * 2))"  # 8
printf '%s\n' {dev,qa,production}
```

`$(...)` captures stdout, stripping trailing newlines. `$((...))` performs integer arithmetic; Bash does not support native floating-point arithmetic. Brace expansion happens before variable expansion and is not a loop.

## 7. Redirection, stdin/stdout/stderr, pipes

```bash
printf 'hello\n' > output.txt     # overwrite
printf 'world\n' >> output.txt    # append
cat < output.txt
ls /not-a-path 2> error.log
ls /tmp > listing.log 2>&1
printf 'a\nb\n' | grep b
```

File descriptor 0 = stdin, 1 = stdout, 2 = stderr. `2>&1` redirects stderr to wherever stdout points **at that point in the command**. `|` connects stdout of one command to stdin of the next. `tee` writes to both file and stdout; `tee -a` appends.

## 8. Wildcards and globbing

```bash
printf '%s\n' *.sh
printf '%s\n' file?.txt
```

`*` matches any sequence and `?` one character in a filename. Globbing is **not regex**. Unmatched globs may remain literal unless `nullglob`/`failglob` is configured. Quote filenames after glob expansion where appropriate.

# Part II — Practical Bash programming

## 9. Tests, conditions, and Boolean logic

```bash
file='config.yaml'
if [[ -f "$file" ]]; then
  printf 'file exists\n'
elif [[ -d "$file" ]]; then
  printf 'directory exists\n'
else
  printf 'missing\n'
fi

if [[ "$ENVIRONMENT" == 'production' && "${APPROVED:-false}" == 'true' ]]; then
  echo 'approval present'
fi
```

Use `[[ ... ]]` in Bash: it avoids many quoting and glob pitfalls of `[ ... ]`. Common operators: `-f` regular file, `-d` directory, `-e` exists, `-r` readable, `-w` writable, `-x` executable, `-z` empty string, `-n` nonempty, `==`, `!=`, `<`, `>`. For integers inside `[[ ]]`, use `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`, or use arithmetic `(( ... ))`. Beware `((i++))` can return status 1 when the expression evaluates to 0, relevant with `set -e`.

```bash
case "${1:-}" in
  dev|qa) echo 'non-production' ;;
  production) echo 'approval required' ;;
  *) echo 'unknown environment' >&2; exit 2 ;;
esac
```

## 10. Loops and iteration

```bash
for env in dev qa production; do
  printf 'Checking %s\n' "$env"
done

for ((i=1; i<=3; i++)); do
  printf 'attempt %d\n' "$i"
done

attempt=0
while ((attempt < 3)); do
  ((attempt += 1))
  printf 'try %d\n' "$attempt"
done
```

`break` exits a loop; `continue` skips to the next iteration. `until command; do ...; done` loops until a command succeeds. Avoid `for file in $(ls ...)`: word splitting breaks names with spaces. Prefer `for file in ./*.log` or `find ... -print0` with `read -d ''`.

## 11. Reading files safely

```bash
while IFS= read -r line || [[ -n "$line" ]]; do
  printf 'Line: %s\n' "$line"
done < input.txt
```

`IFS=` preserves leading/trailing whitespace, `-r` preserves backslashes, and the extra condition processes a final line without a newline. `read -r -p 'Environment: ' env` prompts interactively. Avoid parsing CSV with naive `IFS=, read` if values can contain quoted commas; use a CSV parser.

## 12. Arrays and associative arrays

```bash
services=(frontend catalogue orders cart)
printf '%s\n' "${services[0]}"      # frontend
printf '%s\n' "${#services[@]}"     # 4
for service in "${services[@]}"; do echo "$service"; done

declare -A image_repo=([frontend]=commerce-frontend [orders]=commerce-orders)
printf '%s\n' "${image_repo[orders]}"
```

Indexed arrays store ordered values. Associative arrays map strings to strings (Bash 4+). `"${array[@]}"` preserves one argument per element; `"${array[*]}"` joins elements into one argument. Check membership carefully if unset keys are possible.

## 13. Functions, parameters, return statuses

```bash
log() {
  local level=$1
  shift
  printf '[%s] %s\n' "$level" "$*" >&2
}

check_file() {
  local path=$1
  [[ -f "$path" ]] || { log ERROR "Missing: $path"; return 1; }
  return 0
}

if check_file /etc/passwd; then log INFO 'Ready'; fi
```

Bash functions return an **exit status** (0–255), not arbitrary strings. For data, print to stdout and capture with `value=$(function_name)`; print diagnostics to stderr. `$1`, `$2` are positional parameters; `$#` is count; `"$@"` preserves all positional arguments. `shift` discards the first positional argument. Use `local` for variables inside functions.

## 14. Parameters, defaults, and argument parsing

```bash
script_name=${0##*/}
env=${1:-dev}
: "${REQUIRED_TOKEN:?REQUIRED_TOKEN must be set}"  # exits if unset/empty
```

The `:` command does nothing; it is useful with parameter expansion. `${var:=default}` sets a default when unset/empty. `${var:+alternate}` yields alternate if set/nonempty. `${var#prefix}` and `${var%suffix}` remove patterns. `${var//old/new}` replaces all matches.

```bash
usage() { printf 'Usage: %s [-e env] [-n]\n' "$0"; }
env=dev; dry_run=false
while getopts ':e:nh' opt; do
  case "$opt" in
    e) env=$OPTARG ;;
    n) dry_run=true ;;
    h) usage; exit 0 ;;
    :) echo "Missing value for -$OPTARG" >&2; exit 2 ;;
    \?) echo "Unknown option: -$OPTARG" >&2; exit 2 ;;
  esac
done
shift "$((OPTIND - 1))"
```

`getopts` handles short options. For long flags (`--environment`) use a carefully written `while/case` parser or a dedicated CLI language/library. Always validate values after parsing.

## 15. String operations

```bash
image='ghcr.io/acme/frontend:1.2.3'
echo "${#image}"              # string length
echo "${image##*/}"           # frontend:1.2.3
echo "${image%:*}"            # without last :tag
echo "${image:0:7}"           # substring
echo "${image/frontend/api}" # first replacement
```

Shell pattern operations use glob-style patterns, not full regex. For complex structured strings, prefer purpose-built parsers.

## 16. Text processing: grep, sed, awk, cut, sort, uniq

```bash
printf 'INFO start\nERROR timeout\n' > app.log
grep -n 'ERROR' app.log
grep -E 'ERROR|WARN' app.log
sed -n '1,10p' app.log
awk '{print $1}' app.log
printf 'a\nb\na\n' | sort | uniq -c
printf 'frontend:8080\n' | cut -d: -f2
```

`grep -q` is a silent existence check; `grep -F` searches literal text; `grep -E` uses extended regex. `sed` edits streams, `awk` works with records and fields. Avoid using regex to parse JSON, YAML, XML, or general HTML.

## 17. Regular expressions

```bash
sha='0123456789abcdef0123456789abcdef01234567'
if [[ "$sha" =~ ^[0-9a-f]{40}$ ]]; then
  echo 'looks like a Git SHA-1 object ID'
fi
```

`=~` uses extended regex in `[[ ]]`. Anchors `^` and `$` match start/end. `[0-9]` character class; `{40}` exact repetition. **A format check is not proof that a commit exists or is trusted.** For Git object validation use `git cat-file -e "$sha^{commit}"` after controlling the repository context.

## 18. Files, directories, symlinks, temporary files

```bash
mkdir -p reports
printf 'ok\n' > reports/status.txt
cp -- reports/status.txt reports/status.copy
mv -- reports/status.copy reports/status.old
[[ -L /etc/localtime ]] && readlink /etc/localtime

tmp=$(mktemp)
trap 'rm -f -- "$tmp"' EXIT
printf 'private\n' > "$tmp"
```

`mktemp` avoids predictable temporary filenames. A quoted EXIT trap can refer to variables at execution time; ensure variables still exist and are safe. For robust scripts, use a dedicated cleanup function and a private temporary directory.

## 19. Process management and signals

```bash
ps -ef | head
pgrep -af sshd || true
sleep 2 &
pid=$!
wait "$pid"
```

`$!` is the PID of the last background job. `kill -TERM PID` requests graceful termination; `kill -KILL` cannot be trapped and should be a last resort. `trap` handles EXIT, INT, TERM; understand that EXIT traps do not run on SIGKILL or abrupt host termination.

## 20. Here-documents and here-strings

```bash
cat > config.json <<'JSON'
{"service":"frontend","replicas":2}
JSON

read -r word <<< 'hello'
echo "$word"
```

Quoting the heredoc delimiter (`<<'JSON'`) prevents variable and command expansion. Unquoted delimiters expand shell variables and commands inside the document.

# Part III — Robust production scripting

## 21. Strict mode: helpful, not magic

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

`-e` exits on many failures **but has important exceptions** (conditions, `&&`, `||`, subshells, pipelines, functions invoked in conditions). `-u` rejects unset variable expansions. `pipefail` makes pipeline failure visible when a non-last command fails. `-E` inherits ERR traps into functions/subshells where supported. Changing global `IFS` can surprise existing scripts; many teams instead use targeted `IFS= read -r` and quoted expansions. Strict mode does **not** replace explicit error handling.

```bash
set -o pipefail
if ! grep -q 'ready' status.txt; then
  echo 'not ready (or grep error)' >&2
fi
```

When missing data and command failure must be distinguished, capture and inspect `grep` status 1 vs 2 rather than using `!` alone.

## 22. Logging, errors, and traps

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
log() { printf '%s [%s] %s\n' "$(date -u +%FT%TZ)" "$1" "$2" >&2; }
die() { log ERROR "$1"; exit "${2:-1}"; }
cleanup() { [[ -z "${workdir:-}" ]] || rm -rf -- "$workdir"; }
trap cleanup EXIT
trap 'log ERROR "Failed at line $LINENO"' ERR
workdir=$(mktemp -d)
log INFO 'Started'
```

The `ERR` trap reports failures but is subject to the same conditional-context subtleties as `set -e`. Avoid logging command arguments that may contain credentials. For production cleanup, verify that a path is a private temporary directory before deletion.

## 23. Safe input handling and injection prevention

```bash
user_input='my file; echo hacked'
printf '%s\n' "$user_input"  # data, not executable code
cat -- "$user_input" 2>/dev/null || true
```

Never use `eval "$user_input"` or `bash -c "$user_input"` for untrusted data. Prefer arrays to build commands:

```bash
cmd=(curl --fail --silent --show-error --connect-timeout 5)
cmd+=(--max-time 20 'https://example.com/health')
"${cmd[@]}"
```

Validate allowlists for environment names and file paths; protect credentials via approved secret stores, short-lived OIDC credentials, and least privilege. Avoid `set -x` when secrets may appear.

## 24. Retries, timeouts, and backoff

```bash
retry() {
  local max=$1 delay=1 attempt=1
  shift
  while true; do
    if "$@"; then return 0; fi
    if ((attempt >= max)); then return 1; fi
    sleep "$delay"
    ((attempt += 1))
    ((delay *= 2))
  done
}
retry 3 curl --fail --silent --show-error --max-time 10 https://example.com/health
```

Retry **transient and safe/idempotent operations**. Blindly retrying writes (creating releases, payments, deployment triggers) can duplicate side effects. Add jitter in distributed systems to avoid synchronized retries.

## 25. Idempotency and atomic writes

An idempotent operation can be repeated without undesired additional effects. `mkdir -p` is generally idempotent; `echo line >> file` is not. Avoid partially written configuration by writing a temporary file in the same directory and atomically renaming it on the same filesystem.

```bash
config_dir=$(mktemp -d)
tmp=$(mktemp "$config_dir/.config.XXXXXX")
printf 'enabled=true\n' > "$tmp"
mv -- "$tmp" "$config_dir/config.ini"
cat "$config_dir/config.ini"
rm -rf -- "$config_dir"  # only this newly created lab directory
```

For durable writes, filesystem and fsync behavior matter; atomic rename does not automatically guarantee crash durability.

## 26. Concurrency and locks

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
exec 9>"${TMPDIR:-/tmp}/my-devops-task.lock"
if ! flock -n 9; then
  echo 'Another instance is running' >&2
  exit 75
fi
printf 'Exclusive task started\n'
```

`flock` is Linux-specific and advisory; all cooperating processes must use the same lock. Lock-file location and permissions matter in multi-user systems. `flock` is often safer than homemade PID-file logic.

## 27. Subshells, sourcing, and pipelines

```bash
x=original
( x=changed; echo "$x" )
echo "$x" # original
```

Parentheses run in a subshell. Braces `{ ...; }` group commands in the current shell. A `while read` loop fed through a pipe may run in a subshell and lose variable changes; prefer process substitution when needed:

```bash
count=0
while IFS= read -r line; do ((count += 1)); done < <(printf 'a\nb\n')
echo "$count"  # 2
```

`< <(...)` is process substitution and Bash-specific; errors from the producer need explicit consideration.

## 28. Performance and portability

Use Bash for orchestration and small data transformations; use Python for complicated data models, APIs, rich testing, or extensive parsing. Avoid launching thousands of subprocesses in tight loops; use `awk` for column processing and `jq` for JSON. Prefer `printf` over implementation-dependent `echo -e`. Use `#!/usr/bin/env bash` only when PATH is trustworthy. If POSIX portability is required, avoid Bash-only constructs and test with `dash`.

## 29. Testing, linting, formatting

```bash
bash -n script.sh             # syntax check
shellcheck script.sh         # static analysis
bash -x script.sh            # execution trace: may expose secrets
```

Use ShellCheck for quoting and common pitfalls. `shfmt` is a formatter if installed. Bats provides shell tests:

```bash
# test/example.bats
@test "printf produces expected output" {
  run bash -c 'printf hello'
  [ "$status" -eq 0 ]
  [ "$output" = hello ]
}
```

Run with `bats test/`. Test both successful and failure paths, weird filenames, missing dependencies, empty inputs, timeouts, and permission errors. Mock cloud commands instead of calling live infrastructure in unit tests.

## 30. Security checklist

- Quote expansions and arrays; use `--` to end option parsing when supported.
- Validate environment names, digests, IDs, and paths against allowlists.
- Use least-privilege credentials; do not echo tokens or enable debug tracing around secrets.
- Use `mktemp`, restrictive permissions, and cleanup traps.
- Use timeouts on network calls; retry only safe transient operations.
- Avoid `eval`, untrusted sourcing, `curl | bash`, and arbitrary executable content from PRs.
- Keep release artifacts immutable by digest; verify signatures/provenance where applicable.
- Require explicit approval for production mutations and provide dry-run modes.

# Part IV — DevOps integrations

## 31. Linux operations and systemd

```bash
uname -a
cat /etc/os-release
df -h
free -h
ss -tulpen 2>/dev/null || ss -tuln
systemctl is-active ssh 2>/dev/null || true
journalctl -u ssh --since '1 hour ago' --no-pager 2>/dev/null | tail -30
```

These are primarily read-only diagnostics. Service names vary by distro (`ssh` vs `sshd`). `journalctl` may require elevated permissions. Use `systemctl show -p ActiveState --value service` when scripting machine-readable output.

## 32. Network diagnostics

```bash
ip -br addr
ip route
getent hosts example.com
curl --fail --silent --show-error --max-time 10 -I https://example.com
```

`curl -f` returns nonzero for HTTP 400+ (with exceptions depending on authentication/protocol). Use `--connect-timeout`, `--max-time`, and TLS verification by default. Do not suppress TLS errors with `-k` in production.

## 33. Git automation

```bash
git rev-parse --show-toplevel
git status --porcelain
git branch --show-current
git rev-parse HEAD
git log -1 --format='%H %s'
git diff --name-only HEAD~1 HEAD
```

`git status --porcelain` is designed for scripting. Verify repository identity and branch before changing anything. For release automation, prefer immutable commit SHAs over moving branch names.

## 34. GitHub Actions and shell execution

```yaml
name: Shell quality
on: [push, pull_request]
permissions:
  contents: read
jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check scripts
        shell: bash
        run: |
          sudo apt-get update
          sudo apt-get install -y shellcheck
          find scripts -type f -name '*.sh' -print0 | xargs -0 -r shellcheck
```

For security-sensitive production workflows, pin third-party actions to reviewed full commit SHAs and use minimal permissions. GitHub Actions `run:` defaults vary by platform/shell selection; specify `shell: bash` when Bash behavior is required. Write outputs to `$GITHUB_OUTPUT`, summaries to `$GITHUB_STEP_SUMMARY`; avoid legacy `::set-output`.

```bash
if [[ -n "${GITHUB_OUTPUT:-}" ]]; then
  printf 'release_id=%s\n' "$release_id" >> "$GITHUB_OUTPUT"
fi
```

Only emit validated single-line output values this way. Use documented multiline delimiters for multiline outputs, choosing delimiters that cannot collide with content.

## 35. Docker automation

```bash
docker version
docker ps --format '{{.Names}} {{.Status}}'
docker image ls --format '{{.Repository}}:{{.Tag}} {{.ID}}'
docker inspect --format '{{.State.Status}}' my-container
```

`docker inspect` returns JSON; use `docker inspect ... | jq` when appropriate. Docker socket access often confers host-level privilege: do not grant it casually to untrusted CI jobs. Use image digests for reproducible releases.

## 36. Kubernetes and kubectl

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get deployment -n default -o json | jq -r '.items[].metadata.name'
kubectl auth can-i get pods --all-namespaces
```

Always confirm context/namespace before a mutating command. Prefer `-o json` + `jq` rather than parsing human-oriented `kubectl get` tables. In CI use a narrowly scoped service account and read-only permissions when possible.

## 37. Helm and Argo CD

```bash
helm list -A
helm template demo ./charts/demo -f values-dev.yaml > /tmp/rendered.yaml
kubectl diff -f /tmp/rendered.yaml || status=$?
```

`kubectl diff` returns 1 when differences exist, 0 when none, and >1 on errors; handle these statuses deliberately. `helm template` renders locally and does not install resources. Argo CD CLI examples (requires configured access): `argocd app list`, `argocd app get myapp`, `argocd app history myapp`.

## 38. AWS CLI and cloud automation

```bash
aws sts get-caller-identity --output json
aws configure list
aws ec2 describe-vpcs --region us-east-1 --output json | jq -r '.Vpcs[].VpcId'
aws s3api list-buckets --query 'Buckets[].Name' --output text
```

These examples are read-only but may incur API usage depending on service and account. Use AWS profiles or OIDC roles, never hardcoded access keys. Validate account ID and region before changes. Use `--query` for AWS CLI's JMESPath queries; use `jq` for general JSON transformations.

## 39. Terraform and Ansible

```bash
terraform fmt -check -recursive
terraform validate
terraform plan -input=false -out=tfplan
ansible-inventory -i inventory.ini --list
ansible-playbook -i inventory.ini site.yml --syntax-check
```

`terraform plan` may refresh provider state and contact APIs; it does not normally apply changes, but may involve cost/API side effects. Avoid checking state files into Git: they can contain secrets. Use `ansible-playbook --check --diff` where supported, understanding that check mode is not a perfect simulation.

## 40. JSON, YAML, and release metadata

```bash
printf '%s\n' '{"component":"frontend","runId":42}' | jq -e '.component == "frontend" and (.runId | type == "number")'
```

`jq -e` sets an exit status based on the final output; it is useful for gates. Use `jq --arg name "$name"` rather than interpolating untrusted values into jq programs. YAML should be parsed with a real YAML parser (`yq` with a known implementation/version or Python/PyYAML), not `grep` and `sed`.

## 41. CI/CD design patterns

A reliable delivery script should separate **read-only verification** from **write operations**. Typical stages: parse inputs → validate context → verify immutable artifact → check policy → dry-run plan → explicit mutation → wait for observed evidence → emit structured outputs. Use source-bound provenance, bounded retries, timeouts, logs without secrets, and human approval for production. Avoid treating a successful build as proof of a healthy deployment.

# Part V — Guided labs

Each lab is intended to be run in your **own** `$HOME/bash-devops-lab` directory. Use a test VM/container, not production. Try the extension exercises before reading the hints.

## Lab 01 — Hello Bash, arguments, and exit status

**Goal:** Create an executable CLI accepting an environment.

```bash
cat > lab01.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
env=${1:-dev}
case "$env" in
  dev|qa|production) printf 'Environment: %s\n' "$env" ;;
  *) printf 'Invalid environment: %s\n' "$env" >&2; exit 2 ;;
esac
BASH
chmod +x lab01.sh
./lab01.sh qa
./lab01.sh unknown || echo "Rejected as expected"
```

**Check:** `qa` succeeds; `unknown` exits 2. **Extend:** add `--help` and a usage function.

## Lab 02 — Files and safe quoting

**Goal:** Handle paths with spaces.

```bash
mkdir -p 'lab02 output'
printf 'ok\n' > 'lab02 output/status report.txt'
file='lab02 output/status report.txt'
[[ -f "$file" ]] && cat -- "$file"
```

**Check:** output is `ok`. **Extend:** try a filename starting with `-`, and explain why `--` matters.

## Lab 03 — Arrays and loops

**Goal:** Print service names and positions.

```bash
services=(frontend catalogue orders cart)
for i in "${!services[@]}"; do
  printf '%d %s\n' "$i" "${services[$i]}"
done
```

**Check:** four lines, indices 0–3. **Extend:** build an associative array mapping components to repos.

## Lab 04 — Disk utilization threshold

**Goal:** Extract numeric disk usage and compare it to a threshold.

```bash
threshold=80
used=$(df -P / | awk 'NR==2 {gsub(/%/,"",$5); print $5}')
[[ "$used" =~ ^[0-9]+$ ]] || { echo 'Invalid disk metric' >&2; exit 1; }
if ((used >= threshold)); then
  printf 'WARNING: root disk at %s%%\n' "$used"
else
  printf 'OK: root disk at %s%%\n' "$used"
fi
```

**Check:** valid numeric percentage. **Extend:** accept `--threshold 75` and test invalid inputs.

## Lab 05 — Functions and structured logging

```bash
log() { printf '%s [%s] %s\n' "$(date -u +%FT%TZ)" "$1" "$2" >&2; }
require_command() { command -v "$1" >/dev/null 2>&1 || { log ERROR "Missing command: $1"; return 1; }; }
require_command git
log INFO 'Git available'
```

**Check:** missing dependencies fail with a useful message. **Extend:** return a distinct exit code for missing tools.

## Lab 06 — Read log lines and count errors

```bash
cat > lab06.log <<'LOG'
INFO startup
ERROR timeout
WARN retry
ERROR unavailable
LOG
count=0
while IFS= read -r line; do
  [[ "$line" == ERROR* ]] && ((count += 1))
done < lab06.log
printf 'Errors: %d\n' "$count"
```

**Check:** `Errors: 2`. **Extend:** count WARN messages and output JSON using `jq -n`.

## Lab 07 — Regex validation for release IDs

```bash
release_id=$(printf 'test-release' | sha256sum | awk '{print $1}')
if [[ "$release_id" =~ ^[a-f0-9]{64}$ ]]; then
  echo 'Valid SHA-256 format'
else
  exit 1
fi
```

**Check:** format is valid. **Extend:** distinguish valid format from actual artifact integrity.

## Lab 08 — JSON release facts with jq

```bash
cat > release.json <<'JSON'
{"component":"frontend","digest":"sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa","runId":101}
JSON
jq -e '.component == "frontend" and (.digest | startswith("sha256:")) and (.runId > 0)' release.json
jq -r '.component + " " + .digest' release.json
```

**Check:** validation returns true. **Extend:** enforce exact 64 hexadecimal characters after `sha256:` using jq's `test()`.

## Lab 09 — Retry an unreliable local command

```bash
attempt_file=$(mktemp)
printf '0\n' > "$attempt_file"
flaky() {
  local n
  n=$(cat "$attempt_file")
  ((n += 1))
  printf '%s\n' "$n" > "$attempt_file"
  ((n >= 3))
}
retry() {
  local i
  for ((i=1; i<=4; i++)); do
    if "$@"; then return 0; fi
    sleep 1
  done
  return 1
}
retry flaky
printf 'Succeeded after %s tries\n' "$(cat "$attempt_file")"
rm -f -- "$attempt_file"
```

**Check:** succeeds after 3 tries. **Extend:** exponential backoff and explicit timeout.

## Lab 10 — Safe temporary workspace and cleanup

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
workdir=$(mktemp -d)
cleanup() { rm -rf -- "$workdir"; }
trap cleanup EXIT
printf 'data\n' > "$workdir/result.txt"
cat "$workdir/result.txt"
```

**Check:** temporary directory is deleted on normal exit. **Extend:** test cleanup after an intentional error. Never replace `$workdir` with an unchecked external path.

## Lab 11 — Git repository inspection (read-only)

```bash
repo=$(git rev-parse --show-toplevel)
branch=$(git -C "$repo" branch --show-current)
sha=$(git -C "$repo" rev-parse HEAD)
printf 'repo=%s\nbranch=%s\nsha=%s\n' "$repo" "$branch" "$sha"
```

**Check:** run inside a Git repo. **Extend:** reject dirty working trees using `git status --porcelain`.

## Lab 12 — Docker/Kubernetes inspection (read-only)

```bash
if command -v docker >/dev/null 2>&1; then
  docker ps --format '{{.Names}} {{.Status}}'
fi
if command -v kubectl >/dev/null 2>&1; then
  kubectl config current-context
  kubectl get pods -A -o json | jq -r '.items[] | [.metadata.namespace,.metadata.name,.status.phase] | @tsv'
fi
```

**Check:** gracefully skip absent tools. **Extend:** accept `--namespace` and enforce a safe allowlist.

## Lab 13 — GitHub Actions output and summary

```bash
release_id=$(printf 'four-components' | sha256sum | awk '{print $1}')
if [[ -n "${GITHUB_OUTPUT:-}" ]]; then
  printf 'release_id=%s\n' "$release_id" >> "$GITHUB_OUTPUT"
fi
if [[ -n "${GITHUB_STEP_SUMMARY:-}" ]]; then
  printf '### Release\n\nID: `%s`\n' "$release_id" >> "$GITHUB_STEP_SUMMARY"
fi
printf 'Local release ID: %s\n' "$release_id"
```

**Check:** works locally and emits outputs when Actions environment files are provided. **Extend:** add a validation gate.

## Lab 14 — ShellCheck and Bats CI

Create `scripts/check.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
value=${1:-}
[[ "$value" =~ ^[a-z0-9-]+$ ]] || exit 2
printf '%s\n' "$value"
```

Create `test/check.bats`:

```bash
@test "accepts valid slug" {
  run bash scripts/check.sh frontend-1
  [ "$status" -eq 0 ]
  [ "$output" = frontend-1 ]
}
@test "rejects invalid slug" {
  run bash scripts/check.sh 'bad value'
  [ "$status" -eq 2 ]
}
```

**Check:** `bash -n scripts/check.sh && shellcheck scripts/check.sh && bats test/`. **Extend:** test empty input and an input starting with `-`.

# Part VI — Capstone project

## Capstone: Local immutable-release verification CLI

**Goal:** Build a safe Bash CLI that validates a local release JSON file, computes a deterministic release fingerprint, emits GitHub Actions outputs, and never performs deployment mutations. This is an educational analogue of the release-verification code used in mature GitOps systems, not a replacement for cryptographic provenance verification.

### Directory layout

```text
release-audit/
├── bin/release-audit.sh
├── examples/release.json
└── test/release-audit.bats
```

### Step 1 — Create sample release facts

```bash
mkdir -p release-audit/{bin,examples,test}
cat > release-audit/examples/release.json <<'JSON'
{
  "environment": "qa",
  "components": {
    "frontend": {"image": "ghcr.io/example/frontend", "digest": "sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},
    "catalogue": {"image": "ghcr.io/example/catalogue", "digest": "sha256:bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"},
    "orders": {"image": "ghcr.io/example/orders", "digest": "sha256:cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc"},
    "cart": {"image": "ghcr.io/example/cart", "digest": "sha256:dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"}
  }
}
JSON
```

### Step 2 — Implement the CLI

Save as `release-audit/bin/release-audit.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

usage() {
  cat <<'USAGE'
Usage: release-audit.sh --file FILE [--environment dev|qa|production]
Validate local release JSON and print a deterministic SHA-256 fingerprint.
USAGE
}

die() { printf 'ERROR: %s\n' "$1" >&2; exit "${2:-1}"; }
require_cmd() { command -v "$1" >/dev/null 2>&1 || die "Missing dependency: $1" 69; }

file=''
expected_env=''
while (($#)); do
  case "$1" in
    --file)
      (($# >= 2)) || die '--file requires a value' 2
      file=$2; shift 2 ;;
    --environment)
      (($# >= 2)) || die '--environment requires a value' 2
      expected_env=$2; shift 2 ;;
    --help|-h) usage; exit 0 ;;
    *) die "Unknown argument: $1" 2 ;;
  esac
done

[[ -n "$file" && -f "$file" && -r "$file" ]] || die 'Readable --file is required' 2
[[ -z "$expected_env" || "$expected_env" == dev || "$expected_env" == qa || "$expected_env" == production ]] || die 'Invalid environment' 2
require_cmd jq
require_cmd sha256sum

# Validate exact expected components, fields, and immutable digest shapes.
jq -e '
  type == "object" and
  (keys | sort) == ["components", "environment"] and
  (.environment | IN("dev", "qa", "production")) and
  (.components | type == "object") and
  (.components | keys | sort) == ["cart", "catalogue", "frontend", "orders"] and
  ([.components[] |
    type == "object" and
    (keys | sort) == ["digest", "image"] and
    (.image | type == "string" and startswith("ghcr.io/")) and
    (.digest | type == "string" and test("^sha256:[a-f0-9]{64}$"))
  ] | all)
' "$file" >/dev/null || die 'Release JSON failed schema validation' 3

env=$(jq -r '.environment' "$file")
[[ -z "$expected_env" || "$env" == "$expected_env" ]] || die 'Environment mismatch' 3

# Canonical JSON: jq -S sorts keys, -c emits compact JSON.
# This is a deterministic fingerprint of these exact canonical bytes.
release_id=$(jq -cS . "$file" | sha256sum | awk '{print $1}')
printf 'Environment: %s\nRelease ID: %s\n' "$env" "$release_id"

if [[ -n "${GITHUB_OUTPUT:-}" ]]; then
  printf 'environment=%s\nrelease_id=%s\n' "$env" "$release_id" >> "$GITHUB_OUTPUT"
fi
if [[ -n "${GITHUB_STEP_SUMMARY:-}" ]]; then
  printf '### Verified local release structure\n\n- Environment: `%s`\n- SHA-256: `%s`\n' "$env" "$release_id" >> "$GITHUB_STEP_SUMMARY"
fi
```

**Note:** This fingerprint is stable for the canonical JSON produced by this tool, but it is **not** identical to any other release-ID algorithm that hashes a different subset or uses different JSON separators. It does not verify image signatures, actual registry digest existence, GitHub run identity, or deployment health.

### Step 3 — Run it

```bash
chmod +x release-audit/bin/release-audit.sh
release-audit/bin/release-audit.sh --file release-audit/examples/release.json --environment qa
release-audit/bin/release-audit.sh --file release-audit/examples/release.json --environment production || echo 'Mismatch correctly rejected'
```

### Step 4 — Test with Bats

Save as `release-audit/test/release-audit.bats`:

```bash
setup() {
  SCRIPT="$BATS_TEST_DIRNAME/../bin/release-audit.sh"
  SAMPLE="$BATS_TEST_DIRNAME/../examples/release.json"
}

@test "valid QA release succeeds" {
  run bash "$SCRIPT" --file "$SAMPLE" --environment qa
  [ "$status" -eq 0 ]
  [[ "$output" == *'Release ID:'* ]]
}

@test "wrong environment is rejected" {
  run bash "$SCRIPT" --file "$SAMPLE" --environment production
  [ "$status" -eq 3 ]
}

@test "missing file is rejected" {
  run bash "$SCRIPT" --file /nonexistent/release.json
  [ "$status" -eq 2 ]
}

@test "help works" {
  run bash "$SCRIPT" --help
  [ "$status" -eq 0 ]
  [[ "$output" == *'Usage:'* ]]
}
```

Run:

```bash
bash -n release-audit/bin/release-audit.sh
shellcheck release-audit/bin/release-audit.sh
bats release-audit/test/
```

### Step 5 — Add GitHub Actions validation

Create `.github/workflows/release-audit.yml` in the project repository:

```yaml
name: Release audit
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install checks
        run: |
          sudo apt-get update
          sudo apt-get install -y jq shellcheck bats
      - name: Lint and test
        shell: bash
        run: |
          bash -n release-audit/bin/release-audit.sh
          shellcheck release-audit/bin/release-audit.sh
          bats release-audit/test/
      - name: Audit release JSON
        shell: bash
        run: |
          bash release-audit/bin/release-audit.sh \
            --file release-audit/examples/release.json \
            --environment qa
```

In a real organization, pin actions to reviewed immutable commits and restrict PR workflows' access to sensitive credentials. This capstone intentionally does not run privileged operations.

### Capstone extension challenges

1. Validate a Git commit SHA and expected source repository.
2. Require an allowlisted image repository for each component.
3. Add `--json` output and a meaningful exit-code contract.
4. Test malformed JSON, missing components, unexpected keys, uppercase digests, and empty files.
5. Add a separate read-only GitHub API verification step using `gh api` with a token restricted to read permissions.
6. Add an optional Cosign verification command in a separate, unprivileged workflow; never infer trust from a digest string alone.
7. Implement an explicit `--dry-run` interface for any future mutating extension.

# Part VII — Reference, debugging, assessment

## 42. Common Bash pitfalls and corrections

| Pitfall | Safer approach | Why |
|---|---|---|
| `for f in $(ls)` | `for f in ./*` | Preserves spaces and special characters |
| `rm -rf $dir` | Validate target, then `rm -rf -- "$dir"` | Prevents splitting/options and accidental deletion |
| `cat file | grep x` | `grep x file` | Avoids unnecessary process |
| `echo "$json" | grep ...` | `jq` | JSON is structured, not line-oriented text |
| `cmd || true` everywhere | Handle expected statuses explicitly | Hides real errors |
| `set -e` as only validation | Explicit checks and exit codes | `errexit` has exceptions |
| `eval "$args"` | Command arrays | Prevents code injection |
| `source untrusted.sh` | Parse data, never execute it | Sourcing executes code |
| `curl -k` | Verify TLS normally | Disables certificate verification |
| Parse `kubectl get` table | `kubectl -o json | jq` | Stable structured output |
| Store AWS keys in script | OIDC/profile/secret manager | Reduces credential exposure |
| Assume CI success = deploy success | Observe independent deployment status | CI does not prove rollout health |

## 43. Quick reference

```bash
# Files / tests
[[ -f "$f" ]]             # regular file
[[ -d "$d" ]]             # directory
[[ -n "$s" ]]             # nonempty string
[[ "$s" =~ ^[a-z]+$ ]]    # regex match

# Strings / arrays
${var:-default}            # default if unset/empty
${var:?message}            # fail if unset/empty
${#var}                    # string length
${array[@]}                # array values (quote it!)
${#array[@]}               # array length

# Script execution
"$@"                      # original arguments, separately quoted
$#                        # argument count
$?                        # previous exit status
$!                        # last background PID
$$                        # shell PID

# Diagnostics
bash -n script.sh
shellcheck script.sh
bash -x script.sh          # no secrets!
```

## 44. Debugging workflow

1. Reproduce with minimal non-sensitive input.
2. Run `bash -n` for syntax errors.
3. Run ShellCheck and address quoting/unset-variable warnings.
4. Print relevant values with `printf '%q\n' "$value"` (never secrets).
5. Trace with `bash -x` only in a safe environment.
6. Inspect exit status and stdout/stderr separately.
7. Verify current working directory, PATH, shell version, permissions, and credentials.
8. Add a regression test before fixing the bug.

## 45. Knowledge check

1. What is the difference between `"$@"` and `"$*"`?
2. Why is `[[ ... ]]` preferable to `[ ... ]` in Bash-specific scripts?
3. What is the difference between `return` and `exit`?
4. Why can `set -e` fail to catch an error inside a condition?
5. Why must you quote `"$filename"`?
6. What does `2>&1` do, and why does ordering matter?
7. What is a subshell? How does it affect variable assignments?
8. How would you safely parse JSON from `kubectl`?
9. Why is `curl -k` inappropriate in production?
10. How do you distinguish a successful CI run from a healthy rollout?
11. Why is retrying a mutating API request dangerous?
12. What is the difference between an image tag and an immutable digest?
13. Why should production PRs require independent approval?
14. How do you ensure two scheduled jobs do not run simultaneously?
15. When would you choose Python over Bash?

### Short answer key

1. `"$@"` preserves separate arguments; `"$*"` joins them.
2. `[[ ]]` has safer Bash-specific parsing and operators.
3. `return` leaves a function; `exit` terminates the shell/script.
4. `errexit` is suppressed in several conditional and compound-command contexts.
5. Prevent splitting and unintended glob expansion.
6. Redirect stderr to the current stdout target; redirections are evaluated left-to-right.
7. A child shell environment; its variable changes do not persist in the parent.
8. `kubectl -o json | jq` with explicit error handling.
9. It disables TLS certificate verification.
10. Verify independently recorded deployment and smoke-test evidence.
11. A retry may duplicate a successful but unacknowledged write.
12. Tags can move; digest identifies content (assuming registry/content trust).
13. Separation of duties and risk control.
14. A cooperative lock such as Linux `flock`.
15. Complex data models, parsing, testing, API clients, or maintainability needs.

## 46. Recommended official references

- GNU Bash Reference Manual: https://www.gnu.org/software/bash/manual/
- ShellCheck: https://www.shellcheck.net/
- Bats-core: https://bats-core.readthedocs.io/
- jq manual: https://jqlang.github.io/jq/manual/
- GitHub Actions workflow syntax: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- Docker CLI: https://docs.docker.com/reference/cli/docker/
- kubectl: https://kubernetes.io/docs/reference/kubectl/
- AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/

---

**Final objective:** Be able to read and write reliable Linux automation scripts, understand Bash used in GitHub Actions and GitOps workflows, test scripts locally, and recognize when Python is the better engineering choice.

