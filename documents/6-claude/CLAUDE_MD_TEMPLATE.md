# CLAUDE.md Template

This template provides a standardized structure for CLAUDE.md files in software development projects, incorporating best practices and essential workflow rules derived from real project experiences.

## Project Overview

[PROJECT_NAME] - [Brief project description]

### Key Project Details
- **Language**: [Programming language and version]
- **Build Tool**: [Build system - e.g., Vite, Webpack, etc.]
- **Target**: [Target environment - e.g., Web browsers, Node.js, etc.]
- **Package Manager**: [Package manager - e.g., npm, yarn, pnpm]
- **Architecture**: [Architecture approach - e.g., layered, microservices, etc.]

## Development Commands

```bash
# Essential commands for development
npm run build          # Build the project
npm run dev            # Development server
npm run test           # Run tests
npm run lint           # Code linting
npm run typecheck      # Type checking
```

## Project Architecture

### Core Components Structure
```
src/
├── [component1]       # [Description]
├── [component2]       # [Description]
└── [component3]       # [Description]
```

### Key Architecture Decisions
- **[Decision 1]**: [Rationale]
- **[Decision 2]**: [Rationale]
- **[Decision 3]**: [Rationale]

## Core Dependencies

### Required Dependencies
- `[dependency1]` - [Purpose]
- `[dependency2]` - [Purpose]
- `[dependency3]` - [Purpose]

### Build Dependencies
- `[dev-dependency1]` - [Purpose]
- `[dev-dependency2]` - [Purpose]

## Key Features & Requirements

### API Design
```typescript
// Example API interface
interface [InterfaceName] {
  [property1]: [type];     // [Description]
  [property2]: [type];     // [Description]
}

// Usage example
[functionName]({
  [property1]: '[value]',
  [property2]: '[value]'
});
```

## Testing Strategy

The project uses comprehensive test coverage including:
- **Unit Tests**: Individual component testing
- **Integration Tests**: End-to-end functionality
- **Performance Tests**: Performance benchmarks
- **Compatibility Tests**: Cross-platform functionality

## Code Structure Guidelines

### Best Practices
- Use strict TypeScript configuration
- Implement proper error handling
- Follow interface-based design
- Maintain clear separation of concerns

### Module Organization
- **[module1]**: [Purpose and responsibilities]
- **[module2]**: [Purpose and responsibilities]
- **[module3]**: [Purpose and responsibilities]

## Documentation Workflow Requirements

### MANDATORY Documentation Process
**All implementation work must follow this documentation workflow:**

1. **Pre-implementation Planning**
   - Document goals and scope in DEVELOPMENT_LOG.md
   - Identify estimated time and potential blockers

2. **Learning During Implementation**
   - Document new technologies, APIs, patterns immediately using LEARNING_NOTES_TEMPLATE
   - File location: `documents/5-learning/LEARNING_NOTES_[date].md`

3. **Post-implementation Documentation**
   - Update DEVELOPMENT_LOG.md daily logs immediately after completion
   - Document blocker solutions, performance metrics, and next plans

### Documentation Rules
- **LEARNING_NOTES**: Mandatory when learning new technologies/concepts
- **DEVELOPMENT_LOG**: Mandatory update after all implementation completion
- **File Locations**: 
  - Development log: `documents/4-progress/DEVELOPMENT_LOG.md`
  - Learning notes: `documents/5-learning/LEARNING_NOTES_[YYYY-MM-DD].md`

### Non-compliance Consequences
Implementation without following this documentation process is considered **incomplete work**.
Claude must comply with these rules and generate appropriate documentation at all implementation stages.

## Git and Package Management Rules

### Package Manager Rules
**This project uses yarn as the standard package manager.**

- **Dependency installation**: `yarn install` or `yarn`
- **Add packages**: `yarn add [package-name]`
- **Add dev dependencies**: `yarn add -D [package-name]`
- **npm usage prohibited**: Always use yarn instead of npm commands

### Git Commit Management Rules
**All meaningful changes must be managed with appropriate commits.**

#### Automatic Commit Targets
1. **Dependency changes**: When package.json, yarn.lock change
2. **Implementation completion**: When new module implementation is complete
3. **Test passes**: When TDD Green Phase is complete
4. **Documentation updates**: When important documentation changes occur

#### Commit Message Rules
- **feat**: Add new feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **test**: Add/modify test code
- **refactor**: Code refactoring
- **chore**: Build process or auxiliary tool changes

