# SimplyLoad POC - Testing & Comparison Plan

**Goal:** Validate hypothesis that SimplyLoad can replicate data to Snowflake cheaper than Fivetran with similar UX.

**Last Updated:** 2025-12-31

---

## Overview

### What We're Testing

1. **Cost comparison:** SimplyLoad vs Fivetran vs Airbyte
2. **Feature parity:** Can we match the core experience?
3. **Infrastructure costs:** What will SimplyLoad actually cost to run?
4. **Data quality:** Do we maintain correctness (especially soft deletes)?

### Testing Strategy

```
Sample App (Postgres)
    ↓
    ├─→ Fivetran → Snowflake
    ├─→ Airbyte → Snowflake
    └─→ SimplyLoad → Snowflake
```

**Measure:** Setup time, sync latency, correctness, cost per 1M rows

---

## Project Structure

**Separate playground from main project:**

```
simplyload/                    # Main project (this repo)
├── src/
├── tests/
├── PROJECT_PLAN.md
└── ...

simplyload-playground/         # Separate testing repo
├── README.md
├── testing-app/               # Sample data generator
│   ├── docker-compose.yml
│   ├── schema.sql
│   ├── src/
│   └── scripts/
└── infra/                     # Pulumi infrastructure
    ├── __main__.py
    ├── Pulumi.yaml
    └── README.md
```

**Keep playground separate:** Easier to destroy, won't clutter main repo, different lifecycle.

---

## Part 1: Sample Testing Application

### Database Schema (Lightweight)

**Simulates a SaaS app with realistic patterns:**

```sql
-- Users (slow changing, ~1K rows)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP NULL
);

-- Orders (medium volume, ~5K rows)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total_amount DECIMAL(10, 2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Events (high volume, ~20K rows)
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    event_type VARCHAR(100),
    properties JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Add updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER orders_updated_at BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Indexes
CREATE INDEX idx_users_updated_at ON users(updated_at);
CREATE INDEX idx_orders_updated_at ON orders(updated_at);
CREATE INDEX idx_events_created_at ON events(created_at);
```

### Data Generation Scenarios (Cost-Optimized)

**Configurable row counts via environment variables:**

```python
# Default: Low cost mode
DEFAULT_CONFIG = {
    "initial_users": 1000,      # 1K users
    "initial_orders": 5000,     # 5K orders
    "initial_events": 20000,    # 20K events
    "continuous_orders_per_hour": 50,   # Much lower for cost
    "continuous_events_per_hour": 200,
    "test_duration_hours": 24,  # 1 day test
}

# High volume mode (optional, costs more)
HIGH_VOLUME_CONFIG = {
    "initial_users": 10000,
    "initial_orders": 50000,
    "initial_events": 200000,
    "continuous_orders_per_hour": 500,
    "continuous_events_per_hour": 2000,
    "test_duration_hours": 168,  # 1 week
}
```

**Estimated costs with default config:**
- Total rows: ~30K initial + ~6K/day continuous
- Fivetran MAR (Monthly Active Rows): ~36K = **Free tier!** (up to 500K)
- Snowflake: Minimal queries = **< $2/month**
- RDS: db.t3.micro = **~$15/month** (or use local Docker: $0)

### Sample App Structure

```
testing-app/
├── README.md
├── docker-compose.yml       # Postgres + optional app container
├── schema.sql
├── .env.example
├── requirements.txt
├── src/
│   ├── config.py            # Load from env vars
│   ├── generator.py         # Data generation
│   └── scenarios.py         # Test scenarios
└── scripts/
    ├── 01_setup.py          # Create schema + initial data
    ├── 02_continuous.py     # Run continuous load
    └── 03_validate.py       # Check row counts
```

### Key Code: Configurable Generator

```python
# config.py
import os
from dataclasses import dataclass

@dataclass
class TestConfig:
    # Configurable via environment
    initial_users: int = int(os.getenv("INITIAL_USERS", "1000"))
    initial_orders: int = int(os.getenv("INITIAL_ORDERS", "5000"))
    initial_events: int = int(os.getenv("INITIAL_EVENTS", "20000"))
    continuous_orders_per_hour: int = int(os.getenv("ORDERS_PER_HOUR", "50"))
    continuous_events_per_hour: int = int(os.getenv("EVENTS_PER_HOUR", "200"))
    test_duration_hours: int = int(os.getenv("TEST_DURATION_HOURS", "24"))

    # Database
    db_url: str = os.getenv("DATABASE_URL", "postgresql://testuser:testpass@localhost:5432/testing_saas")

config = TestConfig()
```

