# Development Workflow

## Starting a New PR

### 1. Identify Next PR
```bash
# Read PROGRESS.md to see what's next
cat PROGRESS.md | grep "Next PR"
```

### 2. Read PR Requirements
```bash
# Open PROJECT_PLAN.md and find the PR section
# Read the checklist carefully
```

### 3. Update Progress Before Starting
Edit PROGRESS.md:
```markdown
## Current Status
- **Phase:** X - [Phase Name]
- **Next PR:** PR-X.Y - [PR Name]  # ← Update this
- **Blockers:** None

## Recent Completed PRs
(keep existing...)
```

### 4. Implement
- Follow the PR checklist exactly
- Add dependencies if PR mentions them (update pyproject.toml)
- Write tests as you go
- Stay minimal - resist adding extras

### 5. Test
```bash
# Run tests for what you built
pytest tests/path/to/new_tests.py -v

# If applicable, run type checking
mypy src/simplyload/module_you_changed.py
```

## Completing a PR

### 1. Self-Review Checklist
Ask yourself:
- [ ] Did I implement everything in the PR checklist?
- [ ] Did I add anything NOT in the PR checklist?
- [ ] Are there unused parameters, config options, or abstractions?
- [ ] Did I write tests?
- [ ] Do tests pass?
- [ ] Can I delete any code without breaking functionality?

### 2. Update PROGRESS.md
```markdown
## Current Status
- **Phase:** X - [Phase Name]
- **Next PR:** PR-X.Y+1 - [Next PR Name]  # ← Advance to next
- **Blockers:** None

## Recent Completed PRs
1. PR-X.Y - [PR Name] (2025-12-28)  # ← Add this line
2. PR-X.Y-1 - [Previous PR] (2025-12-27)
...
```

### 3. Commit
```bash
git add .
git commit -m "PR-X.Y: [Brief description]

- Implemented [item 1]
- Implemented [item 2]
- Added tests

Part of Phase X: [Phase Name]"
```

### 4. Create Branch/PR (if desired)
```bash
# Optional: Create feature branch for review
git checkout -b feature/pr-X.Y-description
git push origin feature/pr-X.Y-description
```

## When You Get Stuck

### Ask These Questions:
1. **Am I over-engineering?**
   - Can I do this simpler?
   - Do I need this abstraction right now?

2. **Am I solving problems I don't have?**
   - Is this for "someday" or "today"?
   - Can I hardcode this for now?

3. **Am I following the plan?**
   - Is this in the PR checklist?
   - Should I defer this to a later PR?

### Simplify:
- Delete unused parameters
- Delete unused configuration options
- Delete abstraction layers you're not using
- Hardcode constants
- Remove defensive checks on internal code

## Example Session

```
# Starting PR-2.1: S3 Client Wrapper

1. Read PROJECT_PLAN.md PR-2.1 section
   → Create S3Client class
   → upload_file() method
   → list_files() method
   → delete_files() method
   → Add encryption (SSE-S3)
   → Unit tests with moto

2. Update PROGRESS.md
   → Current Status: PR-2.1 in progress

3. Add dependency
   → poetry add boto3 moto (if not present)

4. Implement src/simplyload/storage/s3_client.py
   → Simple class, concrete methods
   → Hardcode encryption="AES256"
   → No retry logic yet (not in requirements)
   → No timeout config (not needed yet)

5. Write tests/storage/test_s3_client.py
   → Mock S3 with moto
   → Test upload, list, delete

6. Run tests
   → pytest tests/storage/test_s3_client.py -v

7. Self-review
   → ✓ Implemented everything in checklist
   → ✓ Nothing extra added
   → ✓ Tests pass
   → Can delete any code? No.

8. Update PROGRESS.md
   → Mark PR-2.1 complete
   → Set Current Status: PR-2.2

9. Commit
   → git commit -m "PR-2.1: S3 Client Wrapper"
```

## Red Flags (Stop and Simplify)

🚩 You're adding parameters you don't use yet
🚩 You're writing "just in case" code
🚩 You're creating base classes for one implementation
🚩 You're adding config for things that don't vary
🚩 You're validating internal function calls
🚩 You're optimizing without measuring
🚩 You're thinking "this will be useful later"

**When you see a red flag: DELETE IT.**
