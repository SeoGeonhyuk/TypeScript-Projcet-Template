# Git and Package Management Guide

This guide establishes standardized practices for Git workflow and package management to ensure consistent, reliable, and efficient development processes.

## Overview

Proper version control and package management are fundamental to successful software development. This guide provides clear rules and automation strategies for managing code changes and dependencies.

## Package Management Rules

### Standard Package Manager Selection

**Rule: Use yarn as the standard package manager for all projects.**

#### Why Yarn?
- **Deterministic Installs**: yarn.lock ensures consistent installations across environments
- **Performance**: Parallel installation and caching provide faster dependency resolution
- **Security**: Built-in security auditing and license checking
- **Workspace Support**: Better monorepo and workspace management
- **Offline Mode**: Can work offline with cached packages

#### Required Commands
```bash
# Dependency installation
yarn install           # Install all dependencies
yarn                    # Short form of yarn install

# Package management
yarn add [package]      # Add production dependency
yarn add -D [package]   # Add development dependency
yarn remove [package]   # Remove dependency
yarn upgrade [package]  # Update dependency

# Information and maintenance
yarn list              # List installed packages
yarn info [package]    # Show package information
yarn audit             # Security audit
yarn cache clean       # Clean cache
```

#### Prohibited Commands
```bash
# DO NOT USE npm commands in yarn projects
npm install            # ❌ Use yarn install
npm install [package]  # ❌ Use yarn add [package]
npm uninstall [package] # ❌ Use yarn remove [package]
npm update            # ❌ Use yarn upgrade
```

### Package.json Standards

#### Required Fields
```json
{
  "name": "project-name",
  "version": "1.0.0",
  "description": "Clear project description",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "build command",
    "dev": "development server",
    "test": "test runner",
    "lint": "linting command",
    "typecheck": "type checking"
  },
  "keywords": ["relevant", "keywords"],
  "author": "Author Name",
  "license": "MIT",
  "engines": {
    "node": ">=16.0.0"
  }
}
```

#### Script Naming Conventions
- **build**: Production build
- **dev**: Development server
- **test**: Run tests
- **test:watch**: Watch mode testing
- **lint**: Code linting
- **lint:fix**: Auto-fix linting issues
- **typecheck**: TypeScript type checking
- **clean**: Clean build artifacts

## Git Workflow Rules

### Commit Management Strategy

**Rule: All meaningful changes must be managed with appropriate commits.**

#### Automatic Commit Triggers

1. **Dependency Changes**
   - When package.json changes
   - When yarn.lock changes
   - When adding/removing dependencies

2. **Implementation Completion**
   - When new module implementation is complete
   - When feature implementation is finished
   - When bug fixes are complete

3. **Test Phase Completion**
   - When TDD Green Phase is complete
   - When test suite passes
   - When integration tests complete

4. **Documentation Updates**
   - When significant documentation changes occur
   - When API documentation is updated
   - When README or guides are modified

#### Commit Message Standards

Follow [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

##### Commit Types
- **feat**: New feature addition
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, semicolons, etc.)
- **refactor**: Code refactoring without functionality changes
- **test**: Add or modify tests
- **chore**: Build process, dependency updates, or auxiliary tool changes
- **perf**: Performance improvements
- **ci**: Continuous integration changes
- **build**: Build system changes

##### Examples
```bash
# Feature implementation
git commit -m "feat: implement logger module with CustomLogExporter"

# Bug fix
git commit -m "fix: resolve TypeScript type errors in trace.ts"

# Documentation
git commit -m "docs: update API documentation with new examples"

# Dependency updates
git commit -m "chore: update OpenTelemetry dependencies"

# Test additions
git commit -m "test: add comprehensive tests for logger module"
```

### Branch Strategy

#### Branch Naming Convention
- **main**: Stable release branch
- **develop**: Integration branch for features
- **feature/[feature-name]**: New feature development
- **fix/[issue-description]**: Bug fixes
- **docs/[doc-type]**: Documentation updates
- **refactor/[component-name]**: Code refactoring
- **test/[test-scope]**: Test-related changes

#### Branch Examples
```bash
# Feature branches
feature/console-patching
feature/otlp-exporter
feature/batch-processor

# Bug fix branches
fix/memory-leak-logger
fix/timestamp-precision
fix/type-errors

# Documentation branches
docs/api-reference
docs/getting-started
docs/troubleshooting
```

### Automated Git Processes

