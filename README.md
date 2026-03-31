# Sentry CLI

An [agent skill](https://agentskills.io/home) for Sentry release management, debug symbol uploads, source map uploads, commit association, and deploy tracking.

## What it does

When working with Sentry through the command line, this skill helps with:

- Creating and finalizing releases
- Associating commits with releases
- Uploading debug symbols for iOS and mobile crash reporting
- Uploading source maps for JavaScript applications
- Recording deploys across environments
- Verifying Sentry CLI configuration and authentication

## Installation

### As a skill

```bash
npx skills add https://github.com/diegopetrucci/sentry-cli --skill sentry-cli
```

### As a Claude Code plugin

```shell
/plugin marketplace add diegopetrucci/ai-agents-skills
/plugin install sentry-cli@diegopetrucci-claude-plugins
```

## Usage

Trigger the skill when the user needs help with Sentry CLI workflows:

- **Claude Code:** `/sentry-cli`
- Or say: "create a Sentry release", "upload dSYMs", "upload source maps to Sentry", or "record a Sentry deploy"

## More Skills Like This

Found this skill useful? Browse all my hand-crafted ones in the [AI Agents skills](https://github.com/diegopetrucci/ai-agents-skills) repo.

## License

MIT
