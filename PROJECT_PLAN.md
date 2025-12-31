# SimplyLoad - Project Implementation Plan

## Overview
Snowflake-native ETL platform with CDC support, micro-batching, and cost-efficient replication.

**Last Updated:** 2025-12-28

## Key Architectural Decisions

### Language: Python Monolith
- **Primary language:** Python
- **Rationale:** Best ecosystem for data tools (Parquet, Snowflake, API SDKs)
- **Postgres CDC:** Start with `psycopg2` logical replication
- **Future option:** Extract Debezium to Java microservice if CDC requirements grow
- **Trade-off:** Faster development, simpler deployment; slightly less robust CDC than Debezium initially

### Snowflake Connection Model: Direct Connection
- **Approach:** Service connects directly to customer's Snowflake (Fivetran-style)
- **Rationale:** "Just make it work" - prioritize ease of onboarding
- **Customer provides:** Snowflake credentials with minimal permissions
- **Authentication:** Key-pair authentication (more secure than password)
- **Future option:** Add "customer-managed" mode (BYO S3 + SQL scripts) for enterprises

---

## Phase 0: Project Setup & Foundation

### PR-0.1: Repository Structure & Tooling
- [ ] Initialize project structure
  - Choose language (Python recommended for Debezium, Snowflake, API integrations)
  - Setup package manager (Poetry/pip)
  - Create basic directory structure: `src/`, `tests/`, `config/`
- [ ] Add configuration management
  - Environment variables
  - Config file structure (YAML/TOML)
- [ ] Setup linting & formatting (black, ruff, mypy)
- [ ] Add .gitignore
- [ ] Create README with project overview

### PR-0.2: Initial Dependencies & Dev Environment
- [ ] Setup Poetry and pyproject.toml
- [ ] Add minimal initial dependencies:
  - `pydantic` - For configuration models (needed in Phase 1)
  - `python-dotenv` - For environment variables
- [ ] Add dev dependencies:
  - `pytest` - Testing framework
  - `black`, `ruff`, `mypy` - Code quality tools
- [ ] Create requirements.txt for later (if needed)
- [ ] Note: Additional dependencies will be added in PRs where they're first used

### PR-0.3: Logging & Observability Foundation
- [ ] Setup structured logging (structlog or similar)
- [ ] Add basic metrics tracking structure
- [ ] Create observability utilities module
- [ ] Add health check endpoint structure

---

## Phase 1: Core Data Models & Abstractions

### PR-1.1: Core Domain Models
- [ ] Define `SourceTable` model
  - Table identifier (source, database, schema, table)
  - Primary key columns
  - Cursor/ordering column
- [ ] Define `ChangeEvent` model
  - Operation type (insert/update/delete)
  - Primary key values
  - Payload data
  - Source metadata (LSN, timestamp)
- [ ] Define `BatchMetadata` model
  - Batch ID
  - Source table
  - Event count
  - Time range
  - S3 location

### PR-1.2: Abstract Interfaces
- [ ] Create `SourceConnector` abstract base class
  - `connect()` method
  - `read_changes()` generator method
  - `get_checkpoint()` method
  - `commit_checkpoint()` method
- [ ] Create `BatchWriter` abstract base class
  - `write_batch()` method (events → Parquet → S3)
  - `get_batch_location()` method
- [ ] Create `DestinationLoader` abstract base class
  - `create_staging_table()` method
  - `create_final_table()` method
  - `load_to_staging()` method
  - `materialize()` method (MERGE)

### PR-1.3: Configuration Models
- [ ] Define `SourceConfig` dataclass
  - Connection parameters
  - Tables to sync
  - CDC vs polling mode settings
- [ ] Define `DestinationConfig` dataclass
  - Snowflake connection details
  - Database/schema/warehouse
  - Stage configuration
- [ ] Define `PipelineConfig` dataclass
  - Batch size thresholds
  - Time window settings
  - Retry policies
  - S3 configuration

---

## Phase 2: S3 & Parquet Layer