```python
# scenarios.py
import time
import random
from generator import DataGenerator
from config import config

def run_continuous():
    """Run continuous activity based on config"""
    gen = DataGenerator()

    orders_per_second = config.continuous_orders_per_hour / 3600
    events_per_second = config.continuous_events_per_hour / 3600

    print(f"Running continuous load:")
    print(f"  - {config.continuous_orders_per_hour} orders/hour")
    print(f"  - {config.continuous_events_per_hour} events/hour")
    print(f"  - For {config.test_duration_hours} hours")

    end_time = time.time() + (config.test_duration_hours * 3600)

    while time.time() < end_time:
        # Generate orders
        if random.random() < orders_per_second:
            gen.create_order()

        # Generate events
        if random.random() < events_per_second:
            gen.create_event()

        # Occasional updates/deletes (10% of order rate)
        if random.random() < (orders_per_second * 0.1):
            gen.update_random_order()

        if random.random() < (orders_per_second * 0.05):
            gen.soft_delete_random_user()

        time.sleep(1)
```

### Docker Compose (Local Testing)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: testing_saas
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_INITDB_ARGS: "-c wal_level=logical"
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_replication_slots=4"
      - "-c"
      - "max_wal_senders=4"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./schema.sql:/docker-entrypoint-initdb.d/01-schema.sql

  # Optional: Run data generator in container
  generator:
    build: .
    depends_on:
      - postgres
    environment:
      DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testing_saas
      INITIAL_USERS: 1000
      INITIAL_ORDERS: 5000
      INITIAL_EVENTS: 20000
    command: python scripts/02_continuous.py
    profiles:
      - continuous  # Only start when explicitly requested

volumes:
  postgres_data:
```

**Usage:**
```bash
# Just Postgres (for manual testing)
docker-compose up -d postgres

# With continuous data generation
docker-compose --profile continuous up -d
```

---

## Part 2: Infrastructure with Pulumi

### Why Pulumi?
- ✓ Use Python (matches project language)
- ✓ Easy to script and customize
- ✓ Good for POC/experimentation
- ✓ Simple tear down when done

### Infrastructure Components

**Minimal setup for cost-effectiveness:**

```
Option A: Cloud (RDS) - $15-20/month
├── RDS Postgres (db.t3.micro)
├── Security group (allows your IP)
└── S3 bucket (for SimplyLoad)

Option B: Local (Docker) - $0/month
├── Docker Postgres (local)
└── S3 bucket only ($0.25/month)
```

**Recommendation:** Start with **Option B (local)** to minimize costs, upgrade to RDS if you need to test from multiple locations or run longer tests.

### Pulumi Project Structure

```
infra/
├── README.md                # Setup instructions
├── Pulumi.yaml              # Project config
├── Pulumi.dev.yaml          # Dev stack config (gitignored)
├── requirements.txt         # pulumi, pulumi-aws
├── __main__.py              # Main infrastructure code
└── components/
    ├── s3.py                # S3 bucket for SimplyLoad
    └── rds.py               # Optional RDS (for cloud testing)
```

### Minimal Pulumi Code

```python
# __main__.py - Minimal setup
import pulumi
import pulumi_aws as aws

config = pulumi.Config()
enable_rds = config.get_bool("enable_rds") or False  # Default: local only

# 1. S3 bucket for SimplyLoad (always needed)
bucket = aws.s3.Bucket("simplyload-testing",
    acl="private",
    server_side_encryption_configuration={
        "rule": {
            "apply_server_side_encryption_by_default": {
                "sse_algorithm": "AES256"
            }
        }
    },
    lifecycle_rules=[{
        "enabled": True,
        "expiration": {"days": 7}  # Auto-cleanup
    }],
    tags={"project": "simplyload-poc"})

# Export bucket name
pulumi.export("s3_bucket", bucket.id)
pulumi.export("s3_bucket_arn", bucket.arn)

