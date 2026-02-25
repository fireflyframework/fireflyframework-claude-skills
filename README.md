# Firefly Framework — Claude Code Skills

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Skills plugin for accelerating development with [Firefly Framework](https://github.com/fireflyframework) — reactive microservices, CQRS, Sagas, Event-Driven Architecture, and more.

## Installation

```
/plugin marketplace add fireflyframework/fireflyframework-claude-skills
/plugin install fireflyframework-claude-skills@fireflyframework-skills
```

## Phase 1 Skills

| Skill | Description |
|-------|-------------|
| `firefly-conventions` | Enforces Firefly Framework coding conventions and project structure |
| `create-microservice` | Scaffolds a new Firefly-based reactive microservice |
| `implement-cqrs` | Generates CQRS command/query separation with Firefly patterns |
| `implement-saga` | Creates distributed saga orchestration workflows |
| `testing-reactive-services` | Generates reactive test harnesses and integration tests |

> More skills are coming in future phases — stay tuned!

## Project Structure

```
.claude-plugin/
  plugin.json          # Plugin metadata
  marketplace.json     # Marketplace registration
skills/
  firefly-conventions/
  create-microservice/
  implement-cqrs/
  implement-saga/
  testing-reactive-services/
commands/              # Slash commands (coming soon)
agents/                # Agent definitions (coming soon)
```

## Links

- [Firefly Framework GitHub Organization](https://github.com/fireflyframework)
- [Apache 2.0 License](LICENSE)

## License

Copyright 2026 Firefly Software Solutions Inc

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