#### Branch Strategy
- **main**: Stable release branch
- **feature/[feature-name]**: New feature development
- **fix/[bug-name]**: Bug fixes
- **docs/[doc-name]**: Documentation updates

### Automation Process
Claude should automatically perform the following processes:

1. **Commit after dependency installation**
   ```bash
   yarn install
   git add package.json yarn.lock
   git commit -m "chore: update dependencies"
   ```

2. **Commit after implementation completion**
   ```bash
   git add [modified-files]
   git commit -m "feat: implement [feature-name]"
   ```

3. **Commit after test passes**
   ```bash
   git add [test-files] [implementation-files]
   git commit -m "test: add tests for [feature-name] and implement solution"
   ```

## Context Management and /compact Rules

### Context Cleanup on TDD Cycle Completion
**Execute /compact mandatory when major work units (TDD Red-Green-Refactor cycle) are complete**

#### /compact Execution Timing
1. **After TDD cycle completion**
   - Red Phase (failing test creation) → Green Phase (implementation) → Refactor Phase (refactoring) complete
   - After one module (e.g., logger.ts, trace.ts) implementation complete
   - After related documentation updates and commits are complete

2. **After major milestone completion**
   - Phase unit work completion (e.g., Phase 1, Phase 2)
   - After integration testing completion
   - After build and deployment preparation completion

#### /compact Execution Process
```
1. Check current work status
   - Verify all tests pass
   - Verify commits are complete
   - Verify documentation updates are complete

2. Execute /compact
   - Compress conversation context
   - Optimize memory
   - Clean up token usage

3. Prepare for next work
   - Start new TDD cycle
   - Plan next module implementation
```

#### /compact Execution Criteria
- **Mandatory execution**: After TDD cycle completion (Red → Green → Refactor)
- **Recommended execution**: After 500+ lines of code implementation completion
- **Optional execution**: After complex debugging session completion

### Context Management Benefits
1. **Performance optimization**: Reduced token usage and improved response speed
2. **Improved focus**: Better focus on current work by removing unnecessary context
3. **Memory efficiency**: Optimized memory usage through conversation history cleanup
4. **Quality improvement**: Better code quality with clean context

## Error Handling

### Graceful Degradation
- Proper error logging without interfering with application
- Fallback mechanisms for network failures
- Handle missing dependencies gracefully

### Custom Error Classes
```typescript
class [ProjectName]Error extends Error {
  constructor(message: string, public code: string) {
    super(message);
    this.name = '[ProjectName]Error';
  }
}
```

## Performance Requirements

### Target Metrics
- **Bundle size**: < [size]KB
- **Runtime overhead**: < [time]ms
- **Memory usage**: < [size]MB
- **Network latency**: Minimized through [optimization strategy]

## Security Considerations

- **No sensitive data exposure**: Never log secrets or keys
- **Input validation**: Validate all user inputs
- **Error handling**: Don't expose internal details in error messages
- **Dependencies**: Regular security audits of dependencies

## Development Notes

### Current Status
[Current project status - e.g., implementation phase, documentation phase, etc.]

### Next Steps
1. [Step 1]
2. [Step 2]
3. [Step 3]

### Important Considerations
- **[Consideration 1]**: [Details]
- **[Consideration 2]**: [Details]
- **[Consideration 3]**: [Details]

## Customization Guidelines

### How to Use This Template

1. **Copy and Customize**
   - Copy this template to your project root as `CLAUDE.md`
   - Replace all `[PLACEHOLDER]` values with project-specific information
   - Remove sections not applicable to your project

2. **Essential Sections**
   - Always keep: Project Overview, Development Commands, Documentation Workflow
   - Always keep: Git and Package Management Rules, Context Management Rules
   - Customize: Architecture, Dependencies, API Design based on project needs

3. **Project-Specific Adaptations**
   - Add technology-specific sections (e.g., React, Node.js, Python)
   - Include framework-specific best practices
   - Add domain-specific requirements (e.g., frontend, backend, mobile)

4. **Maintenance**
   - Update commands and dependencies as project evolves
   - Add new learnings and patterns discovered during development
   - Keep documentation workflow rules current

### Template Validation Checklist

- [ ] All placeholders replaced with actual values
- [ ] Development commands tested and working
- [ ] Documentation workflow rules clearly understood
- [ ] Git and package management rules configured
- [ ] Context management rules established
- [ ] Project-specific sections added
- [ ] Security considerations addressed
- [ ] Performance targets defined

---

**Template Version**: 1.0  
**Last Updated**: [DATE]  
**Based on**: OpenTelemetry Front-End SDK Project Experience  
**Created by**: Claude AI Development Team