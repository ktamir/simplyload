# SimplyLoad Playground - Implementation Plan

**Purpose:** Build testing environment to validate SimplyLoad hypothesis against Fivetran/Rivery/Airbyte Cloud.

**Location:** `~/projects/simplyload-playground/` (separate repo)

**Timeline:** 1-2 weeks

**Cost:** < $15 total

---

## Overview

Build POC environment in small, testable increments to:
1. Generate realistic test data
2. Compare competitor tools
3. Validate cost hypothesis
4. Inform SimplyLoad development

---

## Phase 0: Playground Setup

### PR-0.1: Initialize Playground Repository
**Branch:** `main` (initial commit)

- [ ] Create directory structure:
```
simplyload-playground/
├── README.md
├── .gitignore
├── testing-app/
└── infra/
```

- [ ] Create README.md:
  - Link to POC_PLAN.md in main repo
  - Overview of playground purpose
  - Quick start instructions (to be filled)

- [ ] Create .gitignore:
  - Python (.venv, __pycache__, etc.)
  - Docker (volumes)
  - Pulumi (.pulumi/, Pulumi.*.yaml with secrets)
  - IDE files

- [ ] Git init and initial commit

**Success:** Empty repo structure, ready for development

---

## Phase 1: Database Schema & Data Generator

### PR-1.1: Database Schema
**Branch:** `feature/database-schema`

- [ ] Create `testing-app/schema.sql`:
  - Users table (with updated_at, deleted_at)
  - Orders table (with status, updated_at)
  - Events table (JSONB properties)
  - updated_at trigger function
  - Indexes on updated_at columns

- [ ] Create `testing-app/docker-compose.yml`:
  - Postgres 15 service
  - Logical replication enabled (wal_level=logical)
  - Mount schema.sql as init script
  - Expose port 5432

- [ ] Test locally:
```bash
docker-compose up -d
psql -h localhost -U testuser -d testing_saas -c '\dt'
```

**Success:** Can start Postgres with schema loaded

---

### PR-1.2: Configuration & Base Generator
**Branch:** `feature/base-generator`

- [ ] Create `testing-app/requirements.txt`:
  - faker
  - psycopg2-binary
  - python-dotenv

- [ ] Create `testing-app/.env.example`:
  - DATABASE_URL
  - INITIAL_USERS=1000
  - INITIAL_ORDERS=5000
  - INITIAL_EVENTS=20000
  - ORDERS_PER_HOUR=50
  - EVENTS_PER_HOUR=200

- [ ] Create `testing-app/src/config.py`:
  - Load config from .env
  - Dataclass for TestConfig
  - Validate values

- [ ] Create `testing-app/src/__init__.py` (empty)

- [ ] Test:
```bash
python -c "from src.config import config; print(config)"
```

**Success:** Configuration loads from environment

---

### PR-1.3: Data Generator Implementation
**Branch:** `feature/data-generator`

- [ ] Create `testing-app/src/generator.py`:
  - `DataGenerator` class
  - `generate_users(count)` method
  - `generate_order(user_id, quantity)` method
  - `generate_event(user_id, event_type)` method
  - `update_order_status(order_id, status)` method
  - `soft_delete_user(user_id)` method
  - Connection management

- [ ] Add type hints
- [ ] Keep it simple (no error handling for impossible cases)

- [ ] Create `testing-app/src/test_generator.py`:
  - Test user generation
  - Test order generation
  - Test soft delete

- [ ] Test manually:
```python
from src.generator import DataGenerator
gen = DataGenerator()
gen.generate_users(10)
```

**Success:** Can generate test data programmatically

---

### PR-1.4: Baseline Data Script
**Branch:** `feature/baseline-script`

- [ ] Create `testing-app/scripts/01_setup.py`:
  - Load config
  - Create DataGenerator
  - Generate initial users
  - Generate initial orders
  - Generate initial events
  - Print summary

- [ ] Add progress indicators (print statements)

- [ ] Test:
```bash
python scripts/01_setup.py
# Should complete in < 1 minute
```

- [ ] Validate:
```sql
SELECT 'users' as table, COUNT(*) FROM users
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'events', COUNT(*) FROM events;
```

**Success:** Can populate database with baseline data

---

### PR-1.5: Continuous Activity Script
**Branch:** `feature/continuous-script`

- [ ] Create `testing-app/src/scenarios.py`:
  - `ContinuousScenario` class
  - Calculate probabilities from config
  - Main loop with sleep(1)
  - Random operations based on probabilities
  - Graceful shutdown (Ctrl+C)

- [ ] Create `testing-app/scripts/02_continuous.py`:
  - Initialize scenario
  - Run for configured duration
  - Print activity stats every 10 minutes

