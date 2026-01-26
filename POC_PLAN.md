# SimplyLoad POC - Testing & Comparison Plan

**Goal:** Validate hypothesis that SimplyLoad can replicate data to Snowflake cheaper than Fivetran with similar UX.

**Last Updated:** 2025-12-31

---

## Overview

### What We're Testing

1. **Cost comparison:** SimplyLoad vs Fivetran vs Rivery vs Airbyte
2. **Feature parity:** Can we match the core experience?
3. **Infrastructure costs:** What will SimplyLoad actually cost to run?
4. **Data quality:** Do we maintain correctness (especially soft deletes)?

### Testing Strategy

```
Sample App (Postgres)
    ↓
    ├─→ Fivetran → Snowflake (Commercial, MAR pricing)
    ├─→ Rivery → Snowflake (Commercial, alternative pricing model)
    ├─→ Airbyte → Snowflake (Open source, self-hosted)
    └─→ SimplyLoad → Snowflake (Our solution)
```

**Measure:** Setup time, sync latency, correctness, cost per 1M rows

### Tools Being Compared

| Tool | Type | Pricing Model | Free Tier | Best For |
|------|------|---------------|-----------|----------|
| **Fivetran** | Commercial SaaS | MAR (Monthly Active Rows) | 500K MAR free | Benchmark (market leader) |
| **Rivery** | Commercial SaaS | Platform + Data volume | 14-day trial | Alternative pricing model |
| **Airbyte Cloud** | Managed SaaS | Credits-based | 14-day trial + $1000 credits | Managed ETL alternative |
| **SimplyLoad** | Our Solution | TBD (compute + storage) | N/A | What we're building |

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

### Networking: Local Postgres + Cloud Snowflake

**Question:** Will local Docker Postgres work with cloud-based Snowflake?

**Answer:** Yes, but with different approaches per tool:

| Tool | Where It Runs | Access to Local Postgres | Access to Cloud Snowflake |
|------|---------------|--------------------------|---------------------------|
| **SimplyLoad** | Your machine | ✓ Direct (localhost:5432) | ✓ Direct (cloud endpoint) |
| **Fivetran** | Fivetran's cloud | ⚠️ Needs public access (ngrok/SSH tunnel) | ✓ Direct |
| **Rivery** | Rivery's cloud | ⚠️ Needs public access (ngrok/SSH tunnel) | ✓ Direct |
| **Airbyte Cloud** | Airbyte's cloud | ⚠️ Needs public access (ngrok/SSH tunnel) | ✓ Direct |

**Solutions for exposing local Postgres to cloud ETL tools:**

**Option 1: ngrok (Easiest for testing)**
```bash
# Install ngrok
brew install ngrok  # or download from ngrok.com

# Expose local Postgres
ngrok tcp 5432

# Use the ngrok URL in Fivetran/Rivery/Airbyte Cloud
# Example: tcp://0.tcp.ngrok.io:12345
```

**Option 2: SSH Tunnel**
- Most tools support SSH tunnel to localhost
- More secure than direct exposure
- Requires SSH server accessible from internet

**Option 3: Use RDS instead**
- Deploy Postgres to RDS (publicly accessible)
- All tools can connect directly
- Costs ~$15/month but simpler networking

**Recommendation:** Use **ngrok** for POC testing (free, simple), or upgrade to **RDS** if ngrok is unreliable.

---

### Setup Phase (3-4 days)

**Day 1: Infrastructure**
- [ ] Create playground repo: `simplyload-playground/`
- [ ] Setup Pulumi (local mode, S3 only)
- [ ] Start local Postgres with Docker
- [ ] Generate small initial dataset (1K users, 5K orders, 20K events)
- [ ] Setup ngrok for Postgres exposure (if testing cloud ETL tools)
- **Cost so far:** $0

**Day 2: Snowflake**
- [ ] Sign up for Snowflake trial (30 days free, $400 credit)
- [ ] Create XS warehouse (auto-suspend 60s)
- [ ] Create schemas: `fivetran`, `rivery`, `airbyte`, `simplyload`
- [ ] Create read-only user for validation queries
- **Cost:** $0 (trial credits)

**Day 3: Cloud ETL Tools**
- [ ] Sign up for Fivetran free tier (500K MAR/month)
- [ ] Sign up for Rivery trial (14 days)
- [ ] Sign up for Airbyte Cloud trial ($1000 credits)
- [ ] Connect each to Postgres via ngrok URL
- [ ] Configure 3 tables: users, orders, events
- [ ] Run initial syncs
- [ ] **Track:** Setup time, UX experience for each
- **Cost:** $0 (all under trial/free tier)

