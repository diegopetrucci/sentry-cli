---
name: sentry-cli
description: Sentry CLI tool for iOS error tracking and monitoring workflows. Use when working with Sentry for (1) Listing and investigating issues and crashes, (2) Uploading debug symbols (dSYMs) for crash report symbolication, (3) Creating and managing releases, (4) Associating commits with releases for issue tracking, (5) Managing deploys across environments, (6) Viewing logs and monitoring cron jobs, (7) Sending test events, or any other Sentry-related iOS development and deployment tasks. Also use when the user mentions "sentry", "crashes", "dSYMs", "symbolication", "debug files", or "crash reports".
license: MIT
metadata:
  author: Diego Petrucci
  version: "2.0"
---

# sentry-cli for iOS

## Installation

```bash
# macOS (recommended)
brew install getsentry/tools/sentry-cli

# npm
npm install -g @sentry/cli

# curl (macOS/Linux)
curl -sL https://sentry.io/get-cli/ | sh

# Pin a specific version
curl -sL https://sentry.io/get-cli/ | SENTRY_CLI_VERSION="3.3.5" sh
```

Update or uninstall:
```bash
sentry-cli update
sentry-cli uninstall
```

## Configuration

sentry-cli loads configuration from multiple sources (in order of precedence): command-line flags, environment variables, `.sentryclirc` in the current directory (searched upward), and `~/.sentryclirc`.

### Project config file (`.sentryclirc` in project root)

```ini
[auth]
token=<your-auth-token>

[defaults]
org=<your-org>
project=<your-project>
```

### Environment variables

```bash
export SENTRY_AUTH_TOKEN="<your-auth-token>"
export SENTRY_ORG="<your-org>"
export SENTRY_PROJECT="<your-project>"
```

### Interactive login

```bash
sentry-cli login

# For self-hosted Sentry
sentry-cli --url https://your-sentry.example.com/ login
```

### Setup verification

Verify auth, org, project, and scopes:

```bash
sentry-cli info
```

If the output shows missing scopes or auth errors, the token may need to be regenerated in Sentry (Settings > Auth Tokens).

Key scopes needed:
- `project:read` / `project:write` — releases
- `event:read` — issues and events
- `org:read` — org-level queries

## Commands

### List issues

#### Basic usage

```bash
# Recent unresolved issues
sentry-cli issues list -s unresolved --max-rows 10 --pages 1

# All issues (including ignored/resolved)
sentry-cli issues list --max-rows 10 --pages 1

# A specific issue by ID
sentry-cli issues list -i <ISSUE_ID>
```

#### CLI flags

| Flag | Description |
|---|---|
| `--max-rows N` | Limit output to N rows |
| `--pages N` | Max pages to fetch (100 issues/page, default: 5) |
| `-s STATUS` | Filter by status: `resolved`, `muted`, `unresolved` |
| `--query QUERY` | Sentry search query (see syntax below) |
| `-i ID` | Fetch a specific issue by numeric ID |

#### Query syntax (`--query`)

The `--query` flag accepts Sentry's structured search syntax. Combine multiple filters — they AND together by default.

**By status and severity:**
```bash
--query "is:unresolved"
--query "is:unresolved level:fatal"
--query "is:unresolved level:error"
```

`is:` values: `unresolved`, `resolved`, `archived`, `assigned`, `unassigned`, `for_review`
`level:` values: `fatal`, `error`, `warning`, `info`

**By error type and handling:**
```bash
--query "error.type:NSInvalidArgumentException"
--query "error.unhandled:true"
--query "error.main_thread:true"
```

**By release and environment:**
```bash
--query "release:<bundle-id>@<version>+<build>"
--query "release.package:<bundle-id>"
```

**By device and platform:**
```bash
--query "device.family:iPhone"
--query "os.build:22A3354"
--query "app.in_foreground:true"
```

**By time:**
```bash
--query "age:-24h"              # first seen in the last 24 hours
--query "lastSeen:-1w"          # seen in the last week
--query "firstSeen:+30d"        # first seen more than 30 days ago
--query "timesSeen:>100"        # seen more than 100 times
```

Time suffixes: `m` (minutes), `h` (hours), `d` (days), `w` (weeks). `-` means "within the last", `+` means "older than".

**By stack trace:**
```bash
--query "stack.module:<ModuleName>"
--query "stack.function:<functionName>"
--query "stack.filename:<FileName>.swift"
```

**By user:**
```bash
--query "user.id:<user-id>"
--query "assigned:me"
--query "assigned:#<team-slug>"
```

**Operators:**
- Negation: `!user.email:test@example.com`
- Comparison: `timesSeen:>100`, `timesSeen:<=10`
- Wildcards: `error.type:NS*Exception`
- Lists: `release:[1.0, 2.0]` (OR)
- Has field: `has:user.email`

**Combining filters:**
```bash
# Fatal crashes on a specific app in the last 7 days
--query "is:unresolved level:fatal release.package:<bundle-id> lastSeen:-7d"

# Unhandled errors in a specific module
--query "is:unresolved error.unhandled:true stack.module:<ModuleName>"
```

#### Sorting by frequency (API only)

The CLI does not support sorting. For frequency-based ranking (most events first), use the Sentry API directly:

```bash
# Top 5 unresolved issues by event count
curl -sH "Authorization: Bearer $SENTRY_AUTH_TOKEN" \
  "https://sentry.io/api/0/projects/<org>/<project>/issues/?sort=freq&query=is:unresolved" \
  | jq '.[0:5] | .[] | {id: .id, shortId: .shortId, title: .title, count: .count}'

# Top fatal crashes by frequency
curl -sH "Authorization: Bearer $SENTRY_AUTH_TOKEN" \
  "https://sentry.io/api/0/projects/<org>/<project>/issues/?sort=freq&query=is:unresolved+level:fatal" \
  | jq '.[0:5] | .[] | {id: .id, shortId: .shortId, title: .title, count: .count, lastSeen: .lastSeen}'

# Sort options: freq (frequency), date (last seen), new (first seen), priority (Sentry priority)
```

### List releases

```bash
sentry-cli releases list
```

### Create and finalize a release

```bash
VERSION="<bundle-id>@<version>+<build>"

sentry-cli releases new "$VERSION"
sentry-cli releases set-commits "$VERSION" --local
sentry-cli releases finalize "$VERSION"
```

Use `--auto` instead of `--local` if the GitHub/GitLab repository integration is configured in Sentry.

### Upload dSYMs

```bash
# From an xcarchive
sentry-cli debug-files upload --include-sources \
  /path/to/App.xcarchive/dSYMs

# With server-side processing wait
sentry-cli debug-files upload --wait /path/to/dsyms

# iOS with BCSymbolMaps (for bitcode builds with hidden symbols)
sentry-cli debug-files upload \
  --symbol-maps /path/to/BCSymbolMaps \
  /path/to/dsyms

# Validate before uploading
sentry-cli debug-files check /path/to/dSYM

# Find missing debug files when crashes show unsymbolicated addresses
sentry-cli debug-files find <debug-identifier>
```

### List events

Events are individual occurrences — more granular than issues (which group related events together).

```bash
# Recent events
sentry-cli events list --max-rows 10 --pages 1

# Include user info and tags
sentry-cli events list --max-rows 10 -U -T
```

| Flag | Description |
|---|---|
| `--max-rows N` | Limit output rows |
| `--pages N` | Max pages (100 events/page, default: 5) |
| `-U` / `--show-user` | Show the user column |
| `-T` / `--show-tags` | Show the tags column |

### Send a test event

Manually fire an event to verify the pipeline works end-to-end. Useful after setting up auth or changing SDK configuration.

```bash
# Simple test error
sentry-cli send-event -m "Test event from sentry-cli"

# With severity level
sentry-cli send-event -m "Test warning" -l warning

# With tags and extra data
sentry-cli send-event -m "Test from CI" -t environment:staging -e build_number:123

# Attach a release
sentry-cli send-event -m "Test event" -r "<bundle-id>@<version>+<build>"

# Send breadcrumbs from a log file (last 100 lines)
sentry-cli send-event -m "Crash context" --logfile /path/to/app.log --with-categories
```

`-l` levels: `debug`, `info`, `warning`, `error` (default), `fatal`

### Logs (beta)

Query structured logs sent by the SDK (requires `enableLogs: true` in SDK configuration).

```bash
# Recent logs
sentry-cli logs list --max-rows 20

# Filter by level
sentry-cli logs list --query "level:error" --max-rows 50

# Live tail (streams new logs every 2 seconds)
sentry-cli logs list --live

# Live tail with custom interval
sentry-cli logs list --live --poll-interval 5
```

| Flag | Description |
|---|---|
| `--max-rows N` | Max entries to fetch (default: 100, max: 1000) |
| `--query QUERY` | Filter with Sentry search syntax (e.g. `level:error`) |
| `--live` | Stream logs continuously |
| `--poll-interval N` | Seconds between polls in live mode (default: 2) |

### Monitors (cron)

Track scheduled jobs — Sentry alerts if a job doesn't check in on time or fails.

```bash
# List configured monitors
sentry-cli monitors list

# Wrap a command so Sentry tracks its execution
sentry-cli monitors run <monitor-slug> -- ./run-job.sh

# With schedule (creates the monitor if it doesn't exist)
sentry-cli monitors run <monitor-slug> \
  --schedule "0 3 * * *" \
  --timezone "Europe/London" \
  --checkin-margin 5 \
  --max-runtime 30 \
  -- ./run-job.sh
```

| Flag | Description |
|---|---|
| `--schedule CRON` | Crontab schedule (creates monitor if needed) |
| `--timezone TZ` | tz database string (e.g. `Europe/London`) |
| `--checkin-margin N` | Minutes after expected time before marking missed |
| `--max-runtime N` | Minutes before marking timed out |
| `-e ENV` | Environment (default: `production`) |

### Create a deploy record

```bash
sentry-cli deploys new --release "$VERSION" -e production
```

### List projects and repos

```bash
# List all projects in the org
sentry-cli projects list

# List linked repositories
sentry-cli repos list
```

Useful for verifying which projects and repos the token has access to, or checking if the repository integration is connected (needed for `releases set-commits --auto`).

## Common iOS release workflow

```bash
VERSION=$(sentry-cli releases propose-version)
sentry-cli releases new "$VERSION"
sentry-cli debug-files upload --wait "$DWARF_DSYM_FOLDER_PATH"
sentry-cli releases set-commits "$VERSION" --local
sentry-cli releases finalize "$VERSION"
sentry-cli deploys new --release "$VERSION" -e production
```