- [ ] Test:
```bash
# Run for 5 minutes
ORDERS_PER_HOUR=60 EVENTS_PER_HOUR=120 TEST_DURATION_HOURS=0.083 \
  python scripts/02_continuous.py
```

**Success:** Generates continuous realistic activity

---

### PR-1.6: Validation Script
**Branch:** `feature/validation-script`

- [ ] Create `testing-app/scripts/03_validate.py`:
  - Connect to Postgres
  - Query row counts
  - Query soft delete counts
  - Query recent activity (last hour)
  - Print formatted summary

- [ ] Test:
```bash
python scripts/03_validate.py
# Should show current database state
```

**Success:** Can validate database state easily

---

### PR-1.7: Testing App Documentation
**Branch:** `feature/testing-app-docs`

- [ ] Create `testing-app/README.md`:
  - Purpose
  - Setup instructions
  - Running baseline data
  - Running continuous load
  - Validating data
  - Configuration options
  - Troubleshooting

- [ ] Update `.env.example` with comments

**Success:** Testing app is documented and usable

---

## Phase 2: Infrastructure with Pulumi

### PR-2.1: Pulumi Project Setup
**Branch:** `feature/pulumi-init`

- [ ] Create `infra/requirements.txt`:
  - pulumi
  - pulumi-aws

- [ ] Create `infra/Pulumi.yaml`:
  - Project name: simplyload-playground
  - Runtime: python
  - Description

- [ ] Create `infra/.gitignore`:
  - .pulumi/
  - venv/
  - Pulumi.*.yaml (stack-specific configs with secrets)

- [ ] Create `infra/__main__.py`:
  - Import pulumi and pulumi_aws
  - Empty for now (just structure)

- [ ] Initialize:
```bash
cd infra
pulumi login --local
pulumi stack init dev
```

- [ ] Test:
```bash
pulumi preview  # Should show "no resources"
```

**Success:** Pulumi project initialized

---

### PR-2.2: S3 Bucket for SimplyLoad
**Branch:** `feature/s3-bucket`

- [ ] Implement in `infra/__main__.py`:
  - Create S3 bucket
  - Private ACL
  - SSE-AES256 encryption
  - 7-day lifecycle rule
  - Tags (project: simplyload-poc)

- [ ] Export bucket name and ARN

- [ ] Test:
```bash
pulumi config set aws:region us-east-1
pulumi up
```

- [ ] Validate:
```bash
aws s3 ls | grep simplyload-testing
pulumi stack output s3_bucket
```

**Success:** S3 bucket created and accessible

---

### PR-2.3: Optional RDS Configuration
**Branch:** `feature/optional-rds`

- [ ] Add RDS logic to `infra/__main__.py`:
  - Check config flag: enable_rds (default: false)
  - If enabled:
    - Get default VPC
    - Create security group (allows Postgres from specified IP)
    - Create RDS instance (db.t3.micro, Postgres 15)
    - Enable logical replication (parameter group)
    - Export connection string

- [ ] Test without RDS:
```bash
pulumi up  # Should only have S3
```

- [ ] Test with RDS (optional, costs money):
```bash
pulumi config set enable_rds true
pulumi config set allowed_ip_cidr "YOUR_IP/32"
pulumi config set --secret db_password "TestPassword123"
pulumi up  # Creates RDS
```

**Success:** Can optionally provision RDS

---

### PR-2.4: Infrastructure Documentation
**Branch:** `feature/infra-docs`

- [ ] Create `infra/README.md`:
  - Prerequisites (Pulumi, AWS credentials)
  - Option A: Local Postgres (free)
  - Option B: RDS Postgres (costs)
  - Configuration reference
  - Deployment instructions
  - Cleanup instructions
  - Troubleshooting

- [ ] Add cost estimates to docs

**Success:** Infrastructure is documented

---

## Phase 3: Cost Tracking & Monitoring

### PR-3.1: Snowflake Cost Tracker
**Branch:** `feature/snowflake-tracker`

- [ ] Create `testing-app/src/cost_tracker.py`:
  - `SnowflakeCostTracker` class
  - Connect to Snowflake
  - Query query_history for warehouse usage
  - Calculate costs (XS = $2/hour)
  - Return summary dict

- [ ] Add snowflake-connector-python to requirements.txt

- [ ] Create `testing-app/scripts/04_measure_costs.py`:
  - Track Snowflake warehouse usage
  - Calculate unit costs (per 1M rows)
  - Compare to Fivetran pricing
  - Print formatted report

- [ ] Test:
```bash
SNOWFLAKE_ACCOUNT=... SNOWFLAKE_USER=... python scripts/04_measure_costs.py
```

**Success:** Can measure actual Snowflake costs

---