### PR-2.1: S3 Client Wrapper
- [ ] Create `S3Client` class
  - Initialize with credentials & bucket
  - `upload_file()` method with error handling
  - `list_files()` method
  - `delete_files()` method (for retention management)
  - Add encryption settings (SSE-S3)
- [ ] Add unit tests with moto (AWS mocking)

### PR-2.2: Parquet Writer
- [ ] Create `ParquetBatchWriter` implementing `BatchWriter`
  - Convert `ChangeEvent` list to PyArrow table
  - Define schema with metadata columns:
    - `_op` (operation)
    - `_source_lsn` or `_source_cursor`
    - `_ingested_at`
  - Write to local temp file
  - Upload to S3 with partitioned path:
    - `s3://bucket/customer_id/source/table/date_hour/batch_id.parquet`
- [ ] Add unit tests

### PR-2.3: Batch Buffer & Flushing Logic
- [ ] Create `BatchBuffer` class
  - In-memory list of events
  - Flush triggers:
    - Size threshold (e.g., 10,000 events)
    - Time window (e.g., 60 seconds)
  - `add_event()` method
  - `should_flush()` method
  - `flush()` method (calls BatchWriter)
- [ ] Add unit tests for flush logic

---

## Phase 3: Checkpoint & State Management

### PR-3.1: Checkpoint Storage Interface
- [ ] Create `CheckpointStore` abstract class
  - `get_checkpoint(source_table)` → dict
  - `save_checkpoint(source_table, checkpoint_data)` → None
  - `list_checkpoints()` → list
- [ ] Document checkpoint data structure:
  - `last_lsn` or `last_cursor`
  - `last_batch_id`
  - `last_committed_at`
  - `parquet_files` (list for idempotency)

### PR-3.2: Local File Checkpoint Implementation
- [ ] Implement `FileCheckpointStore`
  - Store checkpoints as JSON files
  - One file per source table
  - Atomic writes (write to temp, rename)
  - Load on startup
- [ ] Add unit tests

### PR-3.3: Database Checkpoint Implementation (Optional)
- [ ] Implement `DatabaseCheckpointStore`
  - Use SQLite or Postgres for state
  - Create schema for checkpoints table
  - Thread-safe operations
- [ ] Add unit tests
- [ ] Make checkpoint backend configurable

---

## Phase 4: Snowflake Integration

### PR-4.1: Snowflake Connection Manager
- [ ] Create `SnowflakeConnection` class
  - Initialize from config (account, user, password, role, warehouse)
  - Connection pooling
  - `execute_query()` method with error handling
  - `execute_transaction()` context manager
  - Auto-resume warehouse support
- [ ] Add connection tests (require Snowflake credentials)

### PR-4.2: Snowflake Schema Manager
- [ ] Create `SnowflakeSchemaManager` class
  - `create_staging_table(source_table)` method
    - Generate CREATE TRANSIENT TABLE DDL
    - Include _op, _source_ordering, metadata columns
  - `create_final_table(source_table)` method
    - Generate CREATE TABLE DDL
    - Include business columns + _fivetran_synced, _fivetran_deleted
  - `table_exists()` method
  - `get_table_schema()` method
- [ ] Add unit tests with DDL validation

### PR-4.3: Snowflake External Stage Setup
- [ ] Create `SnowflakeStageManager` class
  - `create_external_stage()` method
    - CREATE STAGE with S3 credentials
    - Configure file format (PARQUET)
  - `stage_exists()` method
  - `list_staged_files()` method
- [ ] Add integration tests

### PR-4.4: Snowflake Loader - COPY INTO
- [ ] Create `SnowflakeLoader` class implementing `DestinationLoader`
  - `load_to_staging(batch_metadata)` method
    - Generate COPY INTO statement
    - Reference Parquet files from external stage
    - Use file list for idempotency
    - Handle errors and partial loads
  - Track loaded files to prevent duplicates
- [ ] Add integration tests

### PR-4.5: Snowflake Materializer - MERGE Logic
- [ ] Implement `materialize()` in `SnowflakeLoader`
  - Generate MERGE statement:
    - Match on primary key
    - Use source ordering column for deduplication
    - For INSERT/UPDATE: upsert row, set _fivetran_deleted=false
    - For DELETE: update row, set _fivetran_deleted=true
  - Execute in transaction