#### Process 1: Dependency Management
```bash
# After package installation
yarn install
git add package.json yarn.lock
git commit -m "chore: update dependencies"
```

#### Process 2: Feature Implementation
```bash
# After feature completion
git add [modified-files]
git commit -m "feat: implement [feature-name]

- Add core functionality
- Include comprehensive tests
- Update documentation"
```

#### Process 3: Test Completion
```bash
# After TDD Green Phase
git add [test-files] [implementation-files]
git commit -m "test: add tests for [feature-name] and implement solution

- Red phase: Add failing tests
- Green phase: Implement minimal solution
- All tests passing"
```

### Git Hooks and Automation

#### Pre-commit Hook
```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run type checking
yarn typecheck

# Run linting
yarn lint

# Run tests
yarn test

# Check for secrets
git secrets --scan
```

#### Commit Message Hook
```bash
#!/bin/sh
# .git/hooks/commit-msg

# Validate commit message format
commitizen check --message-file $1
```

## Best Practices

### Repository Structure
```
project-root/
├── .git/
├── .gitignore
├── .gitattributes
├── package.json
├── yarn.lock
├── README.md
├── CHANGELOG.md
├── src/
├── tests/
├── docs/
└── scripts/
```

### .gitignore Standards
```gitignore
# Dependencies
node_modules/
.pnp
.pnp.js

# Production builds
/build
/dist
*.tsbuildinfo

# Environment files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/
*.lcov

# Temporary folders
tmp/
temp/
```

### Security Considerations

#### Dependency Security
```bash
# Regular security audits
yarn audit
yarn audit --level high

# Automatic security updates
yarn upgrade-interactive
```

#### Secret Management
```bash
# Never commit secrets
git secrets --install
git secrets --register-aws
```

## Workflow Automation

### Claude AI Automation Rules

#### Automatic Actions
1. **After yarn install**: Automatically commit package.json and yarn.lock changes
2. **After implementation**: Automatically commit with appropriate message
3. **After tests pass**: Automatically commit test and implementation files
4. **Before major tasks**: Check git status and commit pending changes

#### Automation Scripts
```bash
# scripts/auto-commit-deps.sh
#!/bin/bash
if [[ -n $(git status --porcelain package.json yarn.lock) ]]; then
    git add package.json yarn.lock
    git commit -m "chore: update dependencies"
fi

# scripts/auto-commit-implementation.sh
#!/bin/bash
git add .
git commit -m "feat: implement $1

🤖 Generated with [Claude Code](https://claude.ai/code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### Team Workflow

#### Daily Workflow
1. **Morning**: Pull latest changes, install dependencies
2. **During Work**: Commit frequently with descriptive messages
3. **End of Day**: Push changes, update documentation

#### Weekly Workflow
1. **Monday**: Review dependency updates and security audits
2. **Friday**: Clean up branches, update documentation
3. **As Needed**: Security audits, dependency updates

## Troubleshooting

### Common Issues and Solutions

#### Issue: Dependency Conflicts
```bash
# Solution: Clean installation
rm -rf node_modules yarn.lock
yarn install
```

#### Issue: Git Conflicts
```bash
# Solution: Interactive rebase
git rebase -i HEAD~3
# Edit conflicts, then continue
git rebase --continue
```

#### Issue: Large Commit History
```bash
# Solution: Squash commits
git rebase -i HEAD~5
# Change 'pick' to 'squash' for commits to combine
```

### Performance Optimization

#### Yarn Performance
```bash
# Enable parallel installations
yarn config set network-concurrency 16

# Use local cache
yarn config set cache-folder ~/.yarn-cache

# Disable progress bars in CI
yarn config set progress false
```

#### Git Performance
```bash
# Enable file system monitor
git config core.fsmonitor true

# Enable commit graph
git config core.commitGraph true
git config gc.writeCommitGraph true
```

## Quality Assurance

### Pre-commit Checklist
- [ ] All tests passing
- [ ] Code linted and formatted
- [ ] TypeScript compilation successful
- [ ] Dependencies security audited
- [ ] Commit message follows convention
- [ ] Documentation updated

### Release Preparation
- [ ] Version bumped appropriately
- [ ] CHANGELOG.md updated
- [ ] Dependencies updated and audited
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Build successful

---

**Version**: 1.0  
**Last Updated**: 2025-07-11  
**Based on**: OpenTelemetry Front-End SDK Project Experience  
**Maintained by**: Claude AI Development Team