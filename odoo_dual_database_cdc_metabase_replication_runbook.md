# Odoo Dual Database CDC + Metabase Replication Runbook

## PostgreSQL 9.5.8 + wal2json + Debezium 1.x Edition

This document is the exhaustive, copy-paste-oriented setup guide for replicating two Odoo PostgreSQL databases into a VPS analytics database for Metabase reporting.

This version is specifically for:

```text
PostgreSQL source version: 9.5.8
Logical decoding plugin: wal2json
Debezium version family: 1.x
Transport: Tailscale private network
Target: VPS PostgreSQL analytics database
Reporting: Metabase using unified public views
```

Important change from the PostgreSQL 10+ design:

```text
PostgreSQL 9.5.8 does NOT support CREATE PUBLICATION.
PostgreSQL 9.5.8 does NOT support pgoutput.
PostgreSQL 9.5.8 can work with Debezium only through logical decoding plugins such as wal2json.
```

Therefore this runbook uses:

```text
wal2json replication slots
NO publications
NO pgoutput
Debezium 1.x, not Debezium 2.x
```

---

# Final Architecture

```text
[Windows Odoo Server]
    ├── PostgreSQL 9.5.8
    ├── Odoo database: top_2018
    ├── Odoo database: top_2018_n
    ├── wal2json logical decoding plugin
    ├── Debezium Server 1.x containers
    └── Tailscale node
              ↓
        Private Tailscale network
              ↓
[VPS]
    ├── Tailscale node
    ├── Node.js CDC ingest API on port 8088
    ├── PostgreSQL analytics database: odoo_analytics
    ├── raw schemas: raw_top_2018, raw_top_2018_n
    ├── unified public reporting views
    └── Metabase
```

---

# Final Synced Tables

Use this exact table list everywhere.

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

Excluded because not present in this Odoo setup:

```text
pos_order
pos_order_line
```

---

# Phase 0 — Critical Compatibility Rules

## 0.1 PostgreSQL 9.5.8 rule

Do not use:

```sql
CREATE PUBLICATION ...
```

Do not use:

```properties
debezium.source.plugin.name=pgoutput
```

Use:

```properties
debezium.source.plugin.name=wal2json
```

Create slots with:

```sql
SELECT * FROM pg_create_logical_replication_slot('slot_name', 'wal2json');
```

## 0.2 Debezium version rule

Use Debezium 1.x.

Recommended Docker image:

```text
debezium/server:1.9
```

Do not use Debezium 2.x for this PostgreSQL 9.5 + wal2json setup, because the later Debezium family removed wal2json support.

## 0.3 Raw schema rule

The target raw schemas are replication storage only:

```text
raw_top_2018
raw_top_2018_n
```

They should not be directly used by Metabase.

## 0.4 Reporting rule

Metabase reads unified views only:

```text
public.sale_order
public.sale_order_line
public.account_invoice
...
```

## 0.5 Snapshot-gap rule

Because Odoo may continue writing while setup is happening:

```text
Create wal2json slots first
Then refresh the data-only snapshot
Then start Debezium
```

This prevents missing changes between snapshot and CDC startup.

---

# Phase 1 — Prepare Source Windows Server

## 1.1 Confirm PostgreSQL version

Run in source PostgreSQL:

```sql
SELECT version();
```

Expected pattern:

```text
PostgreSQL 9.5.8 ... 64-bit
```

## 1.2 Confirm Odoo databases exist

Run as PostgreSQL superuser:

```sql
SELECT datname
FROM pg_database
WHERE datname IN ('top_2018', 'top_2018_n')
ORDER BY datname;
```

Expected:

```text
top_2018
top_2018_n
```

## 1.3 Confirm final table list exists

Run inside `top_2018`:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name IN (
    'sale_order',
    'sale_order_line',
    'account_invoice',
    'account_invoice_line',
    'account_move',
    'account_move_line',
    'res_partner',
    'product_product',
    'product_template',
    'stock_picking',
    'stock_move',
    'stock_quant',
    'res_users',
    'res_company',
    'stock_warehouse',
    'sale_locations',
    'product_uom'
  )
ORDER BY table_name;
```

Repeat inside `top_2018_n`.

Expected: 17 rows in each database.

## 1.4 Confirm POS tables are absent

Run inside both source databases:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_name IN ('pos_order', 'pos_order_line');
```