- [ ] Create SQL template system for MERGE
- [ ] Add integration tests with sample data

### PR-4.6: Warehouse Auto-Suspend Optimization
- [ ] Add warehouse management utilities
  - Query to check warehouse state
  - Auto-suspend configuration helper
  - Cost tracking metadata (query history)
- [ ] Document best practices for XS warehouse usage

---

## Phase 5: PostgreSQL Source Connector

### PR-5.1: Postgres Connection & Discovery
- [ ] Add `psycopg2-binary` dependency when starting this phase
- [ ] Create `PostgresSourceConnector` implementing `SourceConnector`
  - Initialize with connection string (read-only user)
  - `connect()` method with SSL support
  - `discover_tables()` method
    - Query `information_schema.tables`
    - Detect primary keys from `information_schema.key_column_usage`
    - Detect cursor columns (updated_at, modified_at, etc.)
  - Connection pooling for efficiency
- [ ] Add unit tests with mock database

### PR-5.2: Postgres Polling Mode (Primary Method)
- [ ] Implement polling-based change detection
  - `read_changes()` using cursor column (updated_at/created_at)
  - Query: `SELECT * FROM table WHERE updated_at > last_checkpoint ORDER BY updated_at LIMIT 1000`
  - Generate `ChangeEvent` objects (all as INSERT/UPDATE operations)
  - Track high watermark (last seen cursor value)
  - Handle pagination for large result sets
  - Handle NULL cursor values
  - Detect soft deletes (deleted_at column)
- [ ] Add integration tests with test Postgres DB
- [ ] Document limitations:
  - Cannot detect hard deletes
  - Requires updated_at column
  - 1-5 minute latency (configurable poll interval)

### PR-5.3: Cursor Column Detection & Validation
- [ ] Implement smart cursor detection
  - Look for: updated_at, modified_at, updated_timestamp, last_modified
  - Validate cursor column type (timestamp, integer, etc.)
  - Check for indexes on cursor column (performance)
  - Allow manual cursor override in config
- [ ] Add validation warnings:
  - Warn if no cursor column found
  - Warn if cursor column not indexed
  - Suggest adding updated_at trigger
- [ ] Add tests

### PR-5.4: Postgres Type Mapping
- [ ] Create type mapping from Postgres → Parquet/Snowflake
  - Handle common types (int, varchar, timestamp, json, etc.)
  - Handle arrays
  - Handle NULL values
- [ ] Add comprehensive type tests

### PR-5.5: (Optional - Future Phase) Postgres CDC with Logical Replication
**Note:** This is a future enhancement. Skip for MVP. Add only if customers request real-time CDC.

- [ ] Implement logical replication slot reader
  - Read from pre-created replication slot using `pg_logical_slot_peek_changes()`
  - Parse pgoutput protocol messages
  - Detect INSERT/UPDATE/DELETE including hard deletes
  - Extract LSN for ordering
- [ ] Auto-detect CDC availability
  - Check if replication slot exists
  - Gracefully fall back to polling if not available
- [ ] Document customer requirements in CUSTOMER_SETUP.md:
  - Admin must enable wal_level=logical
  - Admin must create replication slot
  - Grant SELECT on `pg_logical_slot_peek_changes()` function
- [ ] Alternative: If Python CDC proves insufficient, extract Debezium to separate Java process

---

## Phase 6: Stripe Source Connector

### PR-6.1: Stripe API Client Wrapper
- [ ] Create `StripeSourceConnector` implementing `SourceConnector`
  - Initialize with API key
  - Configure resources to sync (customers, charges, subscriptions, etc.)
  - Implement rate limiting
  - Handle pagination
- [ ] Add unit tests with Stripe mock

### PR-6.2: Stripe Incremental Sync
- [ ] Implement `read_changes()` for Stripe
  - Use `created` or `updated` cursor
  - List API with `created[gte]` or `updated[gte]` filter
  - Convert Stripe objects to `ChangeEvent` (all as upserts)
  - Track cursor per resource type
