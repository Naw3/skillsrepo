---
name: claude-code-plugins
description: |
  Patterns for building Claude Code plugins including commands, agents,
  hooks, MCP servers, and plugin marketplaces. Use when creating
  extensions for Claude Code or distributing plugins to teams.
---

# Claude Code Plugin Development

Patterns for building and distributing Claude Code plugins.

## Plugin Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # Plugin manifest
├── commands/             # Slash commands
│   └── review.md
├── agents/               # Agent definitions
│   └── code-reviewer.md
├── hooks/                # Event hooks
│   └── pre-commit.md
├── mcp-servers/          # MCP server configs
│   └── database.json
└── README.md
```

## Plugin Manifest

```json
// .claude-plugin/plugin.json
{
  "name": "my-awesome-plugin",
  "description": "A collection of productivity tools",
  "version": "1.0.0",
  "author": {
    "name": "Your Name",
    "email": "your@email.com"
  },
  "repository": "https://github.com/username/my-plugin",
  "keywords": ["productivity", "automation"],
  "license": "MIT"
}
```

## Commands

### Simple Command

```markdown
<!-- commands/review.md -->
---
name: review
description: Review code for best practices and issues
---

Review the selected code or recent changes for:
- Potential bugs or edge cases
- Security concerns
- Performance issues
- Readability improvements

Be concise and actionable.
```

### Command with Arguments

```markdown
<!-- commands/translate.md -->
---
name: translate
description: Translate code to another language
arguments:
  - name: language
    description: Target programming language
    required: true
  - name: style
    description: Coding style (functional, oop, procedural)
    required: false
---

Translate the selected code to $language$.
@if style
Use a $style$ coding style.
@endif

Preserve the logic and functionality while following $language$ idioms.
```

### Command with Confirmation

```markdown
<!-- commands/deploy.md -->
---
name: deploy
description: Deploy to production
allowed_tools:
  - run_command
confirm: true
---

Deploy the current project to production:
1. Run tests
2. Build for production
3. Deploy using the configured method
```

## Agents

```markdown
<!-- agents/code-reviewer.md -->
---
name: code-reviewer
description: Thorough code review agent
model_preferences:
  - claude-3-5-sonnet
---

# Code Reviewer Agent

You are an expert code reviewer. When reviewing code:

## Focus Areas
1. **Correctness** - Logic errors, edge cases, null handling
2. **Security** - Input validation, SQL injection, XSS
3. **Performance** - Algorithmic complexity, memory leaks
4. **Maintainability** - Code clarity, naming, documentation

## Review Process
1. Understand the context and purpose
2. Check for obvious issues first
3. Look for subtle bugs
4. Suggest improvements with examples

## Response Format
- Use severity levels: 🔴 Critical, 🟡 Warning, 🟢 Suggestion
- Provide fix examples when possible
- Be constructive and educational
```

## Hooks

### Pre-commit Hook

```markdown
<!-- hooks/pre-commit.md -->
---
name: pre-commit
trigger: pre-commit
description: Validate changes before commit
---

Before committing, verify:
1. All tests pass
2. No console.log statements in production code
3. No hardcoded secrets or API keys
4. TypeScript compiles without errors

If issues found, list them and suggest fixes.
```

### On-file-open Hook

```markdown
<!-- hooks/on-file-open.md -->
---
name: on-file-open
trigger: file-open
patterns:
  - "*.test.ts"
  - "*.spec.ts"
---

This is a test file. Remember:
- Keep tests focused and isolated
- Use descriptive test names
- Mock external dependencies
```

## MCP Server Integration

```json
// mcp-servers/database.json
{
  "name": "database-tools",
  "description": "Database query and management tools",
  "command": "npx",
  "args": ["-y", "@company/db-mcp-server"],
  "env": {
    "DATABASE_URL": "${env:DATABASE_URL}"
  }
}
```

## Plugin Marketplace

### Create a Marketplace

```json
// .claude-plugin/marketplace.json
{
  "name": "team-plugins",
  "owner": {
    "name": "Dev Team",
    "email": "team@company.com"
  },
  "description": "Internal tools for the development team",
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/formatter",
      "description": "Automatic code formatting",
      "version": "2.1.0",
      "category": "productivity",
      "tags": ["formatting", "linting"]
    },
    {
      "name": "deploy-tools",
      "source": {
        "source": "github",
        "repo": "company/deploy-plugin"
      },
      "description": "Deployment automation"
    },
    {
      "name": "external-plugin",
      "source": {
        "source": "git",
        "url": "https://gitlab.company.com/plugins/tool.git",
        "ref": "v1.2.0"
      },
      "description": "Tool from GitLab"
    }
  ]
}
```

### Source Types

```json
// Relative path (same repo)
{ "source": "./plugins/my-plugin" }

// GitHub repo
{ 
  "source": {
    "source": "github",
    "repo": "owner/repo"
  }
}

// GitHub with specific ref
{
  "source": {
    "source": "github",
    "repo": "owner/repo",
    "ref": "v1.0.0"
  }
}

// Any git repository
{
  "source": {
    "source": "git",
    "url": "https://gitlab.com/user/repo.git",
    "ref": "main"
  }
}
```

### Using Marketplaces

```bash
# Add a marketplace
/plugin marketplace add https://github.com/company/plugin-marketplace

# List available plugins
/plugin marketplace list

# Install from marketplace
/plugin install code-formatter@team-plugins

# Update marketplace
/plugin marketplace update
```

## Publishing Workflow

### 1. Create Plugin Repository

```bash
mkdir my-plugin
cd my-plugin
git init

# Create structure
mkdir -p .claude-plugin commands agents

# Create manifest
cat > .claude-plugin/plugin.json << 'EOF'
{
  "name": "my-plugin",
  "description": "My awesome plugin",
  "version": "1.0.0"
}
EOF
```

### 2. Add to Marketplace

```json
// In your marketplace.json
{
  "plugins": [
    {
      "name": "my-plugin",
      "source": {
        "source": "github",
        "repo": "username/my-plugin"
      },
      "description": "My awesome plugin"
    }
  ]
}
```

### 3. Version and Release

```bash
# Tag versions for stability
git tag v1.0.0
git push origin v1.0.0

# Reference specific version in marketplace
{
  "source": {
    "source": "github",
    "repo": "username/my-plugin",
    "ref": "v1.0.0"
  }
}
```

## Best Practices

1. **Clear naming** - Use descriptive names for commands
2. **Good descriptions** - Help users understand what each plugin does
3. **Version semantically** - Follow semver for updates
4. **Document thoroughly** - Include README with examples
5. **Test locally first** - Use `/plugin marketplace add ./local-path`
6. **Use specific refs** - Pin versions in production marketplaces
7. **Validate before distributing** - Use `/plugin validate`

## Troubleshooting

```bash
# Validate plugin structure
/plugin validate ./my-plugin

# Check marketplace syntax
/plugin marketplace validate ./my-marketplace

# Reinstall plugin
/plugin uninstall my-plugin
/plugin install my-plugin@marketplace-name

# Clear plugin cache
# Plugins are cached; changes may require reinstall
```
