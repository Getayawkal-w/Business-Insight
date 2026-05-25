# Odoo Dual Database CDC + Metabase Replication Runbook

## Overview

This document describes the complete implementation of a near real-time analytics replication architecture for two Odoo PostgreSQL databases into a centralized VPS analytics environment using:

- PostgreSQL logical replication
- Debezium Server
- Node.js CDC ingest service
- Tailscale private networking
- Unified PostgreSQL reporting views
- Metabase

This architecture was implemented and validated successfully with:

- Odoo 11
- PostgreSQL 12.4
- Windows on-premise Odoo server
- Ubuntu VPS
- Metabase on VPS

The final system synchronizes changes from two production Odoo databases into centralized analytics schemas within approximately 1–2 seconds.

---

# Final Architecture

```text
[Windows Odoo Server]
    ├── PostgreSQL 12
    ├── Odoo 11
    ├── Debezium Server Containers
    └── Tailscale Node
              ↓
        Private Tailscale Mesh
              ↓
[VPS]
    ├── Tailscale Node
    ├── Node.js CDC Ingest API
    ├── PostgreSQL Analytics Database
    ├── Unified Reporting Views
    └── Metabase
```

---

# Source Databases

Two active Odoo PostgreSQL databases:

| Database | Orders | Prefix |
|---|---|---|
| top_2018 | ~19,518 | SOV- |
| top_2018_n | ~36,515 | NEW_SOV- |

Important:

- Both databases are structurally identical.
- No overlapping sales references were found.
- POS tables do not exist in these databases.

---

# Final Synced Tables

```text
sale_order
sale_order_line
account_invoice
account_invoice_line
account_move
account_move_line
res_partner
product_product
product_template
stock_picking
stock_move
stock_quant
res_users
res_company
stock_warehouse
sale_locations
product_uom
```

---

# Important Architectural Decisions

## Raw schemas

The VPS analytics database stores replicated data in:

```text
raw_top_2018
raw_top_2018_n
```

These schemas:

- Are replication storage only
- Are NOT exposed directly to Metabase
- Do NOT contain full Odoo constraints
- Only require PRIMARY KEY(id)

## Unified reporting layer

Metabase reads from unified views only:

```text
public.sale_order
public.sale_order_line
...
```

These views combine both source databases safely.

---

# Why Tailscale Was Used

Originally the design used:

```text
Debezium → Public HTTPS → Apache Reverse Proxy → Node API
```

This was replaced with:

```text
Debezium → Tailscale Private Network → Node API
```

Benefits:

- No public CDC endpoint
- No SSL management for CDC
- No Apache requirement for CDC
- Simpler firewall model
- More reliable transport
- Better security
- Easier recovery and replication

---

# VPS Preparation

## Install PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

## Create analytics database

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE odoo_analytics;
```

---

# Create Raw Schemas

```bash
psql -h localhost -U postgres -d odoo_analytics
```

```sql
CREATE SCHEMA raw_top_2018;
CREATE SCHEMA raw_top_2018_n;
```

---

# Initial Snapshot Export

Performed from Windows source server.

## Schema dump

Example:

```cmd
pg_dump -h localhost -U openpg -d top_2018 ^
  -t public.sale_order ^
  --section=pre-data ^
  --no-owner ^
  --no-privileges ^
  -f top_2018_schema.sql
```

## Data dump

```cmd
pg_dump -h localhost -U openpg -d top_2018 ^
  -t public.sale_order ^
  --section=data ^
  --no-owner ^
  --no-privileges ^
  -f top_2018_data.sql
```

Repeat for all required tables and for top_2018_n.

---

# Rewrite Schema Names

Because source schema is `public`, dumps were rewritten to target:

```text
raw_top_2018
raw_top_2018_n
```

A Python rewrite script was used. The full script is included here and must be copied exactly.

Create the script on the VPS:

```bash
mkdir -p /root/odoo_analytics
cd /root/odoo_analytics
nano rewrite_dump_schema.py
```

Paste:

```python
import sys
from pathlib import Path

if len(sys.argv) != 4:
    print("Usage: python3 rewrite_dump_schema.py input.sql output.sql target_schema")
    print("Example: python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018")
    sys.exit(1)

input_file = Path(sys.argv[1])
output_file = Path(sys.argv[2])
target_schema = sys.argv[3]

if not input_file.exists():
    print(f"ERROR: input file does not exist: {input_file}")
    sys.exit(1)

content = input_file.read_text(encoding="utf-8", errors="replace")
content = content.replace("public.", f"{target_schema}.")
content = content.replace("SET search_path = public, pg_catalog;", f"SET search_path = {target_schema}, pg_catalog;")
content = content.replace("SELECT pg_catalog.set_config('search_path', '', false);", f"SET search_path = {target_schema}, pg_catalog;")

prefix = f"""CREATE SCHEMA IF NOT EXISTS {target_schema};
SET search_path = {target_schema}, pg_catalog;
"""

output_file.write_text(prefix + content, encoding="utf-8")
print(f"Created {output_file} targeting schema {target_schema}")
```

Run the script for all four dump files:

```bash
cd /root/odoo_analytics
python3 rewrite_dump_schema.py top_2018_predata.sql top_2018_predata_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_n_predata.sql top_2018_n_predata_raw.sql raw_top_2018_n
python3 rewrite_dump_schema.py top_2018_n_data.sql top_2018_n_data_raw.sql raw_top_2018_n
```

Example:

```bash
python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018
```

---

# Import Into VPS

```bash
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_schema_raw.sql
```

```bash
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_data_raw.sql
```

Repeat for top_2018_n.

---

# Validate Counts

```sql
SELECT COUNT(*) FROM raw_top_2018.sale_order;
SELECT COUNT(*) FROM raw_top_2018_n.sale_order;
```

Final validated counts:

```text
raw_top_2018.sale_order   = 19,518
raw_top_2018_n.sale_order = 36,515
```

---

# Enable PostgreSQL Logical Replication

On Windows PostgreSQL:

Edit:

```text
C:\Program Files\PostgreSQL\12\data\postgresql.conf
```

Set:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Restart PostgreSQL.

Verify:

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

Expected:

```text
logical
10
10
```

---

# Create Replication User

```sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'metabase_sync') THEN
    CREATE ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  ELSE
    ALTER ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  END IF;