- [ ] Add integration tests with Stripe test mode

### PR-6.3: Stripe Schema Normalization
- [ ] Flatten nested Stripe objects
  - Handle metadata fields
  - Handle expandable objects
  - Create consistent schema per resource type
- [ ] Add tests for complex objects

### PR-6.4: Stripe Checkpoint Management
- [ ] Implement checkpoint tracking per resource
  - Save last cursor value
  - Handle multiple resources independently
  - Resume from checkpoint
- [ ] Add tests

---

## Phase 7: Salesforce Source Connector

### PR-7.1: Salesforce API Client Wrapper
- [ ] Create `SalesforceSourceConnector` implementing `SourceConnector`
  - Initialize with OAuth credentials
  - Use simple-salesforce or similar
  - Configure objects to sync (Account, Contact, Opportunity, etc.)
  - Handle API limits
- [ ] Add unit tests

### PR-7.2: Salesforce SOQL Incremental Sync
- [ ] Implement `read_changes()` for Salesforce
  - Use SystemModstamp or LastModifiedDate
  - SOQL query with WHERE clause for incremental
  - Convert SOQL results to `ChangeEvent`
  - Handle hard deletes (query Recycle Bin via REST API)
- [ ] Add integration tests with Salesforce sandbox

### PR-7.3: Salesforce Schema Discovery
- [ ] Implement schema discovery using Describe API
  - Get object metadata
  - Detect field types
  - Create type mappings to Parquet/Snowflake
- [ ] Add tests

### PR-7.4: Salesforce Checkpoint Management
- [ ] Track last SystemModstamp per object
  - Save in checkpoint store
  - Resume from checkpoint
- [ ] Add tests

---

## Phase 8: Orchestration & Pipeline

### PR-8.1: Single-Table Sync Pipeline
- [ ] Create `TableSyncPipeline` class
  - Takes SourceConnector, BatchWriter, DestinationLoader
  - Main sync loop:
    1. Read changes from source
    2. Add to batch buffer
    3. Flush when threshold met
    4. Load to Snowflake staging
    5. Materialize to final table
    6. Commit checkpoint
  - Handle errors and retries
- [ ] Add integration tests

### PR-8.2: Multi-Table Orchestrator
- [ ] Create `SyncOrchestrator` class
  - Manage multiple `TableSyncPipeline` instances
  - One pipeline per source table
  - Concurrent execution (threading or asyncio)
  - Shared checkpoint store
  - Graceful shutdown handling
- [ ] Add tests

### PR-8.3: Scheduling & Continuous Sync
- [ ] Add scheduling logic
  - Polling interval configuration
  - Continuous mode vs one-time sync
  - Backfill mode (from beginning)
- [ ] Add signal handling (SIGTERM, SIGINT)

### PR-8.4: Error Handling & Retries
- [ ] Implement retry policies
  - Exponential backoff
  - Max retries per operation
  - Dead letter queue for failed batches
- [ ] Add circuit breaker for failing sources
- [ ] Improve error logging and alerting

---

## Phase 9: Configuration & CLI

### PR-9.1: Configuration File Format
- [ ] Define YAML configuration schema
  - Sources section (list of connectors)
  - Destination section (Snowflake config)
  - Pipeline settings (batch size, intervals)
  - S3 settings
- [ ] Create example configuration files
- [ ] Add configuration validation

### PR-9.2: CLI Interface
- [ ] Create CLI using Click or argparse
  - `sync` command - start syncing
  - `discover` command - list available tables
  - `validate` command - check configuration
  - `backfill` command - historical sync
  - `checkpoint` command - view/reset checkpoints
- [ ] Add help documentation

### PR-9.3: Environment-Based Configuration
- [ ] Support environment variable overrides
  - Credentials from env vars
  - Sensitive data handling
  - Precedence: CLI args > env vars > config file
- [ ] Add .env.example file

---

## Phase 10: Monitoring & Operations

### PR-10.1: Metrics Collection
- [ ] Add metrics instrumentation
  - Events processed per table
  - Batch sizes
  - Latency (source read, S3 write, Snowflake load)
  - Error counts
  - Warehouse cost tracking
