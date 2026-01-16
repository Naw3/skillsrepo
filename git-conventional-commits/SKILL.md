---
name: git-conventional-commits
description: |
  Conventional commits specification and Git workflow patterns.
  Use when writing commit messages, setting up commit hooks,
  or implementing semantic versioning automation.
---

# Git Conventional Commits

Standardized commit messages for semantic versioning and changelogs.

## Commit Format

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

## Types

| Type | Description | Semver |
|------|-------------|--------|
| `feat` | New feature | MINOR |
| `fix` | Bug fix | PATCH |
| `docs` | Documentation only | - |
| `style` | Formatting, missing semicolons | - |
| `refactor` | Code change that neither fixes nor adds | - |
| `perf` | Performance improvement | PATCH |
| `test` | Adding or updating tests | - |
| `build` | Build system or dependencies | - |
| `ci` | CI configuration | - |
| `chore` | Other changes (no src/test) | - |
| `revert` | Reverts a previous commit | varies |

**Breaking changes:** Add `!` after type or `BREAKING CHANGE:` in footer → MAJOR

## Examples

### Simple Commits

```bash
# Feature
git commit -m "feat(auth): add Google OAuth login"

# Bug fix
git commit -m "fix(api): handle null response from external service"

# Documentation
git commit -m "docs(readme): add installation instructions"

# Refactor
git commit -m "refactor(utils): extract date formatting to helper"

# Performance
git commit -m "perf(images): implement lazy loading"
```

### With Scope

```bash
feat(auth): add password reset flow
fix(api): correct rate limiting calculation
docs(contributing): add PR guidelines
style(lint): apply prettier formatting
refactor(hooks): simplify useAuth logic
test(user): add integration tests for signup
build(deps): upgrade React to v19
ci(github): add automated testing workflow
```

### Breaking Changes

```bash
# With ! notation
feat(api)!: change response format to JSON:API spec

# With footer
feat(api): change response format

BREAKING CHANGE: API responses now follow JSON:API specification.
All clients must update their response parsing logic.
```

### With Body

```bash
git commit -m "fix(search): handle empty query gracefully

Previously, an empty search query would cause a 500 error.
Now it returns an empty results array with proper status code.

Fixes #123"
```

### Multi-line Commits

```bash
git commit -F- <<EOF
feat(payments): implement Stripe subscription

Add support for recurring payments using Stripe Billing API:
- Monthly and annual billing cycles
- Free trial period configuration
- Automatic invoice generation

Closes #456
EOF
```

## Scopes by Project Area

```
# Frontend
feat(ui): component changes
feat(pages): page-level changes
feat(hooks): custom hooks
feat(store): state management

# Backend
feat(api): API routes
feat(db): database changes
feat(auth): authentication
feat(queue): background jobs

# Infrastructure
build(docker): containerization
ci(deploy): deployment pipeline
chore(config): configuration files
```

## Git Hooks (Husky + Commitlint)

### Setup

```bash
npm install -D husky @commitlint/cli @commitlint/config-conventional
npx husky init
```

### commitlint.config.js

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'perf',
        'test',
        'build',
        'ci',
        'chore',
        'revert',
      ],
    ],
    'scope-enum': [
      2,
      'always',
      [
        'api',
        'auth',
        'ui',
        'db',
        'deps',
        'config',
      ],
    ],
    'subject-case': [2, 'always', 'lower-case'],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
  },
}
```

### .husky/commit-msg

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no -- commitlint --edit $1
```

## Changelog Generation

### Using standard-version

```bash
npm install -D standard-version
```

```json
// package.json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  }
}
```

### Generated CHANGELOG.md

```markdown
# Changelog

## [1.2.0] - 2024-01-15

### Features

* **auth:** add Google OAuth login ([abc123])
* **payments:** implement Stripe subscription ([def456])

### Bug Fixes

* **api:** handle null response from external service ([ghi789])

### BREAKING CHANGES

* **api:** response format changed to JSON:API spec
```

## Branch Naming

```bash
# Feature branches
git checkout -b feat/user-authentication
git checkout -b feat/JIRA-123-payment-integration

# Bug fix branches
git checkout -b fix/login-redirect
git checkout -b fix/JIRA-456-api-timeout

# Other
git checkout -b docs/api-documentation
git checkout -b refactor/auth-module
git checkout -b chore/upgrade-dependencies
```

## PR Title Convention

Same format as commits:

```
feat(auth): add password reset flow
fix(api): handle rate limiting correctly
docs: update contributing guidelines
```

## Quick Reference

```bash
# Features
feat: new feature
feat(scope): scoped feature
feat!: breaking feature

# Fixes
fix: bug fix
fix(scope): scoped fix

# Other
docs: documentation
style: formatting
refactor: code restructure
perf: performance
test: tests
build: build system
ci: CI/CD
chore: maintenance
revert: revert commit
```