END
$$;
```

Grant permissions:

```sql
GRANT CONNECT ON DATABASE top_2018 TO metabase_sync;
GRANT CONNECT ON DATABASE top_2018_n TO metabase_sync;
```

Inside each database:

```sql
GRANT USAGE ON SCHEMA public TO metabase_sync;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_sync;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO metabase_sync;
```

---

# Create Publications

Example for top_2018:

```sql
DROP PUBLICATION IF EXISTS top_2018_metabase_pub;

CREATE PUBLICATION top_2018_metabase_pub FOR TABLE
public.sale_order,
public.sale_order_line,
public.account_invoice,
public.account_invoice_line,
public.account_move,
public.account_move_line,
public.res_partner,
public.product_product,
public.product_template,
public.stock_picking,
public.stock_move,
public.stock_quant,
public.res_users,
public.res_company,
public.stock_warehouse,
public.sale_locations,
public.product_uom;
```

Repeat for top_2018_n.

---

# Create Replication Slots

Critical step.

Performed BEFORE refreshed snapshot.

In top_2018:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'top_2018_metabase_slot',
  'pgoutput'
);
```

In top_2018_n:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'top_2018_n_metabase_slot',
  'pgoutput'
);
```

Verify:

```sql
SELECT
  slot_name,
  slot_type,
  active,
  plugin,
  database
FROM pg_replication_slots;
```

Expected before Debezium:

```text
active = false
plugin = pgoutput
```

---

# Refreshed Snapshot After Slot Creation

Because Odoo remained active during initial import, a second data-only snapshot was required after slot creation.

Steps:

1. Create fresh data-only dumps
2. Truncate VPS raw tables
3. Import refreshed data
4. Start Debezium

This prevents replication gaps.

---

# Add Primary Keys To Raw Tables

Raw tables require:

```sql
PRIMARY KEY(id)
```

Because Node ingest API uses:

```sql
ON CONFLICT(id) DO UPDATE
```

Primary keys were added to all 34 raw tables.

---

# Install Tailscale

Installed on:

- Windows Odoo server
- VPS

Verified connectivity:

```bash
tailscale ip -4
```

All CDC traffic now flows through Tailscale private networking.

---

# Node.js CDC Ingest API

## Location

```text
/opt/odoo-cdc-ingest
```

## Technologies

- Node.js
- Express
- PostgreSQL client
- PM2

## Responsibilities

- Accept Debezium events
- Validate CDC secret
- Route events to proper raw schema
- UPSERT rows
- DELETE rows
- Track sync events/status

---

# Important Node.js Features

## Allowed table whitelist

Only approved tables may sync.

## Source-to-schema mapping

```js
const SOURCE_TO_SCHEMA = {
  top_2018: "raw_top_2018",
  top_2018_n: "raw_top_2018_n"
};
```

## Tailscale binding

Critical change:

```js
app.listen(8088, "0.0.0.0")
```

instead of localhost-only.

---

# CDC Secret Authentication

Debezium authenticates using:

```text
x-cdc-secret
```

Environment variable on VPS:

```bash
CDC_SECRET='YOUR_SECRET'
```

Debezium config:

```properties
debezium.sink.http.headers=x-cdc-secret:YOUR_SECRET
```

Mismatch caused initial 401 Unauthorized errors.

Fix:

- Update PM2 environment
- Restart Node service

---

# Timestamp Conversion Fix

## Problem

Debezium emitted microsecond timestamps such as:

```text
1779366228216843
```

PostgreSQL timestamp columns rejected these values.

## Fixes

Debezium config:

```properties
timestamp.mode=iso_offset
```

Node.js helpers:

```js
microsToTimestamp()
daysToIsoDate()
```

These convert Debezium payloads into PostgreSQL-compatible ISO strings.

---

# Decimal Handling Fix

## Problem

Debezium emitted decimal values as structs:

```json
{"scale":1,"value":"Aqq/"}
```

PostgreSQL numeric columns could not store these.

## Fix

Debezium config:

```properties
debezium.source.decimal.handling.mode=double
```

This simplified numeric payloads into standard floating-point values.

---

# Clearing Stuck Offsets

Malformed events caused Debezium retry loops.

Fix:

1. Stop container
2. Delete offsets.dat
3. Restart container

Paths:

```text
C:\odoo-cdc\data_top_2018\offsets.dat
C:\odoo-cdc\data_top_2018_n\offsets.dat
```

---

# Debezium Configuration

## Folder Structure

```text
C:\odoo-cdc
  ├── top_2018
  ├── top_2018_n
  ├── data_top_2018
  └── data_top_2018_n
```

## Important Debezium Settings

```properties
debezium.source.snapshot.mode=never
```

Because initial snapshots were loaded manually.

```properties
debezium.source.plugin.name=pgoutput
```

```properties
debezium.source.slot.drop.on.stop=false
```

```properties
debezium.source.decimal.handling.mode=double
```

```properties
timestamp.mode=iso_offset
```

```properties
errors.max.retries=-1
```

---

# Debezium Sink URLs

Using Tailscale private IPs:

```properties
debezium.sink.http.url=http://VPS_TAILSCALE_IP:8088/odoo-cdc/top_2018
```

```properties
debezium.sink.http.url=http://VPS_TAILSCALE_IP:8088/odoo-cdc/top_2018_n
```

Apache and public HTTPS were removed from the CDC path.

---

# Running Debezium Containers

Example:

```cmd
docker run -d --name debezium-top-2018 ^
  --restart unless-stopped ^
  -v C:\odoo-cdc\top_2018:/debezium/conf ^
  -v C:\odoo-cdc\data_top_2018:/debezium/data ^
  debezium/server:2.2
