# Contributing to crewkit CLI

Thank you for your interest in contributing to crewkit! 🎉

---

## How to Contribute

Since the crewkit codebase is private, we welcome the following types of contributions:

### 1. 🐛 Bug Reports

Found a bug? Please [open an issue](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml) with:
- Clear description of the problem
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, Node version, CLI version)
- Error messages or logs (sanitize any sensitive info!)

### 2. 💡 Feature Requests

Have an idea? [Open a feature request](https://github.com/karibew/crewkit-cli/issues/new?template=feature_request.yml) describing:
- What problem it solves
- Proposed solution
- Alternative solutions considered
- Why this would benefit the community

### 3. 📖 Documentation

Help improve our docs:
- Fix typos or unclear explanations
- Add examples or use cases
- Improve installation/setup guides
- Create tutorials or blog posts

To contribute documentation:
1. Fork this repository
2. Make your changes
3. Submit a pull request
4. We'll review and merge!

### 4. 💬 Community Support

Help other users:
- Answer questions in [Discussions](https://github.com/karibew/crewkit-cli/discussions)
- Share your workflows and tips
- Create example projects or templates

---

## Issue Guidelines

### Before Opening an Issue

- **Search existing issues** to avoid duplicates
- **Check the FAQ** and troubleshooting guide
- **Update to latest version** (`npm update -g @crewkit/cli`)

### Writing Good Issues

**Good bug report:**
```
Title: "crewkit code fails with ENOENT on Windows"

Description:
When running `crewkit code` on Windows 11, I get:

Error: ENOENT: no such file or directory

Steps to reproduce:
1. Install @crewkit/cli@0.1.4
2. Run crewkit init
3. Run crewkit code

Environment:
- OS: Windows 11
- Node: v20.10.0
- CLI: @crewkit/cli@0.1.4

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

### Be Patient

- Maintainers are volunteers
- Response times may vary
- Complex issues take time to investigate

---

## Response Times

We strive to:
- **Acknowledge issues** within 2-3 business days
- **Label issues** (bug/enhancement/question) within 1 week
- **Fix critical bugs** in next patch release
- **Consider feature requests** for future releases

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

Thank you for helping make crewkit better! 🙏