Expected for this implementation:

```text
0 rows
```

---

# Phase 2 — Install And Verify wal2json

The user already compiled `wal2json` for PostgreSQL 9.5.8 and confirmed it works. This phase documents how to verify that on any new instance.

## 2.1 Verify wal2json slot creation

Run inside `top_2018`:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'test_wal2json_slot',
  'wal2json'
);
```

Expected: PostgreSQL returns a slot name and LSN.

Then immediately drop the test slot:

```sql
SELECT pg_drop_replication_slot('test_wal2json_slot');
```

## 2.2 If test slot already exists

Check:

```sql
SELECT slot_name, plugin, active
FROM pg_replication_slots
WHERE slot_name = 'test_wal2json_slot';
```

If inactive, drop:

```sql
SELECT pg_drop_replication_slot('test_wal2json_slot');
```

## 2.3 Verify wal2json is usable in second DB

Run inside `top_2018_n`:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'test_wal2json_slot_n',
  'wal2json'
);
```

Drop:

```sql
SELECT pg_drop_replication_slot('test_wal2json_slot_n');
```

---

# Phase 3 — Configure Source PostgreSQL For Logical Decoding

## 3.1 Edit postgresql.conf

On Windows, edit:

```text
C:/Program Files/PostgreSQL/9.5/data/postgresql.conf
```

The path may differ depending on the installation directory.

Set:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Recommended additional settings:

```conf
max_worker_processes = 8
```

## 3.2 Restart PostgreSQL service

Use Windows Services:

```text
Services → postgresql-x64-9.5 → Restart
```

Or command prompt as Administrator:

```cmd
net stop postgresql-x64-9.5
net start postgresql-x64-9.5
```

Service name may differ.

## 3.3 Verify settings

Run:

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

---

# Phase 4 — Create Source Replication User

Run as PostgreSQL superuser.

## 4.1 Create or update metabase_sync user

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

## 4.2 Grant database connection access

```sql
GRANT CONNECT ON DATABASE top_2018 TO metabase_sync;
GRANT CONNECT ON DATABASE top_2018_n TO metabase_sync;
```

## 4.3 Grant table read access in top_2018

Connect to `top_2018`, then run:

```sql
GRANT USAGE ON SCHEMA public TO metabase_sync;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_sync;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO metabase_sync;
```

## 4.4 Grant table read access in top_2018_n

Connect to `top_2018_n`, then run:

```sql
GRANT USAGE ON SCHEMA public TO metabase_sync;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_sync;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO metabase_sync;
```

## 4.5 Verify role

```sql
SELECT rolname, rolreplication, rolcanlogin
FROM pg_roles
WHERE rolname = 'metabase_sync';
```

Expected:

```text
rolreplication = true
rolcanlogin = true
```

---

# Phase 5 — Prepare VPS Analytics Database

## 5.1 Install PostgreSQL if needed

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