### PR-3.2: Infrastructure Cost Tracker
**Branch:** `feature/infra-costs`

- [ ] Add to `cost_tracker.py`:
  - `S3CostTracker` class
  - Calculate S3 storage costs
  - Calculate S3 API call costs (estimate)
  - `EC2CostTracker` class (if using EC2)
  - Aggregate infrastructure costs

- [ ] Update `04_measure_costs.py`:
  - Include infrastructure costs
  - Separate customer costs (Snowflake) from service costs (infra)
  - Show correct comparison

- [ ] Test with actual POC data

**Success:** Complete cost picture (infra + Snowflake)

---

## Phase 4: Competitor Testing Setup

### PR-4.1: Competitor Setup Documentation
**Branch:** `feature/competitor-docs`

- [ ] Create `testing-app/docs/FIVETRAN_SETUP.md`:
  - Sign up instructions
  - Connect Postgres (via ngrok)
  - Configure tables
  - Setup instructions
  - Screenshot placeholders

- [ ] Create `testing-app/docs/RIVERY_SETUP.md`:
  - Similar to Fivetran

- [ ] Create `testing-app/docs/AIRBYTE_SETUP.md`:
  - Similar to Fivetran

- [ ] Create `testing-app/docs/SNOWFLAKE_SETUP.md`:
  - Trial signup
  - Warehouse creation
  - Schema setup for each tool
  - User setup

**Success:** Step-by-step guides for all competitors

---

### PR-4.2: ngrok Helper
**Branch:** `feature/ngrok-helper`

- [ ] Create `testing-app/scripts/expose_postgres.sh`:
```bash
#!/bin/bash
# Exposes local Postgres via ngrok
ngrok tcp 5432
```

- [ ] Document in testing-app/README.md:
  - When to use (connecting cloud ETL tools)
  - How to get ngrok URL
  - Security considerations

**Success:** Easy way to expose local Postgres

---

## Phase 5: Results & Analysis

### PR-5.1: Results Template
**Branch:** `feature/results-template`

- [ ] Create `RESULTS.md` (in playground root):
  - Setup time for each tool (fill during testing)
  - UX notes for each tool
  - Cost comparison table
  - Snowflake efficiency comparison
  - Feature comparison
  - Decision matrix

- [ ] Format as template with blank fields

**Success:** Ready to document findings

---

### PR-5.2: Final Playground Documentation
**Branch:** `feature/final-docs`

- [ ] Update root `README.md`:
  - Complete overview
  - Architecture diagram (ASCII art or link)
  - Quick start (full workflow)
  - Link to all sub-docs
  - Link to POC_PLAN.md in main repo

- [ ] Add `WORKFLOW.md`:
  - Day-by-day testing protocol
  - What to measure and when
  - Validation queries
  - Screenshot checklist

**Success:** Playground is fully documented and ready to use

---

## Timeline & Milestones

### Week 1: Testing App
**Days 1-2:** PR-0.1 through PR-1.7
- Can generate test data locally
- Validate with Docker Postgres

### Week 1: Infrastructure
**Days 3-4:** PR-2.1 through PR-2.4
- Deploy S3 bucket
- Optional: Deploy RDS
- Test from local machine

### Week 1: Cost Tracking
**Day 5:** PR-3.1 through PR-3.2
- Measure Snowflake costs
- Track infrastructure costs

### Week 2: Competitor Testing
**Days 1-2:** PR-4.1 through PR-4.2
- Setup all competitors
- Run baseline syncs
- Document UX

### Week 2: Continuous Testing
**Days 3-5:** Run continuous load
- 48-hour test
- Collect metrics
- Validate correctness

### Week 2: Analysis
**Days 6-7:** PR-5.1 through PR-5.2
- Fill in RESULTS.md
- Make go/no-go decision
- Document learnings

---

## Success Criteria

After completing all PRs, you should be able to:

✅ Generate configurable test data (1K-100K+ rows)
✅ Run continuous realistic load
✅ Expose Postgres to cloud tools (ngrok)
✅ Deploy S3 bucket for SimplyLoad
✅ Measure actual infrastructure costs
✅ Measure Snowflake efficiency
✅ Compare 4 tools side-by-side
✅ Make data-driven decision on SimplyLoad viability

---

## Notes

- Each PR should take 1-3 hours
- All PRs independently testable
- Follow .claude/rules.md from main repo (stay minimal)
- Playground is temporary - optimize for learning, not production quality
- Focus on collecting data, not building perfect code

---

## After POC Complete

**If hypothesis validated:**
1. Archive playground repo
2. Use learnings to build SimplyLoad
3. Reference RESULTS.md during development

**If hypothesis invalidated:**
1. Document findings in RESULTS.md
2. Pivot strategy or target different market
3. Saved months of development time
