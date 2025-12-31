# Customer Setup Documentation

This document outlines the instructions we'll provide to customers and what we need to build in the UI.

**Last Updated:** 2025-12-28

---

## 1. Snowflake Setup (Required for All Customers)

### What Customer Does

**Step 1: Create dedicated user and role**
```sql
-- Run in Snowflake worksheet as ACCOUNTADMIN

-- Create role for SimplyLoad
CREATE ROLE simplyload_role;

-- Create user for SimplyLoad
CREATE USER simplyload_user
  PASSWORD = '<strong-password>' -- Customer generates this
  DEFAULT_ROLE = simplyload_role
  DEFAULT_WAREHOUSE = simplyload_wh;

-- Or using key-pair authentication (recommended):
CREATE USER simplyload_user
  RSA_PUBLIC_KEY = '<public-key>' -- We provide this in UI
  DEFAULT_ROLE = simplyload_role
  DEFAULT_WAREHOUSE = simplyload_wh;

GRANT ROLE simplyload_role TO USER simplyload_user;
```

**Step 2: Create warehouse (cost-optimized)**
```sql
CREATE WAREHOUSE simplyload_wh WITH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60  -- Suspend after 1 minute of inactivity
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

GRANT USAGE ON WAREHOUSE simplyload_wh TO ROLE simplyload_role;
```

**Step 3: Create database and schema**
```sql
-- Create database for replicated data
CREATE DATABASE raw_data;
CREATE SCHEMA raw_data.simplyload;

-- Grant permissions
GRANT USAGE ON DATABASE raw_data TO ROLE simplyload_role;
GRANT USAGE ON SCHEMA raw_data.simplyload TO ROLE simplyload_role;
GRANT CREATE TABLE ON SCHEMA raw_data.simplyload TO ROLE simplyload_role;
GRANT CREATE STAGE ON SCHEMA raw_data.simplyload TO ROLE simplyload_role;
```

**Step 4: Grant table permissions**
```sql
-- Grant permissions on future tables (for our materialized tables)
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES
  IN SCHEMA raw_data.simplyload TO ROLE simplyload_role;

-- Grant permissions on all existing tables (if any)
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES
  IN SCHEMA raw_data.simplyload TO ROLE simplyload_role;
```

### What They Provide to Us (via UI)

- **Account identifier:** e.g., `xy12345.us-east-1.aws`
- **User:** `simplyload_user`
- **Authentication:**
  - Option A: Password (less secure, easier)
  - Option B: Private key (more secure, recommended)
- **Warehouse:** `simplyload_wh`
- **Database:** `raw_data`
- **Schema:** `simplyload`

### What Our UI Shows

**Setup wizard with copy-paste SQL:**
```
Step 1 of 4: Create Snowflake User and Role
-------------------------------------------
Copy and paste this SQL into your Snowflake worksheet:

[SQL code block with copy button]

✓ Tips:
- You need ACCOUNTADMIN privileges to run this
- The password/key will only be used by SimplyLoad
- This user can only access the database you specify
```

**Connection test:**
```
Step 4 of 4: Test Connection
----------------------------
[Connection form with fields above]

[Test Connection Button]

Status: ✓ Connected successfully
        ✓ Can access warehouse
        ✓ Can access database
        ✓ Can create tables

[Continue to Add Sources Button]
```

---

## 2. PostgreSQL Setup

### 2A. Read-Only Mode (Polling - Default, Simplest)

**What Customer Does:**

```sql
-- Run in Postgres as superuser or database owner

-- Create read-only user
CREATE USER simplyload_reader WITH PASSWORD '<strong-password>';

-- Grant connect
GRANT CONNECT ON DATABASE your_database TO simplyload_reader;

-- Grant schema usage
GRANT USAGE ON SCHEMA public TO simplyload_reader;

-- Grant read access to tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO simplyload_reader;

-- Grant read access to future tables (important!)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO simplyload_reader;

-- If tables have sequences (for cursor columns)
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO simplyload_reader;
```

**Limitations of read-only mode:**
- Cannot detect hard deletes (only soft deletes with `deleted_at` column)
- Requires tables to have `updated_at` or similar timestamp column
- Slightly higher latency (polls every 1-5 minutes)

**What They Provide to Us (via UI):**

