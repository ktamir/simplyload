# SimplyLoad - Progress Tracker

**Last Updated:** 2025-12-28

## Current Status
- **Phase:** 0 - Project Setup & Foundation
- **Next PR:** PR-0.1 - Repository Structure & Tooling
- **Blockers:** None

## Quick Stats
- **Total Phases:** 12
- **Completed PRs:** 0
- **In Progress:** None
- **Target:** Postgres → Snowflake MVP (Phase 0-5)

## Recent Completed PRs
None yet

## Upcoming PRs (Next 5)
1. PR-0.1: Repository Structure & Tooling
2. PR-0.2: Core Dependencies
3. PR-0.3: Logging & Observability Foundation
4. PR-1.1: Core Domain Models
5. PR-1.2: Abstract Interfaces

## Key Milestones

### Milestone 1: Foundation Complete ✓
Target: Phase 0-3
- [ ] Project setup
- [ ] Core abstractions
- [ ] S3/Parquet layer
- [ ] Checkpoint system

### Milestone 2: Postgres → Snowflake MVP ✓
Target: Phase 0-5
- [ ] Snowflake integration
- [ ] Postgres CDC
- [ ] Basic replication working

### Milestone 3: Multi-Source Support ✓
Target: Phase 0-7
- [ ] Stripe connector
- [ ] Salesforce connector

### Milestone 4: Production Ready ✓
Target: Phase 0-12
- [ ] Orchestration
- [ ] Monitoring
- [ ] Documentation
- [ ] Security hardening

## Key Decisions Made ✓
- **Language:** Python monolith (can extract Debezium to Java later if needed)
- **Snowflake connection:** Direct connection (Fivetran-style, "just make it work")
- **Postgres CDC:** Start with read-only polling using `psycopg2`, optional logical replication
- **Permissions:** Read-only access for sources (SELECT only)
- **Dependencies:** Add incrementally as needed (not all upfront)
- **S3:** Service-owned bucket initially, 7-day retention
- **Snowflake warehouse:** XS with auto-suspend

## Documents Created
- `PROJECT_PLAN.md` - Full implementation roadmap (12 phases, 70+ PRs)
- `PROGRESS.md` - This file, quick progress tracker
- `CUSTOMER_SETUP.md` - Customer-facing setup instructions and UI requirements

## Questions to Resolve
- [ ] Poetry vs pip for dependency management (leaning Poetry)
- [ ] Async (asyncio) vs threading for concurrency
- [ ] S3 bucket naming: `simplyload-prod-data/customer_<id>/` or similar?
- [ ] Snowflake naming: `raw_data.simplyload.<source>_<database>_<table>`?