**Day 4: SimplyLoad**
- [ ] Setup SimplyLoad (when ready)
- [ ] Connect to local Postgres (direct) and Snowflake (cloud)
- [ ] Configure same 3 tables
- [ ] Run initial sync
- [ ] Validate all row counts match across all tools
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

### Cost Measurement Methodology

**CRITICAL: Understanding Who Pays What**

All ETL tools (Fivetran, Rivery, Airbyte Cloud, SimplyLoad) connect to the **customer's Snowflake account**. The customer receives the Snowflake bill directly.

**Who Pays Snowflake Warehouse Costs?**

| Tool | Snowflake Costs |
|------|-----------------|
| Fivetran | Customer pays Snowflake directly |
| Rivery | Customer pays Snowflake directly |
| Airbyte Cloud | Customer pays Snowflake directly |
| SimplyLoad | Customer pays Snowflake directly |

**Therefore:** Snowflake costs are roughly the same for all tools (assuming equal efficiency) and should NOT be included when comparing service costs.

---

### What to Measure in POC

**Track TWO separate cost categories:**

#### 1. SimplyLoad Infrastructure Costs (What YOU Pay)

```python
# Your operating costs - this determines your pricing
simplyload_infrastructure = {
    "ec2": track_compute_cost(),     # OR Lambda/container costs
    "s3": track_storage_cost(),      # Parquet files
    # DO NOT INCLUDE Snowflake - customer pays that!
}

# Unit economics
cost_per_million_rows = (total_infra_cost / rows_synced) * 1_000_000

# Target: < $20-30 per million rows
# Allows 3x margin = $60-90 price
# Still much cheaper than Fivetran ($180 for 1M MAR)
```

#### 2. Snowflake Efficiency (Competitive Advantage)

```sql
-- Track Snowflake warehouse usage for comparison
SELECT
    warehouse_name,
    SUM(execution_time) / 1000 / 60 as minutes_used,
    COUNT(*) as query_count
FROM snowflake.account_usage.query_history
WHERE warehouse_name IN ('simplyload_wh', 'fivetran_wh')
    AND start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name;

-- If SimplyLoad causes LESS warehouse usage than Fivetran
-- → Additional value proposition!
-- If MORE → Hurts total customer cost (bad!)
```

**Why track Snowflake separately?**
- If your efficiency is BETTER → marketing advantage
- If your efficiency is WORSE → you're hurting customer's total cost
- If your efficiency is EQUAL → costs cancel out, compare service fees only

---

### Correct Cost Comparison Matrix

**Customer's Total Cost = Service Fee + Snowflake Costs**

| Volume | Fivetran Service | SimplyLoad Service | Customer's Snowflake (Same) | Total: Fivetran | Total: SimplyLoad | Savings |
|--------|------------------|-------------------|----------------------------|----------------|------------------|---------|
| 1M rows | $180 | $57 | $100 | $280 | $157 | **44%** |
| 5M rows | $500 | $150 | $250 | $750 | $400 | **47%** |
| 10M rows | $900 | $250 | $400 | $1,300 | $650 | **50%** |

*Assumptions: SimplyLoad infra cost $19/month (1M rows), 3x margin = $57; Snowflake efficiency equal*

**Key Insight:** Even with equal Snowflake efficiency, SimplyLoad can be 40-50% cheaper by having lower infrastructure + margin costs.

---

### POC Cost Tracking (Daily)

```
| Component | Who Pays | Daily Cost | Notes |
|-----------|----------|------------|-------|
| **SimplyLoad Infrastructure:** |
| Postgres (Docker) | You (for testing) | $0 | Local |
| S3 | You (operating cost) | $0.01 | <1GB data |
| EC2 (if used) | You (operating cost) | $0.14 | t4g.micro |
| **Customer's Snowflake:** |
| Warehouse (all tools) | Customer | $0.50 | Same for all tools |
| **Service Fees (in production):** |
| Fivetran | Customer | $0 | Free tier in POC |
| Rivery | Customer | $0 | Trial in POC |
| Airbyte Cloud | Customer | $0 | Trial in POC |
| SimplyLoad | Customer | $0 | Testing phase |
| **POC Total/day** | | **~$0.65** | Mostly your infra |
| **7-day test** | | **~$5** | |
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

## Part 5: Cost Analysis & Viability

### Production Cost Breakdown (1M Rows/Month Example)

**SimplyLoad Operating Costs (What You Pay):**
```
EC2 t4g.small (24/7):    $12/month
S3 storage (~10GB):       $2/month
Data transfer:            $1/month
-----------------------------------------
Total Infrastructure:    $15/month

