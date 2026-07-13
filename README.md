# CloudBees agent skills

This repository hosts [agent skills](https://agentskills.io/home) / [Claude Code plugins](https://code.claude.com/docs/en/discover-plugins) for using CloudBees products through AI agents.

* [Smart Tests](plugins/smart-tests/README.md)

## Installation
Claude Code users can run the following slash command:
```
/plugin marketplace add cloudbees-oss/agent-skills
```

For other agents, use [the skills CLI](https://skills.sh/) from your terminal:
```
npx skills add cloudbees-oss/agent-skills
```

## Update
Claude Code users can run the following slash command:
```
/plugin marketplace update cloudbees
```

For other agents:
```
npx skills update
```

# Contributing
External contributions welcome.

CloudBees employees who are interested in working on this,
please see [this](https://cloudbees.atlassian.net/wiki/x/EAAWcwE).

## Acknowledgements
The `flaky-tests` skill builds on the foundational work by [@hsbt](https://github.com/hsbt) in [#2](https://github.com/cloudbees-oss/agent-skills/pull/2).
