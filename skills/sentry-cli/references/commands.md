# Sentry CLI Commands Reference

## Installation

```bash
brew install getsentry/tools/sentry-cli    # macOS
npm install -g @sentry/cli                 # npm
curl -sL https://sentry.io/get-cli/ | sh  # curl (macOS/Linux)
```

## Configuration & Authentication

### Login
```bash
sentry-cli login
sentry-cli --url https://myserver.invalid/ login   # self-hosted
sentry-cli login --auth-token <token>
```

### Verify Configuration
```bash
sentry-cli info
```

### Configuration File (~/.sentryclirc or .sentryclirc in project root)

Loaded from current directory upward, with `~/.sentryclirc` as fallback. Standard INI syntax.

```ini
[auth]
token=<your-auth-token>

[defaults]
url=https://sentry.io
org=<your-org>
project=<your-project>
vcs_remote=origin
```

### Properties File (Java/React Native)
```bash
export SENTRY_PROPERTIES=/path/to/sentry.properties
```

### HTTP & Proxy Settings

| Config Key | Purpose |
|-----------|---------|
| `http.proxy_url` | Proxy URL |
| `http.proxy_username` | Proxy auth username |
| `http.proxy_password` | Proxy auth password |
| `http.verify_ssl` | SSL verification (default: true) |
| `http.keepalive` | HTTP keepalive (default: true) |
| `http.max_retries` | Max retry attempts (default: 5) |

### Debug File Upload Limits

| Config Key | Purpose |
|-----------|---------|
| `dif.max_upload_size` | Max DIF upload batch size (default: 35-100MB) |
| `dif.max_item_size` | Max single file in DIF bundle (default: 1MB) |

## Release Management

### Create Release
```bash
sentry-cli releases new "$VERSION"

# Auto-determined version from git
VERSION=$(sentry-cli releases propose-version)
sentry-cli releases new "$VERSION"

# Create and finalize immediately
sentry-cli releases new "$VERSION" --finalize

# Override org and project
sentry-cli releases -o <org> -p <project> new "$VERSION"
```

### Finalize Release
```bash
sentry-cli releases finalize "$VERSION"
```

### Associate Commits
```bash
# Automatic (requires repository integration)
sentry-cli releases set-commits "$VERSION" --auto

# Local git (no integration needed)
sentry-cli releases set-commits "$VERSION" --local

# Manual commit spec
sentry-cli releases set-commits "$VERSION" --commit "repo-owner/repo-name@deadbeef"

# Commit range
sentry-cli releases set-commits "$VERSION" --commit \
  "repo-owner/repo-name@<previous>..<current>"

# Ignore missing commits (rebased/amended)
sentry-cli releases set-commits "$VERSION" --auto --ignore-missing
```

### List Releases
```bash
sentry-cli releases list
```

## Debug Information Files (dSYMs)

### Upload Debug Symbols
```bash
# Basic upload (recursively scans directory)
sentry-cli debug-files upload /path/to/dsyms

# With server-side processing wait
sentry-cli debug-files upload --wait /path/to/dsyms

# With source bundles for source context
sentry-cli debug-files upload --include-sources /path/to/dsyms

# iOS: BCSymbolMaps for hidden symbols (bitcode builds)
sentry-cli debug-files upload \
  --symbol-maps /path/to/BCSymbolMaps \
  /path/to/dsyms

# Override org/project
sentry-cli debug-files upload -o <org> -p <project> /path/to/dsyms
```

### Validate Debug Files
```bash
sentry-cli debug-files check <file>
```

### Find Missing Debug Files
```bash
sentry-cli debug-files find <debug-identifier>
```

### Create Source Bundles Separately
```bash
sentry-cli debug-files bundle-sources /path/to/files...
sentry-cli debug-files upload --type sourcebundle /path/to/bundles...
```

## Issues

### List Issues
```bash
sentry-cli issues list -s unresolved --max-rows 10 --pages 1
sentry-cli issues list -i <ISSUE_ID>
```

| Flag | Description |
|---|---|
| `--max-rows N` | Limit output to N rows |
| `--pages N` | Max pages (100 issues/page, default: 5) |
| `-s STATUS` | `resolved`, `muted`, `unresolved` |
| `--query QUERY` | Sentry search query |
| `-i ID` | Specific issue by numeric ID |

## Events

### List Events
```bash
sentry-cli events list --max-rows 10 --pages 1
sentry-cli events list --max-rows 10 -U -T
```

| Flag | Description |
|---|---|
| `--max-rows N` | Limit output rows |
| `--pages N` | Max pages (100 events/page, default: 5) |
| `-U` / `--show-user` | Show user column |
| `-T` / `--show-tags` | Show tags column |

## Send Event

```bash
sentry-cli send-event -m "message"
sentry-cli send-event -m "message" -l warning
sentry-cli send-event -m "message" -t key:value -e extra_key:value
sentry-cli send-event -m "message" -r "<release>"
sentry-cli send-event -m "message" --logfile /path/to/log --with-categories
```

`-l` levels: `debug`, `info`, `warning`, `error` (default), `fatal`

## Logs (beta)

```bash
sentry-cli logs list --max-rows 20
sentry-cli logs list --query "level:error" --max-rows 50
sentry-cli logs list --live
sentry-cli logs list --live --poll-interval 5
```

| Flag | Description |
|---|---|
| `--max-rows N` | Max entries (default: 100, max: 1000) |
| `--query QUERY` | Sentry search syntax |
| `--live` | Stream continuously |
| `--poll-interval N` | Seconds between polls (default: 2) |

## Monitors (Cron)

```bash
sentry-cli monitors list
sentry-cli monitors run <slug> -- <command>
sentry-cli monitors run <slug> \
  --schedule "0 3 * * *" \
  --timezone "Europe/London" \
  --checkin-margin 5 \
  --max-runtime 30 \
  -- <command>
```

## Deploys

### Create Deploy
```bash
sentry-cli deploys new --release "$VERSION" -e <environment>

# With duration tracking
start=$(date +%s)
# ... deployment ...
now=$(date +%s)
sentry-cli deploys new --release "$VERSION" -e <environment> -t $((now-start))
```

### List Deploys
```bash
sentry-cli deploys list --release "$VERSION"
```

## Projects & Repos

```bash
sentry-cli projects list
sentry-cli repos list
```

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `SENTRY_AUTH_TOKEN` | Primary authentication token |
| `SENTRY_API_KEY` | Legacy API key |
| `SENTRY_DSN` | DSN for direct connection |
| `SENTRY_URL` | Server endpoint (defaults to sentry.io) |
| `SENTRY_ORG` | Organization slug |
| `SENTRY_PROJECT` | Project slug |
| `SENTRY_VCS_REMOTE` | VCS remote name (default: origin) |
| `SENTRY_PIPELINE` | Environment for User-Agent header |
| `SENTRY_LOG_LEVEL` | Verbosity: `trace`, `debug`, `info`, `warn`, `error` |
| `SENTRY_HTTP_MAX_RETRIES` | Max retries (default: 5) |
| `SENTRY_NO_PROGRESS_BAR` | Disable progress bars (set to `1`) |
| `SENTRY_LOAD_DOTENV` | Load `.env` file (default: enabled; `0` to disable) |
| `SENTRY_DOTENV_PATH` | Custom `.env` file path |
| `SENTRY_PROPERTIES` | Path to properties file (Java/React Native) |
| `SENTRY_ALLOW_FAILURE` | Ignore symbol upload failures |
| `SENTRY_DISABLE_UPDATE_CHECK` | Disable auto-update checks |