## 5.2 Create analytics database

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE odoo_analytics;
```

If it already exists:

```sql
SELECT datname FROM pg_database WHERE datname = 'odoo_analytics';
```

## 5.3 Create schemas

Connect:

```bash
psql -h localhost -U postgres -d odoo_analytics
```

Run:

```sql
CREATE SCHEMA IF NOT EXISTS raw_top_2018;
CREATE SCHEMA IF NOT EXISTS raw_top_2018_n;
CREATE SCHEMA IF NOT EXISTS sync_admin;
```

## 5.4 Create sync admin tables

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

---

# Phase 6 — Initial Snapshot Export From Source

Create dump folder on Windows:

```cmd
mkdir C:/Users/PC/Desktop/BI
mkdir C:/Users/PC/Desktop/BI/dumps_clean
```

## 6.1 Schema dump for top_2018

```cmd
pg_dump -h localhost -U openpg -d top_2018 -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=pre-data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_predata.sql"
```

## 6.2 Data dump for top_2018

```cmd
pg_dump -h localhost -U openpg -d top_2018 -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_data.sql"
```

## 6.3 Schema dump for top_2018_n

```cmd
pg_dump -h localhost -U openpg -d top_2018_n -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=pre-data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_n_predata.sql"
```

## 6.4 Data dump for top_2018_n

```cmd
pg_dump -h localhost -U openpg -d top_2018_n -t public.sale_order -t public.sale_order_line -t public.account_invoice -t public.account_invoice_line -t public.account_move -t public.account_move_line -t public.res_partner -t public.product_product -t public.product_template -t public.stock_picking -t public.stock_move -t public.stock_quant -t public.res_users -t public.res_company -t public.stock_warehouse -t public.sale_locations -t public.product_uom --section=data --no-owner --no-privileges -f "C:/Users/PC/Desktop/BI/dumps_clean/top_2018_n_data.sql"
```

---

# Phase 7 — Rewrite Dump Schemas For VPS Raw Schemas

Upload dump files to VPS:

```text
/root/odoo_analytics/top_2018_predata.sql
/root/odoo_analytics/top_2018_data.sql
/root/odoo_analytics/top_2018_n_predata.sql
/root/odoo_analytics/top_2018_n_data.sql
```

## 7.1 Create rewrite script

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

## 7.2 Rewrite files

```bash
cd /root/odoo_analytics
python3 rewrite_dump_schema.py top_2018_predata.sql top_2018_predata_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_n_predata.sql top_2018_n_predata_raw.sql raw_top_2018_n
python3 rewrite_dump_schema.py top_2018_n_data.sql top_2018_n_data_raw.sql raw_top_2018_n
```

---

# Phase 8 — Import Initial Snapshot Into VPS

Run:

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

# Phase 9 — Create wal2json Replication Slots

No publications are used in PostgreSQL 9.5.

Create slot inside `top_2018`:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'top_2018_metabase_slot',
  'wal2json'
);
```

Create slot inside `top_2018_n`:

```sql
SELECT *
FROM pg_create_logical_replication_slot(
  'top_2018_n_metabase_slot',
  'wal2json'
);
```

Verify:

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

Expected before Debezium starts:

```text
plugin = wal2json
active = false
```

---

# Phase 10 — Refresh Data Snapshot After Slots

This is the anti-gap step.

## 10.1 Create fresh data-only dumps

Repeat Phase 6 data dump commands only:

```text
top_2018_data.sql
top_2018_n_data.sql
```

Upload them to VPS.

## 10.2 Rewrite refreshed data dumps

```bash
cd /root/odoo_analytics
python3 rewrite_dump_schema.py top_2018_data.sql top_2018_data_refresh_raw.sql raw_top_2018
python3 rewrite_dump_schema.py top_2018_n_data.sql top_2018_n_data_refresh_raw.sql raw_top_2018_n
```

## 10.3 Truncate raw tables

Run on VPS:

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

## 10.4 Import refreshed data

```bash
cd /root/odoo_analytics
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_data_refresh_raw.sql
psql -v ON_ERROR_STOP=1 -h localhost -U postgres -d odoo_analytics -f top_2018_n_data_refresh_raw.sql
```

---

# Phase 11 — Add Primary Keys To Raw Tables

Run on VPS:

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

Expected:

```text
34 rows
```

---

# Phase 12 — Configure Tailscale Transport

Install Tailscale on both:

```text
Windows Odoo server
VPS
```

Get VPS Tailscale IP:

```bash
tailscale ip -4
```

Test from Windows:

```cmd
ping VPS_TAILSCALE_IP
```

The CDC API will be accessed at:

```text
http://VPS_TAILSCALE_IP:8088
```

No Apache or public HTTPS is required for CDC.

---

# Phase 13 — Deploy Node.js CDC Ingest API

## 13.1 Install Node.js and PM2 on VPS

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

## 13.2 Create project

```bash
sudo mkdir -p /opt/odoo-cdc-ingest
sudo chown -R $USER:$USER /opt/odoo-cdc-ingest
cd /opt/odoo-cdc-ingest
nano package.json
```

Paste:

```json
{"name":"odoo-cdc-ingest","version":"1.0.0","main":"server.js","scripts":{"start":"node server.js"},"dependencies":{"express":"^4.18.3","pg":"^8.11.5"}}
```

Install:

```bash
npm install
```

## 13.3 Create server.js

```bash
nano server.js
```

Paste:

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
  for (const [key, value] of Object.entries(row)) normalized[key] = normalizeValue(key, value);
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