# 2. Optional: RDS Postgres (only if enabled)
if enable_rds:
    # Use default VPC for simplicity
    default_vpc = aws.ec2.get_vpc(default=True)

    # Security group: Allow Postgres from your IP
    sg = aws.ec2.SecurityGroup("postgres-sg",
        vpc_id=default_vpc.id,
        description="Allow Postgres access for testing",
        ingress=[{
            "protocol": "tcp",
            "from_port": 5432,
            "to_port": 5432,
            "cidr_blocks": [config.require("allowed_ip_cidr")],  # Your IP
        }],
        egress=[{
            "protocol": "-1",
            "from_port": 0,
            "to_port": 0,
            "cidr_blocks": ["0.0.0.0/0"],
        }],
        tags={"project": "simplyload-poc"})

    # RDS instance (smallest/cheapest)
    rds = aws.rds.Instance("testing-postgres",
        allocated_storage=20,
        engine="postgres",
        engine_version="15",
        instance_class="db.t3.micro",  # ~$15/month
        db_name="testing_saas",
        username="testuser",
        password=config.require_secret("db_password"),
        vpc_security_group_ids=[sg.id],
        publicly_accessible=True,
        skip_final_snapshot=True,
        backup_retention_period=0,  # No backups (save cost)
        enabled_cloudwatch_logs_exports=[],  # No logs (save cost)
        # Enable logical replication
        parameter_group_name=create_parameter_group(),
        tags={"project": "simplyload-poc"})

    pulumi.export("rds_endpoint", rds.endpoint)
    pulumi.export("rds_connection_string",
        pulumi.Output.concat(
            "postgresql://testuser:",
            config.require_secret("db_password"),
            "@",
            rds.endpoint,
            "/testing_saas"
        ))
```

### Setup Instructions

**infra/README.md:**
```markdown
# SimplyLoad Testing Infrastructure

## Prerequisites
- AWS account
- Pulumi CLI: `curl -fsSL https://get.pulumi.com | sh`
- AWS credentials configured

## Option A: Local Postgres (Free)

1. Install Pulumi:
   ```bash
   curl -fsSL https://get.pulumi.com | sh
   ```

2. Setup stack:
   ```bash
   cd infra
   pulumi login --local  # Use local state (no Pulumi Cloud)
   pulumi stack init dev
   pulumi config set aws:region us-east-1
   ```

3. Deploy S3 bucket only:
   ```bash
   pulumi up
   ```

4. Use Docker Postgres from testing-app:
   ```bash
   cd ../testing-app
   docker-compose up -d postgres
   ```

**Cost:** ~$0.25/month (S3 storage only)

## Option B: Cloud Postgres (RDS)

1. Follow steps 1-2 from Option A

2. Configure RDS:
   ```bash
   pulumi config set enable_rds true
   pulumi config set allowed_ip_cidr "YOUR_IP/32"  # Your public IP
   pulumi config set --secret db_password YourStrongPassword123
   ```

3. Deploy:
   ```bash
   pulumi up
   ```

4. Get connection string:
   ```bash
   pulumi stack output rds_connection_string
   ```

**Cost:** ~$15-20/month

## Cleanup

When done testing:
```bash
pulumi destroy
```
```

---

## Part 3: Testing Protocol (Cost-Optimized)

### Setup Phase (3-4 days)

**Day 1: Infrastructure**
- [ ] Create playground repo: `simplyload-playground/`
- [ ] Setup Pulumi (local mode, S3 only)
- [ ] Start local Postgres with Docker
- [ ] Generate small initial dataset (1K users, 5K orders, 20K events)
- **Cost so far:** $0

**Day 2: Snowflake**
- [ ] Sign up for Snowflake trial (30 days free, $400 credit)
- [ ] Create XS warehouse (auto-suspend 60s)
- [ ] Create schemas: `fivetran`, `airbyte`, `simplyload`
- [ ] Create read-only user for validation queries
- **Cost:** $0 (trial credits)

**Day 3: Fivetran**
- [ ] Sign up for Fivetran free tier (500K MAR/month free)
- [ ] Connect Postgres (local, use ngrok or expose port)
- [ ] Select 3 tables: users, orders, events
- [ ] Run initial sync
- [ ] **Track:** Setup time, UX experience
- **Cost:** $0 (under free tier limit)

**Day 4: Airbyte & SimplyLoad**
- [ ] Deploy Airbyte locally (Docker)
- [ ] Configure same 3 tables
- [ ] Setup SimplyLoad (when ready)
- [ ] Validate all row counts match
- **Cost:** $0

### Testing Phase (24-48 hours)

**Continuous load test (1-2 days):**
- Run `02_continuous.py` with default config
- 50 orders/hour = 1,200 orders/day
- 200 events/hour = 4,800 events/day
- Total new rows: ~6K/day
- **Fivetran cost:** Still $0 (way under 500K limit)

**Validation queries (run every 6 hours):**
```sql
-- Check row counts
SELECT
    'source' as location,
    (SELECT COUNT(*) FROM postgres.users) as users,
    (SELECT COUNT(*) FROM postgres.orders) as orders,
    (SELECT COUNT(*) FROM postgres.events) as events
