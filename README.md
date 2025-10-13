# crewkit

AI agent management for development teams

[![oclif](https://img.shields.io/badge/cli-oclif-brightgreen.svg)](https://oclif.io)
[![Version](https://img.shields.io/npm/v/@crewkit/cli.svg)](https://www.npmjs.com/package/@crewkit/cli)
[![Downloads/week](https://img.shields.io/npm/dw/@crewkit/cli.svg)](https://www.npmjs.com/package/@crewkit/cli)

Crewkit is a CLI-first platform for managing AI agents like Claude Code with role-based configurations, team collaboration, and performance monitoring.

## Installation

```bash
npm install -g @crewkit/cli
```

## Quick Start

```bash
# Initialize your project
crewkit init

# Authenticate with crewkit.io
crewkit auth login

# Start Claude Code with synced agents
crewkit code
```

## Features

- **Role-based agent configurations** - Customize AI behavior by team member seniority
- **Team collaboration** - Share agent configurations across your organization
- **Version control** - Track and roll back agent modifications
- **Performance monitoring** - Measure agent effectiveness and cost

## Commands

### Authentication

```bash
crewkit auth login   # Authenticate with crewkit.io
crewkit auth status  # Check authentication status
crewkit auth logout  # Clear credentials
```

### Project Setup

```bash
crewkit init [org] [project]  # Initialize crewkit in your project
```

### Development

```bash
crewkit code         # Start Claude Code with synced agents
crewkit code --no-watch  # Start without file watching
```

## Configuration

### API URL Override

By default, the CLI connects to `https://api.na.crewkit.io`. You can override this by setting the `CREWKIT_API_URL` environment variable:

```bash
# Use a different region
export CREWKIT_API_URL=https://api.eu.crewkit.io

# Use a custom/self-hosted instance
export CREWKIT_API_URL=https://crewkit.example.com

# For local development
export CREWKIT_API_URL=http://localhost:3050
```

The environment variable is respected by all commands, so you can set it once in your shell configuration.

## Documentation

Visit [docs.crewkit.io](https://docs.crewkit.io) for full documentation.

## Support

- GitHub Issues: [github.com/crewkit/crewkit/issues](https://github.com/crewkit/crewkit/issues)
- Email: support@crewkit.io

## License

MIT