app.get('/health', (req, res) => res.json({ ok: true }));

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

app.listen(8088, '0.0.0.0', () => console.log('Odoo CDC ingest API running on 0.0.0.0:8088'));
```

## 13.4 Start with PM2

Generate secret:

```bash
openssl rand -hex 32
```

Start:

```bash
cd /opt/odoo-cdc-ingest
CDC_SECRET='REPLACE_WITH_GENERATED_SECRET' PGHOST='127.0.0.1' PGPORT='5432' PGDATABASE='odoo_analytics' PGUSER='postgres' PGPASSWORD='REPLACE_WITH_POSTGRES_PASSWORD' pm2 start server.js --name odoo-cdc-ingest
pm2 save
pm2 startup
```

Test:

```bash
curl http://127.0.0.1:8088/health
```

From Windows:

```cmd
curl http://VPS_TAILSCALE_IP:8088/health
```

---

# Phase 14 — Configure Debezium 1.x With wal2json

## 14.1 Create folders on Windows

```cmd
mkdir C:/odoo-cdc
mkdir C:/odoo-cdc/top_2018
mkdir C:/odoo-cdc/top_2018_n
mkdir C:/odoo-cdc/data_top_2018
mkdir C:/odoo-cdc/data_top_2018_n
```

## 14.2 top_2018 Debezium config

Create:

```text
C:/odoo-cdc/top_2018/application.properties
```

Paste:

```properties
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=host.docker.internal
debezium.source.database.port=5432
debezium.source.database.user=metabase_sync
debezium.source.database.password=REPLACE_WITH_REPLICATION_PASSWORD
debezium.source.database.dbname=top_2018
debezium.source.database.server.name=top_2018
debezium.source.plugin.name=wal2json
debezium.source.slot.name=top_2018_metabase_slot
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

## 14.3 top_2018_n Debezium config

Create:

```text
C:/odoo-cdc/top_2018_n/application.properties
```

Paste:

```properties
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=host.docker.internal
debezium.source.database.port=5432
debezium.source.database.user=metabase_sync
debezium.source.database.password=REPLACE_WITH_REPLICATION_PASSWORD
debezium.source.database.dbname=top_2018_n
debezium.source.database.server.name=top_2018_n
debezium.source.plugin.name=wal2json
debezium.source.slot.name=top_2018_n_metabase_slot
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

## 14.4 Start Debezium 1.9 containers

```cmd
docker run -d --name debezium-top-2018 --restart unless-stopped -v C:/odoo-cdc/top_2018:/debezium/conf -v C:/odoo-cdc/data_top_2018:/debezium/data debezium/server:1.9
```

```cmd
docker run -d --name debezium-top-2018-n --restart unless-stopped -v C:/odoo-cdc/top_2018_n:/debezium/conf -v C:/odoo-cdc/data_top_2018_n:/debezium/data debezium/server:1.9
```

Check logs:

```cmd
docker logs -f debezium-top-2018
docker logs -f debezium-top-2018-n
```

If old malformed events are stuck after a config change:

```cmd
docker stop debezium-top-2018
docker stop debezium-top-2018-n
```

Delete:

```text
C:/odoo-cdc/data_top_2018/offsets.dat
C:/odoo-cdc/data_top_2018_n/offsets.dat
```

Then restart containers.

---

# Phase 15 — Validate CDC End-To-End

## 15.1 Slot active check

Run on source PostgreSQL:

```sql
SELECT slot_name, active, plugin, database, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name IN ('top_2018_metabase_slot','top_2018_n_metabase_slot');
```

Expected while Debezium is running:

```text
active = true
plugin = wal2json
```

## 15.2 Test source update

Run inside `top_2018`:

```sql
UPDATE sale_order
SET write_date = now()
WHERE id = (SELECT id FROM sale_order ORDER BY id DESC LIMIT 1);
```

Run inside `top_2018_n`:

```sql
UPDATE sale_order
SET write_date = now()
WHERE id = (SELECT id FROM sale_order ORDER BY id DESC LIMIT 1);
```

## 15.3 Check VPS sync logs

```sql
SELECT * FROM sync_admin.sync_status;