```

Repeat for top_2018_n.

---

# Verify Replication Slots Active

```sql
SELECT
  slot_name,
  active,
  confirmed_flush_lsn
FROM pg_replication_slots;
```

Expected:

```text
active = true
```

---

# Unified Reporting Views

Metabase reads from unified views only.

Example:

```sql
CREATE OR REPLACE VIEW public.sale_order AS
SELECT
  'TOP_2018' AS source_db,
  'TOP_2018:' || id::text AS global_id,
  id AS source_id,
  *
FROM raw_top_2018.sale_order

UNION ALL

SELECT
  'TOP_2018_N' AS source_db,
  'TOP_2018_N:' || id::text AS global_id,
  id AS source_id,
  *
FROM raw_top_2018_n.sale_order;
```

This safely merges both databases.

---

# Why Unified Views Were Necessary

Metabase cannot safely treat:

```text
raw_top_2018.sale_order
raw_top_2018_n.sale_order
```

as one logical table.

Unified views solve:

- duplicate integer IDs
- safe joins
- source tracking
- one-table analytics

---

# Metabase Configuration

Metabase connects only to:

```text
public.*
```

Never expose:

```text
raw_top_2018.*
raw_top_2018_n.*
```

These are replication storage layers only.

---

# Final Result

The completed system now provides:

```text
✔ Near real-time synchronization (~1–2 seconds)
✔ Two Odoo DBs unified into one analytics environment
✔ No public CDC exposure
✔ Tailscale-secured replication
✔ Safe UPSERT replication
✔ Reliable recovery using replication slots
✔ Centralized Metabase reporting
✔ Unified reporting layer
✔ Recoverable Debezium offsets
✔ Stable production-ready CDC architecture
```

---

# Recovery Procedures

## Restart Node ingest API

```bash
pm2 restart odoo-cdc-ingest
```

## View Node logs

```bash
pm2 logs odoo-cdc-ingest
```

## Restart Debezium

```cmd
docker restart debezium-top-2018
docker restart debezium-top-2018-n
```

## View Debezium logs

```cmd
docker logs --tail 100 debezium-top-2018
docker logs --tail 100 debezium-top-2018-n
```

## Check replication slots

```sql
SELECT
  slot_name,
  active,
  confirmed_flush_lsn
FROM pg_replication_slots;
```

## Check sync activity

```sql
SELECT *
FROM sync_admin.sync_events
ORDER BY id DESC
LIMIT 50;
```

```sql
SELECT *
FROM sync_admin.sync_status;
```

---

# Important Lessons Learned

1. Always create replication slots BEFORE final snapshot refresh.
2. Never expose raw schemas directly to Metabase.
3. Tailscale simplified the architecture dramatically.
4. Debezium decimal and timestamp payloads often require normalization.
5. offsets.dat can retain broken payload state.
6. CDC_SECRET mismatches are common failure points.
7. Raw schemas should remain replication-only.
8. Unified reporting views are the correct way to combine multiple Odoo DBs.

---

# Additional Notes

This runbook is intended to be executed sequentially from infrastructure preparation through CDC validation and Metabase integration. The exact implementation order, recovery procedures, replication-slot handling, Debezium configuration, Tailscale networking, Node.js CDC ingestion, unified reporting views, and troubleshooting guidance are all included above and should be treated as the operational source of truth for future environment replication.

# Appendix A — Copy-Paste Setup Assets

This appendix exists so a future operator does not need to reconstruct scripts or configuration from memory. Every required script, SQL block, and configuration template is written explicitly below.

---

# A1 — Final Synced Table List

Use this exact list everywhere: pg_dump, publications, Debezium include lists, Node whitelist, validation queries, and primary-key creation.

```text
sale_order
sale_order_line
account_invoice
account_invoice_line
account_move
account_move_line
res_partner
product_product
product_template
stock_picking
stock_move
stock_quant
res_users
res_company
stock_warehouse
sale_locations
product_uom
```

The following tables were checked and excluded because they do not exist in this Odoo setup:

```text
pos_order
pos_order_line
```

---

# A2 — Complete Python Dump Rewrite Script

Create this file on the VPS:

```bash
mkdir -p /root/odoo_analytics
cd /root/odoo_analytics
nano rewrite_dump_schema.py
```

Paste this full script:

```python
import sys
from pathlib import Path

if len(sys.argv) != 4:
    print("Usage: python3 rewrite_dump_schema.py input.sql output.sql target_schema")
    print("Example: python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018")
    sys.exit(1)

input_file = Path(sys.argv[1])
output_file = Path(sys.argv[2])
target_schema = sys.argv[3]

if not input_file.exists():
    print(f"ERROR: input file does not exist: {input_file}")
    sys.exit(1)

content = input_file.read_text(encoding="utf-8", errors="replace")

content = content.replace("public.", f"{target_schema}.")
content = content.replace("SET search_path = public, pg_catalog;", f"SET search_path = {target_schema}, pg_catalog;")
content = content.replace("SELECT pg_catalog.set_config('search_path', '', false);", f"SET search_path = {target_schema}, pg_catalog;")

prefix = f"CREATE SCHEMA IF NOT EXISTS {target_schema};
SET search_path = {target_schema}, pg_catalog;
"
output_file.write_text(prefix + content, encoding="utf-8")