- **Host:** e.g., `db.example.com`
- **Port:** `5432` (default)
- **Database:** `production`
- **User:** `simplyload_reader`
- **Password:** `***`
- **SSH Tunnel (optional):** For databases not publicly accessible

### 2B. CDC Mode with Logical Replication (Optional, Advanced)

**When to use:** Customer wants real-time CDC with delete detection

**What Customer Does:**

```sql
-- Step 1: Enable logical replication (requires Postgres restart)
-- Add to postgresql.conf:
-- wal_level = logical
-- max_replication_slots = 4
-- max_wal_senders = 4
-- Then restart Postgres

-- Step 2: Create publication (as superuser)
CREATE PUBLICATION simplyload_pub FOR ALL TABLES;
-- Or for specific tables:
-- CREATE PUBLICATION simplyload_pub FOR TABLE users, orders, products;

-- Step 3: Create replication slot
SELECT pg_create_logical_replication_slot('simplyload_slot', 'pgoutput');

-- Step 4: Grant permissions
CREATE USER simplyload_cdc WITH PASSWORD '<strong-password>';
GRANT CONNECT ON DATABASE your_database TO simplyload_cdc;
GRANT USAGE ON SCHEMA public TO simplyload_cdc;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO simplyload_cdc;

-- Grant permission to read replication slot
GRANT pg_read_all_data TO simplyload_cdc;  -- Postgres 14+
-- Or for older versions:
-- ALTER ROLE simplyload_cdc WITH REPLICATION;
```

**Benefits of CDC mode:**
- Real-time change detection (sub-second latency)
- Detects all operations including hard deletes
- Lower load on source database
- No need for `updated_at` columns

**What They Provide to Us (via UI):**
- Same as read-only mode
- System will auto-detect CDC capability and use it

### What Our UI Shows

**Source setup wizard:**

```
Add PostgreSQL Source
---------------------

Connection Details:
[Host field]
[Port field: 5432]
[Database field]
[Username field]
[Password field]
[SSL Mode dropdown: prefer/require/disable]

[Test Connection Button]

Status: ✓ Connected successfully
        ℹ Mode: Polling (read-only)
        ℹ Can replicate: INSERT, UPDATE
        ⚠ Cannot detect: Hard deletes

Want real-time CDC with delete detection?
[View CDC Setup Instructions]

[Continue Button]
```

**Table selection:**
```
Select Tables to Replicate
---------------------------

Database: production

✓ public.users (10,245 rows)
  Cursor: updated_at ✓
  Primary key: id ✓

✓ public.orders (125,839 rows)
  Cursor: updated_at ✓
  Primary key: id ✓

☐ public.sessions (2.1M rows)
  ⚠ No updated_at column found
  [Configure custom cursor]

[Select All] [Deselect All]

Initial Load:
○ Full historical sync (recommended)
○ Start from now (faster setup)

[Continue Button]
```

---

## 3. Stripe Setup

### What Customer Does

1. Go to Stripe Dashboard
2. Navigate to Developers → API Keys
3. Create restricted key (recommended) OR use secret key
4. If using restricted key, grant read permissions:
   - Customers: Read
   - Charges: Read
   - Invoices: Read
   - Subscriptions: Read
   - Products: Read
   - Prices: Read
   - Payment Methods: Read

### What They Provide to Us (via UI)

- **API Key:** `sk_live_...` or `rk_live_...`

### What Our UI Shows

```
Add Stripe Source
-----------------

API Key: [input field]
         [Test mode / Live mode auto-detected]

[Test Connection Button]

Status: ✓ Connected successfully
        ✓ API key is valid
        ℹ Mode: Live
        ℹ Account: acct_1ABC123 (Example Corp)

Select Resources to Sync:
✓ Customers
✓ Charges
✓ Subscriptions
✓ Invoices
☐ Payment Intents
☐ Disputes

Sync Frequency: Every 5 minutes ▼

[Continue Button]
```

---

## 4. Salesforce Setup

### What Customer Does

**Option A: Username/Password + Security Token (Simpler)**
1. Get username and password
2. Get security token:
   - Go to Settings → Reset Security Token
   - Token will be emailed

**Option B: OAuth (More Secure)**
1. We provide OAuth flow
2. Customer clicks "Connect Salesforce"
3. Logs in and approves permissions

### What They Provide to Us (via UI)

**Option A:**
- Username
- Password
- Security Token
- Environment (Production / Sandbox)

