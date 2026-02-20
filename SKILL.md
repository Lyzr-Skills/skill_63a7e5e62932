---
name: code-review
description: Review code for bugs, security issues, and best practices.
---

When asked to review code, follow this structured process:

## Step 1: Read the files
Read all files the user wants reviewed. If they provide a directory, list it first and read each file.

## Step 2: Analyze
Check for these categories:

### Bugs & Logic Errors
- Off-by-one errors
- Null/undefined access
- Unhandled edge cases
- Race conditions
- Memory leaks

### Security
- SQL injection
- XSS vulnerabilities
- Command injection
- Hardcoded secrets or credentials
- Insecure dependencies

### Performance
- N+1 queries
- Unnecessary re-renders
- Missing memoization
- Inefficient algorithms
- Large bundle imports

### Code Quality
- Dead code
- Duplicated logic
- Missing error handling
- Unclear naming
- Overly complex functions (>30 lines)

## Step 3: Report
Format your review as:

```
## Code Review: [filename]

### Critical (must fix)
- [issue with line reference]

### Warning (should fix)
- [issue with line reference]

### Suggestion (nice to have)
- [improvement idea]

### What's good
- [positive observations]
```

Be specific. Always reference line numbers. Suggest fixes, not just problems.