SELECT *
FROM sync_admin.sync_events
ORDER BY id DESC
LIMIT 50;
```

## 15.4 Check target row changed

Use the changed source ID and run:

```sql
SELECT id, write_date
FROM raw_top_2018.sale_order
ORDER BY write_date DESC
LIMIT 10;

SELECT id, write_date
FROM raw_top_2018_n.sale_order
ORDER BY write_date DESC
LIMIT 10;
```

---

# Phase 16 — Create Unified Public Views

Create generator:

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
psql -h localhost -U postgres -d odoo_analytics -f /root/odoo_analytics/create_unified_views.sql
```

Validate:

```sql
SELECT table_schema, table_name
FROM information_schema.views
WHERE table_schema = 'public'
ORDER BY table_name;
```

---

# Phase 17 — Configure Metabase

## 17.1 Create readonly user

Run in `odoo_analytics`:

```sql
CREATE USER metabase_readonly WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
GRANT CONNECT ON DATABASE odoo_analytics TO metabase_readonly;
GRANT USAGE ON SCHEMA public TO metabase_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO metabase_readonly;
```

## 17.2 Metabase connection

Use:

```text
Database type: PostgreSQL
Host: 127.0.0.1 or VPS private database host
Port: 5432
Database: odoo_analytics
User: metabase_readonly
Schema: public
```

Only expose:

```text
public.*
```

Do not expose:

```text
raw_top_2018.*
raw_top_2018_n.*
```

---

# Phase 18 — Operations And Recovery

## 18.1 Restart Node ingest API

```bash
pm2 restart odoo-cdc-ingest
pm2 logs odoo-cdc-ingest --lines 100
```

## 18.2 Restart Debezium

```cmd
docker restart debezium-top-2018
docker restart debezium-top-2018-n
```

## 18.3 Stop Debezium

```cmd
docker stop debezium-top-2018
docker stop debezium-top-2018-n
```

## 18.4 Recreate Debezium container after config changes

```cmd
docker stop debezium-top-2018
docker rm debezium-top-2018
docker run -d --name debezium-top-2018 --restart unless-stopped -v C:/odoo-cdc/top_2018:/debezium/conf -v C:/odoo-cdc/data_top_2018:/debezium/data debezium/server:1.9
```

Repeat for `debezium-top-2018-n`.

## 18.5 Clear offsets after bad event loop

Only do this if Debezium is stuck retrying malformed old events after a config fix.

```cmd
docker stop debezium-top-2018
docker stop debezium-top-2018-n
```

Delete:

```text
C:/odoo-cdc/data_top_2018/offsets.dat
C:/odoo-cdc/data_top_2018_n/offsets.dat
```

Restart containers.

## 18.6 Monitor slot backlog risk

Run on source:

```sql
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name IN ('top_2018_metabase_slot','top_2018_n_metabase_slot');
```

If Debezium is stopped for a long time, WAL can accumulate because replication slots retain WAL until consumed.

---

# Phase 19 — Common Troubleshooting

## 19.1 Node returns 401 Unauthorized

Cause:

```text
Debezium x-cdc-secret does not match VPS CDC_SECRET.
```

Fix:

```bash
pm2 delete odoo-cdc-ingest
CDC_SECRET='CORRECT_SECRET' PGHOST='127.0.0.1' PGPORT='5432' PGDATABASE='odoo_analytics' PGUSER='postgres' PGPASSWORD='PASSWORD' pm2 start server.js --name odoo-cdc-ingest
pm2 save
```

Update Debezium config to use same secret.

## 19.2 Decimal struct errors

Symptom:

```text
{"scale":1,"value":"..."}
```

Fix:

```properties
debezium.source.decimal.handling.mode=double
```

## 19.3 Timestamp out-of-range errors

Fix:

```properties
timestamp.mode=iso_offset
```

And keep Node helpers:

```text
microsToTimestamp()
daysToIsoDate()
```

## 19.4 Slot plugin error

If slot creation fails:

```sql
SELECT * FROM pg_create_logical_replication_slot('test_slot', 'wal2json');
```

Then wal2json is not installed or not loadable by PostgreSQL.

## 19.5 Debezium says publication missing

This means the wrong config was used.

For PostgreSQL 9.5, remove:

```properties
debezium.source.publication.name=...
```

Use:

```properties
debezium.source.plugin.name=wal2json
```

---

# End Of Runbook