**Option B:**
- Just clicks OAuth button

### What Our UI Shows

```
Add Salesforce Source
---------------------

Environment:
○ Production (login.salesforce.com)
○ Sandbox (test.salesforce.com)

Authentication:
○ OAuth (recommended)
  [Connect with Salesforce Button]

○ Username/Password
  Username: [input]
  Password: [input]
  Security Token: [input]

[Test Connection Button]

Status: ✓ Connected successfully
        ℹ Org: Example Corp (00D...)
        ℹ User: admin@example.com

Select Objects to Sync:
✓ Account
✓ Contact
✓ Opportunity
✓ Lead
☐ Case
☐ Custom Objects (15 available)

[Continue Button]
```

---

## 5. S3 Setup (Internal - Customer Doesn't Configure)

**For MVP:** We manage S3 bucket
- Service-owned AWS account
- One bucket: `simplyload-prod-data`
- Partitioned by customer: `s3://bucket/customer_<id>/source/table/`
- Encrypted with SSE-S3
- 7-day lifecycle policy

**Customer never sees or configures this.**

**Future:** Option for customers to BYO S3 bucket (enterprise feature)

---

## 6. Onboarding Flow (Full UI Flow)

### Step 1: Sign Up
```
Welcome to SimplyLoad
---------------------
Email: [input]
Password: [input]
Company: [input]

[Sign Up Button]
```

### Step 2: Connect Snowflake
```
Connect Your Snowflake Account
-------------------------------
[Shows SQL setup wizard from Section 1]
```

### Step 3: Add First Source
```
Add Your First Data Source
---------------------------
[Cards:]
[PostgreSQL] [MySQL] [Stripe] [Salesforce]

[Shows setup wizard based on selection]
```

### Step 4: Start Syncing
```
You're All Set! 🎉
------------------
Syncing:
- PostgreSQL: production (3 tables)
  public.users → raw_data.simplyload.postgres_production_users

First sync: In progress (45% complete)
Est. completion: 2 minutes

[View Dashboard Button]
```

### Dashboard View
```
Dashboard
---------

Sources (1 active)
  PostgreSQL: production
    Last sync: 2 minutes ago
    Status: ✓ Healthy
    Tables: 3
    Rows synced today: 12,482

  [+ Add Source]

Tables
  postgres_production_users
    Last sync: 2 min ago | 10,245 rows | ✓ Syncing

  postgres_production_orders
    Last sync: 2 min ago | 125,839 rows | ✓ Syncing

  postgres_production_products
    Last sync: 2 min ago | 856 rows | ✓ Syncing

Snowflake Usage (Today)
  Warehouse: simplyload_wh
  Runtime: 4.2 minutes
  Est. cost: $0.08

[View in Snowflake Button]
```

---

## 7. Documentation We Need to Write

### For Each Source:
1. **Quick Start Guide** - Copy/paste SQL, 5 minutes to first sync
2. **Permissions Reference** - Detailed breakdown of what we need and why
3. **Troubleshooting** - Common issues and solutions
4. **CDC Setup Guide** (Postgres) - How to enable logical replication

### For Snowflake:
1. **Initial Setup** - Creating user/role/warehouse
2. **Security Best Practices** - Minimal permissions, network policies
3. **Cost Optimization** - Warehouse sizing, auto-suspend settings
4. **Table Structure** - Understanding _fivetran_synced and _fivetran_deleted

### For Platform:
1. **Architecture Overview** - How the system works
2. **Data Freshness** - Sync frequencies, latency expectations
3. **Schema Mapping** - How source types map to Snowflake types

---

## UI Components to Build

### Phase 1 (MVP):
- [ ] Snowflake connection wizard with SQL generator
- [ ] PostgreSQL connection form with test
- [ ] Stripe connection form with test
- [ ] Table selection UI with cursor detection
- [ ] Basic dashboard with sync status
- [ ] Error/warning display for connection issues

### Phase 2:
- [ ] Salesforce OAuth flow
- [ ] Advanced table configuration (custom cursors)
- [ ] Sync history and logs viewer
- [ ] Cost tracking dashboard
- [ ] Schema change notifications

### Phase 3:
- [ ] PostgreSQL CDC setup assistant
- [ ] SSH tunnel configuration
- [ ] Custom transformation rules
- [ ] Alerting configuration
- [ ] Usage analytics