UNION ALL
SELECT
    'fivetran' as location,
    (SELECT COUNT(*) FROM fivetran.users) as users,
    (SELECT COUNT(*) FROM fivetran.orders) as orders,
    (SELECT COUNT(*) FROM fivetran.events) as events
UNION ALL
SELECT
    'simplyload' as location,
    (SELECT COUNT(*) FROM simplyload.users) as users,
    (SELECT COUNT(*) FROM simplyload.orders) as orders,
    (SELECT COUNT(*) FROM simplyload.events) as events;

-- Check soft deletes
SELECT COUNT(*) FROM simplyload.users WHERE _fivetran_deleted = true;
```

### Cost Tracking Spreadsheet

```
| Component | Daily Cost | Notes |
|-----------|------------|-------|
| Postgres (Docker) | $0 | Local |
| S3 | $0.01 | <1GB data |
| Snowflake | $0.50 | Trial credits |
| Fivetran | $0 | Free tier |
| Airbyte | $0 | Local Docker |
| SimplyLoad compute | $0.50 | EC2 t4g.micro (if needed) |
| **Total/day** | **~$1** | |
| **Total 7-day test** | **~$7** | |
```

---

## Part 4: Quick Start Commands

### 1. Create Playground Repo

```bash
# Outside simplyload repo
mkdir simplyload-playground
cd simplyload-playground
git init

mkdir -p testing-app infra
```

### 2. Setup Infrastructure

```bash
cd infra

# Install Pulumi
curl -fsSL https://get.pulumi.com | sh

# Initialize
pulumi login --local  # No Pulumi Cloud account needed
pulumi stack init dev
pulumi config set aws:region us-east-1

# Deploy (S3 only)
pulumi up -y

# Get S3 bucket name
export SIMPLYLOAD_BUCKET=$(pulumi stack output s3_bucket)
```

### 3. Start Testing App

```bash
cd ../testing-app

# Start Postgres
docker-compose up -d postgres

# Setup schema and initial data
pip install -r requirements.txt
python scripts/01_setup.py

# Start continuous generation (optional)
python scripts/02_continuous.py
```

### 4. Cleanup

```bash
# Stop testing app
cd testing-app
docker-compose down -v

# Destroy infrastructure
cd ../infra
pulumi destroy -y
```

---

## Part 5: Expected Costs (Per Month)

### Minimal Setup (Recommended for POC)
- **Postgres:** $0 (Docker local)
- **S3:** $0.25 (storage)
- **Snowflake:** $0 (trial) → $8/month after (XS warehouse, 1hr/day)
- **Fivetran:** $0 (free tier up to 500K MAR)
- **SimplyLoad:** $0 (local) or $5/month (t4g.micro EC2)
- **Total:** **< $10/month** (or $0 during trial period)

### If Scaling Up (Higher Volume)
- **Postgres RDS:** $15/month (db.t3.micro)
- **S3:** $1/month (10GB)
- **Snowflake:** $15/month (more usage)
- **Fivetran:** $100+/month (if exceed free tier)
- **SimplyLoad:** $10-15/month
- **Total:** **~$40/month**

---

## Success Criteria

SimplyLoad is viable if:
- ✓ **Setup time:** < 10 minutes (vs Fivetran ~5 min)
- ✓ **Correctness:** 100% row count match, soft deletes work
- ✓ **Cost:** 50-70% cheaper than Fivetran at scale
- ✓ **Latency:** < 5 minutes for polling mode

---

## Next Steps

1. **Create playground repo** (separate from simplyload)
2. **Start with local setup** (Docker Postgres + Pulumi S3)
3. **Build testing app** (simple Python generator)
4. **Test Fivetran first** (learn from the best)
5. **Implement SimplyLoad** (following PROJECT_PLAN.md)
6. **Compare results** (cost, UX, correctness)

**Estimated time:** 2-3 weeks to complete full POC
**Estimated cost:** < $30 total (mostly during trial periods)