print(f"Created {output_file} targeting schema {target_schema}")
```

Run it like this after uploading dump files to `/root/odoo_analytics`:

```bash
cd /root/odoo_analytics
python3 rewrite_dump_schema.py top_2018_predata.sql top_2018_predata_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_n_predata.sql top_2018_n_predata_raw.sql raw_top_2018_n
python3 rewrite_dump_schema.py top_2018_n_data.sql top_2018_n_data_raw.sql raw_top_2018_n
```

---

# A3 — Complete Windows Dump Commands

Use forward-slash paths on Windows to avoid escaping issues.

Create folders:

```cmd
mkdir C:/Users/PC/Desktop/BI
mkdir C:/Users/PC/Desktop/BI/dumps_clean
```

Schema dump for `top_2018`:

```cmd
pg_dump -h localhost -U openpg -d top_2018 -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=pre-data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_predata.sql"
```

Data dump for `top_2018`:

```cmd
pg_dump -h localhost -U openpg -d top_2018 -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_data.sql"
```

Schema dump for `top_2018_n`:

```cmd
pg_dump -h localhost -U openpg -d top_2018_n -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=pre-data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_n_predata.sql"
```

Data dump for `top_2018_n`:

```cmd
pg_dump -h localhost -U openpg -d top_2018_n -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_n_data.sql"
```

---

# A4 — Complete VPS Database Setup SQL

Create database:

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE odoo_analytics;
```

Connect:

```bash
psql -h localhost -U postgres -d odoo_analytics
```

Create schemas and sync tables:

```sql
CREATE SCHEMA IF NOT EXISTS raw_top_2018;
CREATE SCHEMA IF NOT EXISTS raw_top_2018_n;
CREATE SCHEMA IF NOT EXISTS sync_admin;

CREATE TABLE IF NOT EXISTS sync_admin.sync_events (
    id BIGSERIAL PRIMARY KEY,
    source_db TEXT NOT NULL,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL,
    source_id TEXT,
    received_at TIMESTAMPTZ DEFAULT now(),
    processed BOOLEAN DEFAULT false,
    error_message TEXT
);

CREATE TABLE IF NOT EXISTS sync_admin.sync_status (
    source_db TEXT PRIMARY KEY,
    last_event_at TIMESTAMPTZ,
    last_received_at TIMESTAMPTZ DEFAULT now(),
    last_error TEXT
);
```

Import rewritten dumps:

```bash
cd /root/odoo_analytics
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_predata_raw.sql
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_data_raw.sql
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_n_predata_raw.sql
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_n_data_raw.sql
```

Validate:

```sql
SELECT COUNT(*) FROM raw_top_2018.sale_order;
SELECT COUNT(*) FROM raw_top_2018_n.sale_order;

SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema IN ('raw_top_2018', 'raw_top_2018_n')
ORDER BY table_schema, table_name;
```

---

# A5 — Complete Raw Primary-Key Creation SQL

Run this after raw tables exist:

```sql
DO $$
DECLARE
  s text;
  t text;
BEGIN
  FOREACH s IN ARRAY ARRAY['raw_top_2018','raw_top_2018_n']
  LOOP
    FOREACH t IN ARRAY ARRAY['sale_order','sale_order_line','account_invoice','account_invoice_line','account_move','account_move_line','res_partner','product_product','product_template','stock_picking','stock_move','stock_quant','res_users','res_company','stock_warehouse','sale_locations','product_uom']
    LOOP
      IF NOT EXISTS (
        SELECT 1
        FROM pg_constraint c
        JOIN pg_class rel ON rel.oid = c.conrelid
        JOIN pg_namespace nsp ON nsp.oid = rel.relnamespace
        WHERE c.contype = 'p'
          AND nsp.nspname = s
          AND rel.relname = t
      ) THEN
        EXECUTE format('ALTER TABLE %I.%I ADD PRIMARY KEY (id)', s, t);
      END IF;
    END LOOP;
  END LOOP;
END $$;
```

Verify:

```sql
SELECT n.nspname AS schema_name, c.relname AS table_name, con.conname AS primary_key
FROM pg_constraint con
JOIN pg_class c ON c.oid = con.conrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE con.contype = 'p'
  AND n.nspname IN ('raw_top_2018', 'raw_top_2018_n')
ORDER BY n.nspname, c.relname;
```

Expected: 34 rows.

---

# A6 — Complete Source PostgreSQL Logical Replication SQL

Edit Windows PostgreSQL config:

```text
C:/Program Files/PostgreSQL/12/data/postgresql.conf
```

Set:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Restart PostgreSQL, then verify:

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

Create or update replication user:

```sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'metabase_sync') THEN
    CREATE ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  ELSE
    ALTER ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  END IF;
END
$$;

GRANT CONNECT ON DATABASE top_2018 TO metabase_sync;
GRANT CONNECT ON DATABASE top_2018_n TO metabase_sync;
```

Run inside each source DB:

```sql
GRANT USAGE ON SCHEMA public TO metabase_sync;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_sync;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO metabase_sync;
```

Create publication in `top_2018`:

```sql
DROP PUBLICATION IF EXISTS top_2018_metabase_pub;
CREATE PUBLICATION top_2018_metabase_pub FOR TABLE public.sale_order, public.sale_order_line, public.account_invoice, public.account_invoice_line, public.account_move, public.account_move_line, public.res_partner, public.product_product, public.product_template, public.stock_picking, public.stock_move, public.stock_quant, public.res_users, public.res_company, public.stock_warehouse, public.sale_locations, public.product_uom;
```

Create publication in `top_2018_n`:

```sql
DROP PUBLICATION IF EXISTS top_2018_n_metabase_pub;
CREATE PUBLICATION top_2018_n_metabase_pub FOR TABLE public.sale_order, public.sale_order_line, public.account_invoice, public.account_invoice_line, public.account_move, public.account_move_line, public.res_partner, public.product_product, public.product_template, public.stock_picking, public.stock_move, public.stock_quant, public.res_users, public.res_company, public.stock_warehouse, public.sale_locations, public.product_uom;
```

Create replication slots after publications and before final data refresh:

```sql
SELECT * FROM pg_create_logical_replication_slot('top_2018_metabase_slot', 'pgoutput');
SELECT * FROM pg_create_logical_replication_slot('top_2018_n_metabase_slot', 'pgoutput');
```

