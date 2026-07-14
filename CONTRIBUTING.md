# Contributing

Thank you for your interest in contributing to python-best-practices!

## Ways to Contribute

- **Add new rules** - Expand coverage with additional best practices
- **Improve existing rules** - Enhance examples, fix errors, or add clarity
- **Report issues** - Found a problem? Open an issue

## Project Structure

```
skills/
├── coding-standards/          # Python coding best practices
│   ├── SKILL.md               # Skill definition and overview
│   ├── metadata.json          # Skill metadata
│   └── rules/
│       ├── _template.md       # Rule template
│       └── {prefix}-{name}.md # Individual rules
├── tooling/                   # Development tool configuration
│   ├── SKILL.md
│   ├── metadata.json
│   └── rules/
└── testing/                   # Test-writing best practices
    ├── SKILL.md
    ├── metadata.json
    └── rules/
```

## Adding a New Rule

### 1. Choose the right skill and category

| Skill | Categories |
|-------|-----------|
| coding-standards | `error-`, `perf-`, `async-`, `design-`, `solid-`, `doc-`, `validation-`, `oop-` |
| tooling | `analysis-`, `lint-`, `type-`, `fmt-`, `test-`, `pkg-` |
| testing | `struct-`, `fixture-`, `param-`, `mock-` |

### 2. Create the rule file

Copy the template and create your rule:

```bash
cp skills/{skill}/rules/_template.md skills/{skill}/rules/{prefix}-{name}.md
```

### 3. Write the rule

Follow this structure:

```markdown
---
title: Rule Title
impact: CRITICAL | HIGH | MEDIUM | LOW
impactDescription: Brief impact (e.g., "2x faster", "O(1) vs O(n)")
tags: [tag1, tag2, tag3]
---

# Rule Title [IMPACT]

## Description
Why this rule matters and when to apply it.

## Bad Example
```python
# Code to avoid
```

## Good Example
```python
# Recommended approach
```

## Notes
- Edge cases and exceptions
- Additional tips

## References
- [Link](URL)
```

### 4. Update SKILL.md and _sections.md

Add your rule to the appropriate category table in `skills/{skill}/SKILL.md` and to its section in `skills/{skill}/rules/_sections.md`.

### 5. Update metadata.json

Increment the `rules` count in `skills/{skill}/metadata.json`.

## Guidelines

- **Be concise** - Rules should be scannable by AI agents
- **Show, don't tell** - Good/bad examples are more valuable than lengthy explanations
- **Include impact** - Quantify benefits when possible (e.g., "1.5-2x faster")
- **Add references** - Link to official docs or authoritative sources

## Pull Requests

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

Keep PRs focused on a single change when possible.
