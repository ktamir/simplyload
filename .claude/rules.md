# SimplyLoad - Development Rules

**Context:** These rules apply during initial MVP development (Phases 0-5). Once we have a working product, the workflow may shift to feature-driven development with different processes.

## Prime Directive: Stay Minimal

You are building an MVP incrementally. Every line of code must be justified by current requirements in PROJECT_PLAN.md.

## Before Writing Any Code

1. **Check PROJECT_PLAN.md** - Verify which PR you're implementing
2. **Check PROGRESS.md** - Confirm current status
3. **Update PROGRESS.md** - Mark PR in-progress when starting

## Implementation Rules

### DO:
- ✓ Implement ONLY what the current PR specifies
- ✓ Write tests for code you write
- ✓ Use the simplest solution that works
- ✓ Add dependencies when first needed (not upfront)
- ✓ Update PROGRESS.md when PR completes
- ✓ Add type hints

### DON'T:
- ✗ Add features not in current PR scope
- ✗ Create abstractions for single use cases
- ✗ Add error handling for impossible scenarios
- ✗ Add configuration for things that don't vary (see below)
- ✗ Write docstrings for obvious code
- ✗ Add "future-proofing"
- ✗ Optimize without measuring first
- ✗ Add backwards compatibility for non-existent code

## Anti-Patterns to Avoid

### No Premature Abstraction:
```python
# BAD
class StorageProvider(ABC):
    @abstractmethod
    def upload(self): pass

# GOOD - one concrete implementation
def upload_to_s3(data): ...
```

### No Hypothetical Parameters:
```python
# BAD
def load_batch(batch, retry=3, timeout=30): ...

# GOOD
def load_batch(batch): ...
```

### No Configuration for Constants:
**Principle:** Configuration is code. Only make something configurable if you have proof that the value needs to vary.

```python
# BAD - configuring things that never change
config = {
    "batch_size": 10000,           # Will this ever change? No.
    "parquet_compression": "snappy", # Will users choose this? No.
    "max_retries": 3,              # Do we have retry logic yet? No.
    "s3_encryption": "SSE-S3",     # Will we support others? Not in MVP.
}

# GOOD - hardcode constants, extract only when needed
BATCH_SIZE = 10000
s3_client.put_object(..., ServerSideEncryption="AES256")
```

**When to add configuration:**
- ✓ Customer credentials (Snowflake account, Postgres host)
- ✓ Customer choices (which tables to sync)
- ✓ Values proven to vary (found multiple environments need different values)

**When NOT to add configuration:**
- ✗ "Maybe someone will want to change this someday"
- ✗ "This makes it more flexible"
- ✗ "It's a best practice to make things configurable"

**Rule of thumb:** If you can't name 2+ real scenarios where the value differs, hardcode it.

### No Over-Validation:
```python
# BAD - validating internal code
def merge_table(table: SourceTable):
    if not table: raise ValueError()
    # ...

# GOOD - trust internal calls, validate at boundaries
def merge_table(table: SourceTable):
    # Type hint is enough validation
    # ...
```

## Self-Check Before PR Complete

- [ ] Implemented ONLY what's in PR checklist?
- [ ] No "what if" code?
- [ ] No unused parameters or config options?
- [ ] Tests written?
- [ ] PROGRESS.md updated?
- [ ] Can any code be deleted?

## Progress Tracking

**Starting:** Update PROGRESS.md "Current Status" to this PR
**Completing:** Mark complete in PROGRESS.md, update to next PR

## Remember

- Three similar lines > premature abstraction
- Delete unused code immediately
- Hardcode first, extract when proven necessary
- Minimum complexity for current requirements
