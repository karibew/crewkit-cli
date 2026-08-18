# Contributing to crewkit CLI

Thank you for your interest in contributing to crewkit.

---

## How to Contribute

The crewkit codebase is private — this repository hosts releases, public
documentation, and the issue tracker. Here's how you can contribute:

### 1. Bug Reports

Found a bug? Please [open an issue](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml) with:
- Clear description of the problem
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, CLI version from `crewkit --version`, install method)
- Error messages or logs — the per-session debug log lives at
  `.crewkit/debug-latest.log` in your project root (sanitize any sensitive info!)

For code-intelligence problems, attach `crewkit lsp doctor` output.

### 2. Feature Requests

Have an idea? [Open a feature request](https://github.com/karibew/crewkit-cli/issues/new?template=feature_request.yml) describing:
- What problem it solves
- Proposed solution
- Alternative solutions considered
- Why this would benefit other teams using crewkit

### 3. Documentation

Help improve our docs:
- Fix typos or unclear explanations
- Add examples or use cases
- Improve installation/setup guides
- Create tutorials or blog posts

To contribute documentation:
1. Fork this repository
2. Make your changes
3. Submit a pull request
4. We'll review it

### 4. Community Support

Help other users:
- Answer questions in [Discussions](https://github.com/karibew/crewkit-cli/discussions)
- Share your workflows and tips
- Create example projects or templates

---

## Issue Guidelines

### Before Opening an Issue

- **Search existing issues** to avoid duplicates
- **Check the [FAQ](docs/faq.md)** and [troubleshooting guide](docs/troubleshooting.md)
- **Update to the latest version** (`crewkit update`, or `brew upgrade crewkit` /
  `npm update -g @crewkit/cli` if you installed through a package manager)

### Writing Good Issues

**Good bug report:**
```
Title: "crewkit code fails with ENOENT on Windows"

Description:
When running `crewkit code` on Windows 11, I get:

Error: ENOENT: no such file or directory

Steps to reproduce:
1. Install crewkit 0.3.3 (via choco)
2. Run crewkit init
3. Run crewkit code

Environment:
- OS: Windows 11
- CLI: crewkit 0.3.3 (choco install)

Expected: Claude Code should launch
Actual: ENOENT error
```

**Good feature request:**
```
Title: "Add --config flag to override agent configs"

Problem:
I want to test different agent configurations without modifying files.

Proposed solution:
Add `crewkit code --config custom-rails.md` to override specific agents.

Benefits:
- Faster experimentation
- No file system changes
- Easier A/B testing locally
```

---

## Code of Conduct

### Be Respectful

- Use welcoming and inclusive language
- Respect differing viewpoints and experiences
- Accept constructive criticism gracefully
- Focus on what's best for the community

### Be Helpful

- Provide clear and actionable feedback
- Help others understand technical concepts
- Share knowledge generously

---

## Response Times

crewkit is built by a small team. As a rule of thumb, we aim to acknowledge
and label new issues within a few business days, prioritize critical bugs
for the next release, and consider feature requests as part of roadmap
planning. These are targets, not guarantees.

---

## Security Issues

**Do not open public issues for security vulnerabilities.**

Instead, please see [SECURITY.md](SECURITY.md) for responsible disclosure.

---

## Questions?

- **General questions**: [Start a discussion](https://github.com/karibew/crewkit-cli/discussions)
- **Bug reports**: [Open an issue](https://github.com/karibew/crewkit-cli/issues/new/choose)
- **Security**: See [SECURITY.md](SECURITY.md)

---

Thank you for helping make crewkit better.