Verify:

```sql
SELECT slot_name, slot_type, active, plugin, database, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name IN ('top_2018_metabase_slot','top_2018_n_metabase_slot');
```

---

# A7 — Complete Node.js Ingest API

Create project:

```bash
sudo mkdir -p /opt/odoo-cdc-ingest
sudo chown -R $USER:$USER /opt/odoo-cdc-ingest
cd /opt/odoo-cdc-ingest
nano package.json
```

Paste `package.json`:

```json
{"name":"odoo-cdc-ingest","version":"1.0.0","main":"server.js","scripts":{"start":"node server.js"},"dependencies":{"express":"^4.18.3","pg":"^8.11.5"}}
```

Install:

```bash
npm install
```

Create server:

```bash
nano server.js
```

Paste `server.js`:

```js
const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json({ limit: '100mb' }));

const CDC_SECRET = process.env.CDC_SECRET;
if (!CDC_SECRET) {
  console.error('CDC_SECRET is required');
  process.exit(1);
}

const pool = new Pool({
  host: process.env.PGHOST || '127.0.0.1',
  port: Number(process.env.PGPORT || 5432),
  database: process.env.PGDATABASE || 'odoo_analytics',
  user: process.env.PGUSER || 'postgres',
  password: process.env.PGPASSWORD
});

const SOURCE_TO_SCHEMA = {
  top_2018: 'raw_top_2018',
  top_2018_n: 'raw_top_2018_n'
};

const ALLOWED_TABLES = new Set(['sale_order','sale_order_line','account_invoice','account_invoice_line','account_move','account_move_line','res_partner','product_product','product_template','stock_picking','stock_move','stock_quant','res_users','res_company','stock_warehouse','sale_locations','product_uom']);

const TIMESTAMP_COLUMNS = new Set(['create_date','write_date','date_order','confirmation_date','validity_date','date_invoice','date_due','date','date_maturity','scheduled_date','date_expected','date_done']);
const DATE_COLUMNS = new Set(['date_invoice','date_due','date_maturity']);

function quoteIdent(name) {
  return '"' + String(name).split('"').join('""') + '"';
}

function microsToTimestamp(value) {
  if (value === null || value === undefined) return value;
  if (typeof value !== 'number') return value;
  const millis = Math.floor(value / 1000);
  return new Date(millis).toISOString();
}

function daysToIsoDate(value) {
  if (value === null || value === undefined) return value;
  if (typeof value !== 'number') return value;
  const epoch = new Date(Date.UTC(1970, 0, 1));
  epoch.setUTCDate(epoch.getUTCDate() + value);
  return epoch.toISOString().slice(0, 10);
}

function normalizeValue(column, value) {
  if (value === null || value === undefined) return value;
  if (DATE_COLUMNS.has(column) && typeof value === 'number') return daysToIsoDate(value);
  if (TIMESTAMP_COLUMNS.has(column) && typeof value === 'number') return microsToTimestamp(value);
  return value;
}

function normalizeRow(row) {
  if (!row) return row;
  const normalized = {};
  for (const [key, value] of Object.entries(row)) {
    normalized[key] = normalizeValue(key, value);
  }
  return normalized;
}

function normalizeEvent(event) {
  return event.payload || event;
}

async function upsertRow(client, schema, table, inputRow) {
  const row = normalizeRow(inputRow);
  if (!row || row.id === undefined || row.id === null) throw new Error(`Missing id for upsert on ${schema}.${table}`);

  const columns = Object.keys(row);
  const quotedColumns = columns.map(quoteIdent);
  const placeholders = columns.map((_, i) => `$${i + 1}`);
  const updateColumns = columns.filter((col) => col !== 'id').map((col) => `${quoteIdent(col)} = EXCLUDED.${quoteIdent(col)}`);

  const sql = `INSERT INTO ${quoteIdent(schema)}.${quoteIdent(table)} (${quotedColumns.join(', ')}) VALUES (${placeholders.join(', ')}) ON CONFLICT ("id") DO UPDATE SET ${updateColumns.join(', ')}`;
  await client.query(sql, columns.map((col) => row[col]));
}

async function deleteRow(client, schema, table, inputRow) {
  const row = normalizeRow(inputRow);
  if (!row || row.id === undefined || row.id === null) throw new Error(`Missing id for delete on ${schema}.${table}`);
  await client.query(`DELETE FROM ${quoteIdent(schema)}.${quoteIdent(table)} WHERE "id" = $1`, [row.id]);
}

app.get('/health', (req, res) => {
  res.json({ ok: true });
});

app.post('/odoo-cdc/:source', async (req, res) => {
  if (req.header('x-cdc-secret') !== CDC_SECRET) return res.status(401).json({ error: 'Unauthorized' });

  const source = req.params.source;
  const schema = SOURCE_TO_SCHEMA[source];
  if (!schema) return res.status(400).json({ error: 'Unknown source' });

  const events = Array.isArray(req.body) ? req.body : [req.body];
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    for (const event of events) {
      const payload = normalizeEvent(event);
      const table = payload?.source?.table;
      const op = payload?.op;
      const after = payload?.after;
      const before = payload?.before;

      if (!table || !op) continue;
      if (!ALLOWED_TABLES.has(table)) throw new Error(`Table not allowed: ${table}`);

      if (op === 'c' || op === 'r' || op === 'u') await upsertRow(client, schema, table, after);
      else if (op === 'd') await deleteRow(client, schema, table, before);

      await client.query('INSERT INTO sync_admin.sync_events (source_db, table_name, operation, source_id, processed) VALUES ($1, $2, $3, $4, true)', [source, table, op, after?.id || before?.id || null]);
      await client.query('INSERT INTO sync_admin.sync_status (source_db, last_event_at, last_received_at, last_error) VALUES ($1, now(), now(), null) ON CONFLICT (source_db) DO UPDATE SET last_event_at = now(), last_received_at = now(), last_error = null', [source]);
    }

    await client.query('COMMIT');
    res.json({ ok: true, count: events.length });
  } catch (error) {
    await client.query('ROLLBACK');
    await client.query('INSERT INTO sync_admin.sync_status (source_db, last_received_at, last_error) VALUES ($1, now(), $2) ON CONFLICT (source_db) DO UPDATE SET last_received_at = now(), last_error = $2', [source, error.message]);
    res.status(500).json({ error: error.message });
  } finally {
    client.release();
  }
});

app.listen(8088, '0.0.0.0', () => {
  console.log('Odoo CDC ingest API running on 0.0.0.0:8088');
});
```