Add 3x margin:           $45/month  ← Your service price
```

**Competitor Service Fees:**
```
Fivetran (1M MAR):      $180/month
Rivery (1M rows):       $249/month
Airbyte Cloud:          $150/month (estimated)
```

**Customer's Total Cost (Including Snowflake):**
```
                        Service Fee    Snowflake    Total
Fivetran:               $180          $100         $280
Rivery:                 $249          $100         $349
Airbyte Cloud:          $150          $100         $250
SimplyLoad:             $45           $100         $145

SimplyLoad saves:       48% vs Fivetran
                        58% vs Rivery
                        42% vs Airbyte Cloud
```

*Note: Snowflake costs (~$100/month for XS warehouse, 50 hrs) are roughly equal across all tools assuming similar efficiency*

---

### Scaling Economics

| Monthly Volume | SimplyLoad Infra | SimplyLoad Price (3x) | Fivetran Price | Customer Saves | Savings % |
|----------------|------------------|----------------------|----------------|----------------|-----------|
| 500K rows | $10 | $30 | $0 (free tier) | -$30 | ❌ |
| 1M rows | $15 | $45 | $180 | $135 | **75%** |
| 5M rows | $50 | $150 | $500 | $350 | **70%** |
| 10M rows | $80 | $240 | $900 | $660 | **73%** |
| 50M rows | $300 | $900 | $3,500 | $2,600 | **74%** |

**Key Insights:**
- Not competitive below 500K rows (Fivetran free tier)
- Highly competitive at 1M+ rows (70%+ savings)
- Margins improve at scale

---

### POC Phase Costs (Testing Period)

**Week 1-2 Testing (~$5-15 total):**
```
Your Infrastructure:
- S3 storage (5GB):              $0.12
- Compute options:
  - Local Docker:                $0.00 (recommended for POC)
  - EC2 t4g.micro:               $3.36 (if testing deployment)
  - ECS Fargate:                 $5.00 (if testing container deploy)
  - Lambda:                      $1.00 (if testing serverless)

Snowflake (customer's):
- Trial credits:                 $0.00
- OR: XS warehouse (2 hrs):      $4.00

Service Trials:
- Fivetran (free tier):          $0.00
- Rivery (trial):                $0.00
- Airbyte Cloud (trial):         $0.00

Total Cost: ~$5-15 depending on setup
```

**Note on Compute Choice:**
- **POC:** Use local Docker ($0 cost)
- **Production:** ECS Fargate recommended (no server management, $15-20/month)
- **At Scale:** Multi-tenant architecture (one service handles all customers)
- Cost estimates in this doc use EC2 as baseline, but any compute option works

---

## Success Criteria

### SimplyLoad is Viable If:

✅ **Infrastructure Cost:** < $30 per million rows
   - Allows 3x margin while staying 60%+ cheaper than competitors
   - Measured: EC2 + S3 only (not including Snowflake)

✅ **Snowflake Efficiency:** Equal or better than Fivetran
   - Measured: Warehouse minutes per 1K rows synced
   - If worse → hurts customer's total cost
   - If better → additional competitive advantage

✅ **Setup Time:** < 10 minutes
   - Close to Fivetran's ~5 minute experience
   - Measured: Time from credentials to first sync complete

✅ **Correctness:** 100% accuracy
   - Row counts match source exactly
   - Soft deletes work (_fivetran_deleted column)
   - No data loss on failures/restarts

✅ **Latency:** < 5 minutes (polling mode)
   - Measured: Time from source change to Snowflake availability
   - Competitive with Fivetran's sync frequency

### Decision Matrix

| Outcome | Action |
|---------|--------|
| **Infrastructure < $30/1M rows AND 60%+ cheaper** | ✅ Build SimplyLoad - hypothesis validated |
| **Infrastructure $30-50/1M rows, 40-60% cheaper** | ⚠️ Build with caution - smaller margins |
| **Infrastructure > $50/1M rows OR < 40% cheaper** | ❌ Pivot strategy or target different market |
| **Snowflake usage 2x worse than Fivetran** | ❌ Optimize or reconsider - hurts customer cost |

---

## Next Steps

1. **Create playground repo** (separate from simplyload)
2. **Follow PLAYGROUND_PLAN.md** (22 PRs, 5 phases)
3. **Build testing app** (Week 1: data generator + infra)
4. **Test competitors** (Week 2: Fivetran, Rivery, Airbyte Cloud)
5. **Analyze results** (Cost, UX, correctness comparison)
6. **Make go/no-go decision** (Build SimplyLoad if validated)

**Estimated time:** 1-2 weeks to complete full POC
**Estimated cost:** < $15 total (mostly trial periods)

**Detailed implementation plan:** See `PLAYGROUND_PLAN.md` for breakdown of all PRs