- [ ] Export metrics (Prometheus format or similar)

### PR-10.2: Health Checks
- [ ] Implement health check endpoint
  - Source connectivity
  - Snowflake connectivity
  - S3 connectivity
  - Last successful sync per table
- [ ] Add HTTP server for health checks (optional)

### PR-10.3: Data Quality Checks
- [ ] Add validation logic
  - Row count reconciliation
  - Primary key uniqueness checks
  - NULL value monitoring
  - Schema drift detection
- [ ] Log validation results

### PR-10.4: S3 Retention Management
- [ ] Implement S3 cleanup job
  - Delete Parquet files older than retention period
  - Configurable retention (7-30 days)
  - Safe cleanup (only after Snowflake load confirmed)
- [ ] Add scheduling for cleanup job

---

## Phase 11: Testing & Hardening

### PR-11.1: Integration Test Suite
- [ ] End-to-end tests for Postgres CDC
  - Insert/update/delete operations
  - Verify final Snowflake state
  - Verify soft deletes
- [ ] End-to-end tests for Stripe
- [ ] End-to-end tests for Salesforce

### PR-11.2: Fault Tolerance Tests
- [ ] Test checkpoint recovery
  - Kill process mid-batch
  - Verify no data loss on restart
  - Verify no duplicates
- [ ] Test S3 upload failures
- [ ] Test Snowflake load failures

### PR-11.3: Performance Testing
- [ ] Benchmark Postgres CDC throughput
- [ ] Benchmark batch writing performance
- [ ] Test with large datasets (millions of rows)
- [ ] Measure Snowflake warehouse costs

### PR-11.4: Security Hardening
- [ ] Audit credential handling
  - No secrets in logs
  - Encrypted S3 storage
  - Secure Snowflake connections
- [ ] Add input validation
- [ ] Security documentation

---

## Phase 12: Documentation & Deployment

### PR-12.1: User Documentation
- [ ] Getting started guide
- [ ] Configuration reference
- [ ] Source connector setup guides
  - Postgres CDC prerequisites
  - Stripe API key setup
  - Salesforce OAuth setup
- [ ] Snowflake setup guide
  - User/role creation
  - Warehouse configuration
  - Permissions required

### PR-12.2: Architecture Documentation
- [ ] System architecture diagrams
- [ ] Data flow documentation
- [ ] Checkpoint and fault tolerance design
- [ ] Cost optimization guide

### PR-12.3: Deployment Guide
- [ ] Docker image creation
- [ ] Kubernetes manifests (optional)
- [ ] AWS ECS/Fargate deployment guide
- [ ] Environment setup instructions

### PR-12.4: Operations Runbook
- [ ] Monitoring and alerting setup
- [ ] Troubleshooting guide
- [ ] Common issues and solutions
- [ ] Backup and recovery procedures

---

## Success Criteria

### Phase 0-3 Complete
- [ ] Project structure established
- [ ] Core abstractions defined and tested
- [ ] S3 and Parquet writing functional
- [ ] Checkpoint system working

### Phase 4-5 Complete (Postgres → Snowflake MVP)
- [ ] Postgres CDC working
- [ ] Snowflake loading and materialization working
- [ ] Can replicate a Postgres table with soft deletes
- [ ] Fault tolerance demonstrated

### Phase 6-7 Complete (Multi-Source Support)
- [ ] Stripe integration working
- [ ] Salesforce integration working
- [ ] Multiple sources can sync concurrently

### Phase 8-12 Complete (Production Ready)
- [ ] Orchestration and scheduling robust
- [ ] Monitoring and observability in place
- [ ] Documentation complete
- [ ] Security hardening done
- [ ] Deployable to production

---

## Notes

- Each PR should be reviewable in 30-60 minutes
- Include tests with each PR
- Update this plan as you learn and adjust
- Mark items complete as you go
- Add new phases/PRs as needed for unforeseen work

---

## Current Status

**Phase:** 0 (Project Setup)
**Next PR:** 0.1 (Repository Structure & Tooling)
**Blockers:** None