Start with PM2:

```bash
openssl rand -hex 32
cd /opt/odoo-cdc-ingest
CDC_SECRET='REPLACE_WITH_GENERATED_SECRET' PGHOST='127.0.0.1' PGPORT='5432' PGDATABASE='odoo_analytics' PGUSER='postgres' PGPASSWORD='REPLACE_WITH_POSTGRES_PASSWORD' pm2 start server.js --name odoo-cdc-ingest
pm2 save
pm2 startup
```

Test:

```bash
curl http://127.0.0.1:8088/health
```

From Windows over Tailscale:

```cmd
curl http://VPS_TAILSCALE_IP:8088/health
```

---

# A8 — Complete Debezium Config Files

Create folders on Windows:

```cmd
mkdir C:/odoo-cdc
mkdir C:/odoo-cdc/top_2018
mkdir C:/odoo-cdc/top_2018_n
mkdir C:/odoo-cdc/data_top_2018
mkdir C:/odoo-cdc/data_top_2018_n
```

Create `C:/odoo-cdc/top_2018/application.properties`:

```properties
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=host.docker.internal
debezium.source.database.port=5432
debezium.source.database.user=metabase_sync
debezium.source.database.password=REPLACE_WITH_REPLICATION_PASSWORD
debezium.source.database.dbname=top_2018
debezium.source.topic.prefix=top_2018
debezium.source.plugin.name=pgoutput
debezium.source.slot.name=top_2018_metabase_slot
debezium.source.publication.name=top_2018_metabase_pub
debezium.source.slot.drop.on.stop=false
debezium.source.schema.include.list=public
debezium.source.table.include.list=public.sale_order,public.sale_order_line,public.account_invoice,public.account_invoice_line,public.account_move,public.account_move_line,public.res_partner,public.product_product,public.product_template,public.stock_picking,public.stock_move,public.stock_quant,public.res_users,public.res_company,public.stock_warehouse,public.sale_locations,public.product_uom
debezium.source.snapshot.mode=never
debezium.source.decimal.handling.mode=double
timestamp.mode=iso_offset
errors.max.retries=-1
debezium.format.key=json
debezium.format.value=json
debezium.sink.type=http
debezium.sink.http.url=http://REPLACE_WITH_VPS_TAILSCALE_IP:8088/odoo-cdc/top_2018
debezium.sink.http.headers=x-cdc-secret:REPLACE_WITH_CDC_SECRET
```

Create `C:/odoo-cdc/top_2018_n/application.properties`:

```properties
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=host.docker.internal
debezium.source.database.port=5432
debezium.source.database.user=metabase_sync
debezium.source.database.password=REPLACE_WITH_REPLICATION_PASSWORD
debezium.source.database.dbname=top_2018_n
debezium.source.topic.prefix=top_2018_n
debezium.source.plugin.name=pgoutput
debezium.source.slot.name=top_2018_n_metabase_slot
debezium.source.publication.name=top_2018_n_metabase_pub
debezium.source.slot.drop.on.stop=false
debezium.source.schema.include.list=public
debezium.source.table.include.list=public.sale_order,public.sale_order_line,public.account_invoice,public.account_invoice_line,public.account_move,public.account_move_line,public.res_partner,public.product_product,public.product_template,public.stock_picking,public.stock_move,public.stock_quant,public.res_users,public.res_company,public.stock_warehouse,public.sale_locations,public.product_uom
debezium.source.snapshot.mode=never
debezium.source.decimal.handling.mode=double
timestamp.mode=iso_offset
errors.max.retries=-1
debezium.format.key=json
debezium.format.value=json
debezium.sink.type=http
debezium.sink.http.url=http://REPLACE_WITH_VPS_TAILSCALE_IP:8088/odoo-cdc/top_2018_n
debezium.sink.http.headers=x-cdc-secret:REPLACE_WITH_CDC_SECRET
```

Start containers:

```cmd
docker run -d --name debezium-top-2018 --restart unless-stopped -v C:/odoo-cdc/top_2018:/debezium/conf -v C:/odoo-cdc/data_top_2018:/debezium/data debezium/server:2.2
```

```cmd
docker run -d --name debezium-top-2018-n --restart unless-stopped -v C:/odoo-cdc/top_2018_n:/debezium/conf -v C:/odoo-cdc/data_top_2018_n:/debezium/data debezium/server:2.2
```

Check logs:

```cmd
docker logs -f debezium-top-2018
docker logs -f debezium-top-2018-n
```

If malformed old events are stuck after fixing config, stop container, delete its offset file from the matching `data_*` folder, and restart.

---

# A9 — Complete Unified Views Generator

Create this file on the VPS:

```bash
cd /root/odoo_analytics
nano create_unified_views.sql
```

Paste:

```sql
DO $$
DECLARE
  t text;
  cols text;
BEGIN
  FOREACH t IN ARRAY ARRAY['sale_order','sale_order_line','account_invoice','account_invoice_line','account_move','account_move_line','res_partner','product_product','product_template','stock_picking','stock_move','stock_quant','res_users','res_company','stock_warehouse','sale_locations','product_uom']
  LOOP
    SELECT string_agg(format('r.%I', column_name), ', ' ORDER BY ordinal_position)
    INTO cols
    FROM information_schema.columns
    WHERE table_schema = 'raw_top_2018'
      AND table_name = t;

    EXECUTE format(
      'CREATE OR REPLACE VIEW public.%I AS SELECT %L::text AS source_db, %L || r.id::text AS global_id, r.id AS source_id, %s FROM raw_top_2018.%I r UNION ALL SELECT %L::text AS source_db, %L || r.id::text AS global_id, r.id AS source_id, %s FROM raw_top_2018_n.%I r',
      t,
      'TOP_2018',
      'TOP_2018:',
      cols,
      t,
      'TOP_2018_N',
      'TOP_2018_N:',
      cols,
      t
    );
  END LOOP;
END $$;
```

Run:

```bash
psql -h localhost -U postgres -d odoo_analytics -f create_unified_views.sql
```

Validate:

```sql
SELECT table_schema, table_name
FROM information_schema.views
WHERE table_schema = 'public'
ORDER BY table_name;
```

---

# A10 — Complete Validation And Troubleshooting Commands

VPS API health:

```bash
curl http://127.0.0.1:8088/health
pm2 logs odoo-cdc-ingest --lines 100
```

Windows to VPS over Tailscale:

```cmd
curl http://VPS_TAILSCALE_IP:8088/health
```

Debezium logs:

```cmd
docker logs --tail 100 debezium-top-2018
docker logs --tail 100 debezium-top-2018-n
```

Replication slots:

```sql
SELECT slot_name, active, plugin, database, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name IN ('top_2018_metabase_slot','top_2018_n_metabase_slot');
```

Sync events:

```sql
SELECT * FROM sync_admin.sync_events ORDER BY id DESC LIMIT 50;
SELECT * FROM sync_admin.sync_status;
```

Test source update:

```sql
UPDATE sale_order
SET write_date = now()
WHERE id = (SELECT id FROM sale_order ORDER BY id DESC LIMIT 1);
```

# Appendix L — Exhaustive SQL Query Checklist

This section lists the exact SQL queries that must be run or used for verification during replication setup.

## L1 — VPS database and schemas

```sql
CREATE DATABASE odoo_analytics;
```

Connect to `odoo_analytics`, then run:

```sql
CREATE SCHEMA IF NOT EXISTS raw_top_2018;
CREATE SCHEMA IF NOT EXISTS raw_top_2018_n;
CREATE SCHEMA IF NOT EXISTS sync_admin;
```

## L2 — Sync admin tables

```sql
CREATE TABLE IF NOT EXISTS sync_admin.sync_events (
    id BIGSERIAL PRIMARY KEY,
    source_db TEXT NOT NULL,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL,
    source_id TEXT,
    received_at TIMESTAMPTZ DEFAULT now(),
    processed BOOLEAN DEFAULT false,
    error_message TEXT
);

CREATE TABLE IF NOT EXISTS sync_admin.sync_status (
    source_db TEXT PRIMARY KEY,
    last_event_at TIMESTAMPTZ,
    last_received_at TIMESTAMPTZ DEFAULT now(),
    last_error TEXT
);
```

## L3 — Validate raw tables exist

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema IN ('raw_top_2018', 'raw_top_2018_n')
ORDER BY table_schema, table_name;
```

## L4 — Validate source and target sale order counts

On source `top_2018`:

```sql
SELECT COUNT(*) FROM sale_order;
```

On source `top_2018_n`:

```sql
SELECT COUNT(*) FROM sale_order;
```

On VPS:

```sql
SELECT COUNT(*) FROM raw_top_2018.sale_order;
SELECT COUNT(*) FROM raw_top_2018_n.sale_order;
```

## L5 — Check missing POS tables

Run on source:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_name IN ('pos_order', 'pos_order_line');
```

Expected in this implementation:

```text
0 rows
```

## L6 — PostgreSQL logical replication settings

Run on source PostgreSQL:

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

Expected:

```text
wal_level = logical
max_replication_slots >= 2
max_wal_senders >= 2
```

## L7 — Create or update replication user

Run on source PostgreSQL as superuser:

```sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'metabase_sync') THEN
    CREATE ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  ELSE
    ALTER ROLE metabase_sync WITH LOGIN REPLICATION PASSWORD 'CHANGE_STRONG_PASSWORD';
  END IF;
END
$$;

GRANT CONNECT ON DATABASE top_2018 TO metabase_sync;
GRANT CONNECT ON DATABASE top_2018_n TO metabase_sync;
```

Run inside both source databases:

```sql
GRANT USAGE ON SCHEMA public TO metabase_sync;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_sync;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO metabase_sync;
```

## L8 — Create publications

Inside `top_2018`:

```sql
DROP PUBLICATION IF EXISTS top_2018_metabase_pub;

CREATE PUBLICATION top_2018_metabase_pub FOR TABLE
public.sale_order,
public.sale_order_line,
public.account_invoice,
public.account_invoice_line,
public.account_move,
public.account_move_line,
public.res_partner,
public.product_product,
public.product_template,
public.stock_picking,
public.stock_move,
public.stock_quant,
public.res_users,
public.res_company,
public.stock_warehouse,
public.sale_locations,
public.product_uom;
```

Inside `top_2018_n`:

```sql
DROP PUBLICATION IF EXISTS top_2018_n_metabase_pub;

CREATE PUBLICATION top_2018_n_metabase_pub FOR TABLE
public.sale_order,
public.sale_order_line,
public.account_invoice,
public.account_invoice_line,
public.account_move,
public.account_move_line,
public.res_partner,
public.product_product,
public.product_template,
public.stock_picking,
public.stock_move,
public.stock_quant,
public.res_users,
public.res_company,
public.stock_warehouse,
public.sale_locations,
public.product_uom;
```

Verify publications:

```sql
SELECT pubname FROM pg_publication;

SELECT pubname, schemaname, tablename
FROM pg_publication_tables
WHERE pubname IN ('top_2018_metabase_pub', 'top_2018_n_metabase_pub')
ORDER BY pubname, tablename;
```

## L9 — Create replication slots

Inside `top_2018`:

```sql
SELECT * FROM pg_create_logical_replication_slot('top_2018_metabase_slot', 'pgoutput');
```

Inside `top_2018_n`:

```sql
SELECT * FROM pg_create_logical_replication_slot('top_2018_n_metabase_slot', 'pgoutput');
```

Verify slots:

```sql
SELECT
  slot_name,
  slot_type,
  active,
  plugin,
  database,
  restart_lsn,
  confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name IN ('top_2018_metabase_slot','top_2018_n_metabase_slot');
```

## L10 — Truncate raw tables before refreshed data-only import

Run on VPS only after replication slots have been created and fresh data dumps are ready:

```sql
TRUNCATE
raw_top_2018.sale_order,
raw_top_2018.sale_order_line,
raw_top_2018.account_invoice,
raw_top_2018.account_invoice_line,
raw_top_2018.account_move,
raw_top_2018.account_move_line,
raw_top_2018.res_partner,
raw_top_2018.product_product,
raw_top_2018.product_template,
raw_top_2018.stock_picking,
raw_top_2018.stock_move,
raw_top_2018.stock_quant,
raw_top_2018.res_users,
raw_top_2018.res_company,
raw_top_2018.stock_warehouse,
raw_top_2018.sale_locations,
raw_top_2018.product_uom;

TRUNCATE
raw_top_2018_n.sale_order,
raw_top_2018_n.sale_order_line,
raw_top_2018_n.account_invoice,
raw_top_2018_n.account_invoice_line,
raw_top_2018_n.account_move,
raw_top_2018_n.account_move_line,
raw_top_2018_n.res_partner,
raw_top_2018_n.product_product,
raw_top_2018_n.product_template,
raw_top_2018_n.stock_picking,
raw_top_2018_n.stock_move,
raw_top_2018_n.stock_quant,
raw_top_2018_n.res_users,
raw_top_2018_n.res_company,
raw_top_2018_n.stock_warehouse,
raw_top_2018_n.sale_locations,
raw_top_2018_n.product_uom;
```

## L11 — Add primary keys to raw tables

```sql
DO $$
DECLARE
  s text;
  t text;
BEGIN
  FOREACH s IN ARRAY ARRAY['raw_top_2018','raw_top_2018_n']
  LOOP
    FOREACH t IN ARRAY ARRAY['sale_order','sale_order_line','account_invoice','account_invoice_line','account_move','account_move_line','res_partner','product_product','product_template','stock_picking','stock_move','stock_quant','res_users','res_company','stock_warehouse','sale_locations','product_uom']
    LOOP
      IF NOT EXISTS (
        SELECT 1
        FROM pg_constraint c
        JOIN pg_class rel ON rel.oid = c.conrelid
        JOIN pg_namespace nsp ON nsp.oid = rel.relnamespace
        WHERE c.contype = 'p'
          AND nsp.nspname = s
          AND rel.relname = t
      ) THEN
        EXECUTE format('ALTER TABLE %I.%I ADD PRIMARY KEY (id)', s, t);
      END IF;
    END LOOP;
  END LOOP;
END $$;
```

## L12 — Validate primary keys

```sql
SELECT
  n.nspname AS schema_name,
  c.relname AS table_name,
  con.conname AS primary_key
FROM pg_constraint con
JOIN pg_class c ON c.oid = con.conrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE con.contype = 'p'
  AND n.nspname IN ('raw_top_2018', 'raw_top_2018_n')
ORDER BY n.nspname, c.relname;
```

Expected:

```text
34 rows
```

## L13 — Validate CDC activity

```sql
SELECT *
FROM sync_admin.sync_status;

SELECT *
FROM sync_admin.sync_events
ORDER BY id DESC
LIMIT 50;
```

## L14 — Source test update

Run on source DB:

```sql
UPDATE sale_order
SET write_date = now()
WHERE id = (SELECT id FROM sale_order ORDER BY id DESC LIMIT 1);
```

Then check VPS:

```sql
SELECT *
FROM sync_admin.sync_events
ORDER BY id DESC
LIMIT 10;
```

## L15 — Unified views generator

Create `/root/odoo_analytics/create_unified_views.sql`:

```sql
DO $$
DECLARE
  t text;
  cols text;
BEGIN
  FOREACH t IN ARRAY ARRAY['sale_order','sale_order_line','account_invoice','account_invoice_line','account_move','account_move_line','res_partner','product_product','product_template','stock_picking','stock_move','stock_quant','res_users','res_company','stock_warehouse','sale_locations','product_uom']
  LOOP
    SELECT string_agg(format('r.%I', column_name), ', ' ORDER BY ordinal_position)
    INTO cols
    FROM information_schema.columns
    WHERE table_schema = 'raw_top_2018'
      AND table_name = t;

    EXECUTE format(
      'CREATE OR REPLACE VIEW public.%I AS SELECT %L::text AS source_db, %L || r.id::text AS global_id, r.id AS source_id, %s FROM raw_top_2018.%I r UNION ALL SELECT %L::text AS source_db, %L || r.id::text AS global_id, r.id AS source_id, %s FROM raw_top_2018_n.%I r',
      t,
      'TOP_2018',
      'TOP_2018:',
      cols,
      t,
      'TOP_2018_N',
      'TOP_2018_N:',
      cols,
      t
    );
  END LOOP;
END $$;
```

Run:

```bash
psql -h localhost -U postgres -d odoo_analytics -f /root/odoo_analytics/create_unified_views.sql
```

Validate:

```sql
SELECT table_schema, table_name
FROM information_schema.views
WHERE table_schema = 'public'
ORDER BY table_name;
```

# End Of Runbook

