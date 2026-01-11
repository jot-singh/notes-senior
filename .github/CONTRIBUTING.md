# Contributing Guidelines

## Git Workflow

### Commit After Each Task

**After completing each required task, always commit your changes:**

```bash
git add .
git commit -m "Descriptive message about what was completed"
git push
```

### Commit Message Format

Use clear, descriptive commit messages:

- ✅ `Add microservices fundamentals notes`
- ✅ `Expand REST API authentication section with OAuth examples`
- ✅ `Add Java concurrency patterns documentation`
- ❌ `Update files`
- ❌ `WIP`
- ❌ `Changes`

### When to Commit

Commit after:
- Creating a new note file
- Completing a major section
- Adding code examples or diagrams
- Finishing expansion of existing content
- Completing any logical unit of work

### Branch Strategy

- Work on `main` branch for documentation updates
- Create feature branches for experimental content: `feature/topic-name`
- Keep commits atomic and focused

### Before Committing

1. Review your changes
2. Ensure all code examples are properly formatted
3. Check that markdown renders correctly
4. Verify all links work

## Code Examples

- Include language identifiers in code blocks
- Add comments for complex logic
- Ensure examples are production-quality

## Documentation Standards

- Use proper heading hierarchy
- Include practical examples
- Add "What You'll Learn" and "Why This Matters" sections
- Reference real-world use cases
- Avoid emojis in technical content
