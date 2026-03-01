# Real-Time E-Commerce Order Processing Pipeline
### Confluent Kafka + Apache Spark Structured Streaming + Delta Lake on Databricks

---

## Project Overview

An end-to-end real-time order processing pipeline that simulates an e-commerce platform. Orders are produced to Confluent Kafka, consumed and validated using Spark Structured Streaming, written to Delta Lake tables, and visualized on a Databricks Lakeview Dashboard.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATABRICKS                               │
│                                                                 │
│  spark_order_producer                                           │
│  (generates random orders)                                      │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────────┐                    │
│  │         CONFLUENT KAFKA (Azure)         │                    │
│  │                                         │                    │
│  │  orders topic (3 partitions)            │                    │
│  │  ├── partition 0 → normal users         │                    │
│  │  ├── partition 1 → normal users         │                    │
│  │  └── partition 2 → premium users        │                    │
│  │                                         │                    │
│  │  dead_letter topic (failed orders)      │                    │
│  │  notifications topic                    │                    │
│  └─────────────────────────────────────────┘                    │
│         │                                                       │
│         ▼                                                       │
│  stream_order_processor                                         │
│  (readStream → validate → foreachBatch → Delta Lake)            │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────────┐                    │
│  │         DELTA LAKE (sales_catalog)      │                    │
│  │  main schema    → operational tables    │                    │
│  │  reporting schema → analytics tables    │                    │
│  └─────────────────────────────────────────┘                    │
│         │                                                       │
│         ▼                                                       │
│  Lakeview Dashboard                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Message Broker | Confluent Kafka (Azure eastus) |
| Stream Processing | Apache Spark Structured Streaming |
| Storage | Delta Lake (Databricks Unity Catalog) |
| Compute | Databricks Serverless |
| Orchestration | Databricks Workflows (Jobs) |
| Visualization | Databricks Lakeview Dashboard |
| Language | Python (PySpark) |

---

## Project Structure

```
Confluent_Kafka_Project/
├── init/
│   ├── kafka_config              # loads secrets, sets catalog, configs
│   └── setup_tables              # creates all Delta tables
├── data/
│   └── insert_sample_data        # inserts sample users, items, inventory
├── producer/
│   ├── order_producer            # confluent-kafka producer (learning)
│   └── spark_order_producer      # spark native producer
├── consumer/
│   ├── order_processor           # confluent-kafka consumer (learning)
│   └── stream_order_processor    # spark structured streaming consumer
└── utils/
    ├── reset_pipeline            # clears tables, resets data
    ├── add_master_data           # add users, items, inventory
    └── reporting_views           # builds reporting schema tables
```

---

## Confluent Kafka Setup

### Cluster
- **Provider:** Azure
- **Region:** eastus
- **Type:** Basic cluster

### Topics

| Topic | Partitions | Purpose |
|---|---|---|
| `orders` | 3 | incoming order events |
| `dead_letter` | 1 | failed/invalid orders |
| `notifications` | 1 | successful order notifications |

### Partition Routing Strategy

```
Normal users  → partition 0 or 1 (random load balancing)
Premium users → partition 2 (dedicated partition)
```

This ensures premium orders get dedicated throughput and are never delayed by normal order traffic.

### Secrets Configuration

All Kafka credentials stored in Databricks Secret Scope:
```
Scope: kafka_secrets
Keys:
  bootstrap_server  → Confluent bootstrap URL
  api_key           → Confluent API key
  api_secret        → Confluent API secret
```

---

## Delta Lake Schema

### Catalog: `sales_catalog`

#### Main Schema — Operational Tables

| Table | Columns | Purpose |
|---|---|---|
| `users` | user_id, username, email, balance | master user data |
| `items` | item_id, item_name, price | master item catalog |
| `inventory` | item_id, quantity_available | baseline inventory |
| `orders` | order_id, user_id, item_id, item_quantity, category, status, timestamp | placed orders |
| `order_logs` | order_id, user_id, item_id, item_quantity, status, user_balance_before, timestamp | all order attempts |
| `balance_events` | user_id, amount_change, order_id, timestamp | balance deduction events |
| `inventory_events` | item_id, quantity_change, order_id, timestamp | inventory deduction events |
| `notifications` | user_id, username, item_name, item_quantity, balance_after, timestamp | order notifications |

#### Reporting Schema — Analytics Tables

| Table | Purpose |
|---|---|
| `dim_users` | users with current balance and total spent |
| `dim_items` | items with current stock and total sold |
| `fact_orders` | all orders enriched with full date/time dimensions |
| `daily_summary` | orders and revenue aggregated by day |
| `hourly_summary` | orders and revenue aggregated by hour |
| `item_summary` | performance metrics per item |
| `user_summary` | behaviour and spend metrics per user |
| `failure_summary` | breakdown of order failure reasons |

### Event Sourcing Pattern

User balances and inventory are **never updated directly.** All changes flow through event tables:

```
Current Balance  = users.balance + SUM(balance_events.amount_change)
Current Stock    = inventory.quantity_available + SUM(inventory_events.quantity_change)
```

This provides full audit trail and exactly-once semantics with Spark checkpointing.

---

## Sample Data

### Users

| user_id | username | email | balance | category |
|---|---|---|---|---|
| U001 | alice | alice@gmail.com | 1500.00 | normal |
| U002 | bob | bob@gmail.com | 800.00 | normal |
| U003 | charlie | charlie@gmail.com | 300.00 | normal |
| U004 | diana | diana@gmail.com | 2000.00 | premium |
| U005 | eve | eve@gmail.com | 150.00 | normal |

### Items

| item_id | item_name | price |
|---|---|---|
| I001 | laptop | 999.99 |
| I002 | phone | 499.99 |
| I003 | tablet | 299.99 |
| I004 | headphones | 79.99 |
| I005 | charger | 29.99 |

---

## Notebooks

### `init/kafka_config`
Loaded at the start of every notebook via `%run ../init/kafka_config`.
- Loads Kafka credentials from Databricks secrets
- Sets Unity Catalog to `sales_catalog.main`
- Initializes `producer_conf` and `consumer_conf` dictionaries

### `init/setup_tables`
Run once to create all Delta tables.
- Uses `mode("ignore")` — safe to rerun, never overwrites existing data
- Creates all tables in `sales_catalog.main`

### `data/insert_sample_data`
Inserts baseline users, items, inventory using `DeltaTable.merge()`.
- Idempotent — safe to run multiple times, no duplicates

### `producer/spark_order_producer`
Generates random orders as a Spark DataFrame and writes to Kafka.

```python
# key design decisions:
# 1. UUID for order_id → no race conditions
# 2. partition column controls routing
# 3. write.format("kafka") → spark native, no extra library

kafka_df.write \
    .format("kafka") \
    .option("kafka.bootstrap.servers", bootstrap) \
    .option("topic", "orders") \
    .save()
```

### `consumer/stream_order_processor`
Core pipeline notebook. Reads from Kafka using Structured Streaming.

**Stream setup:**
```python
df_raw = spark.readStream \
    .format("kafka") \
    .option("subscribe", "orders") \
    .option("startingOffsets", "earliest") \
    .load()
```

**Validation chain (in order):**
1. Duplicate check — `left_anti` join against orders table
2. `FAILED_INVALID_USER` — user_id not in users table
3. `FAILED_INVALID_ITEM` — item_id not in items table
4. `FAILED_INVENTORY` — quantity_available < item_quantity
5. `FAILED_BALANCE` — balance < total_cost
6. `ORDER_PLACED` — all checks passed

**foreachBatch sink:**
```python
query = df_validated.writeStream \
    .foreachBatch(process_batch) \
    .option("checkpointLocation", checkpoint_path) \
    .trigger(availableNow=True) \
    .start()
```

**Key implementation notes:**
- `createOrReplaceTempView` used for materialization (serverless limitation — cache/persist not supported)
- Orders written **last** in foreachBatch to prevent lazy re-evaluation issue
- Static tables re-read fresh every batch for latest balance/inventory state
- `availableNow=True` trigger used for Databricks Community Edition compatibility

### `utils/add_master_data`
Utility notebook with Databricks widgets for adding master data.

**Supported actions (via widget dropdown):**
- `add_user` — adds new user to users table
- `add_item` — adds new item to items table
- `add_inventory` — set or restock inventory for an item
- `restock_all` — adds fixed quantity to all items
- `view_all` — displays current state of all master tables

**Running as Databricks Job:**
```
Parameters:
  action           = restock_all
  restock_quantity = 10
```

### `utils/reporting_views`
Builds the reporting schema by reading operational tables and writing enriched analytics tables. Overwrites all reporting tables on every run.

---

## Order Validation Flow

```
Kafka message arrives
        │
        ▼
Parse JSON → extract order fields
        │
        ▼
left_anti join vs orders table ──→ DUPLICATE? → skip
        │
        ▼
LEFT JOIN users table ──────────→ NULL username? → FAILED_INVALID_USER
        │
        ▼
LEFT JOIN items table ──────────→ NULL item_name? → FAILED_INVALID_ITEM
        │
        ▼
LEFT JOIN inventory table ──────→ stock < qty? → FAILED_INVENTORY
        │
        ▼
Calculate total_cost ───────────→ balance < cost? → FAILED_BALANCE
        │
        ▼
ORDER_PLACED ✓
        │
        ├──→ balance_events (deduction recorded)
        ├──→ inventory_events (stock deduction recorded)
        ├──→ notifications (user notified)
        ├──→ order_logs (audit trail)
        └──→ orders (placed order recorded)

Failed orders:
        ├──→ order_logs (failure reason recorded)
        └──→ dead_letter Kafka topic
```

---

## Databricks Job Pipeline

```
Job: order_pipeline (scheduled hourly)
│
├── Task 1: producer/spark_order_producer
│           generates 10 random orders
│           sends to Kafka orders topic
│           depends on: nothing
│
├── Task 2: consumer/stream_order_processor
│           reads new orders from Kafka
│           validates and writes to Delta tables
│           depends on: Task 1
│
├── Task 3: utils/add_master_data
│           action=restock_all, restock_quantity=10
│           restocks all items after each run
│           depends on: Task 2
│
└── Task 4: utils/reporting_views
            refreshes all reporting schema tables
            depends on: Task 3
```

---

## Dashboard

Built using Databricks Lakeview Dashboard on top of `sales_catalog.reporting` tables.

### Datasets Connected
- `daily_summary`
- `dim_items`
- `dim_users`
- `fact_orders`
- `failure_summary`
- `hourly_summary`
- `item_summary`
- `user_summary`

### Visualizations

| Chart | Type | Dataset | X axis | Y axis |
|---|---|---|---|---|
| Revenue by Hour | Bar | hourly_summary | hour | total_revenue |
| Orders by Hour | Bar | hourly_summary | hour | total_orders |
| Revenue by Item | Bar | item_summary | item_name | total_revenue |
| Current Stock | Bar | item_summary | item_name | current_stock |
| Top Spenders | Bar | user_summary | username | total_spent |
| Normal vs Premium | Pie | fact_orders | category | total_cost |
| Failure Reasons | Pie | failure_summary | status | total_failures |
| User Summary | Table | user_summary | all columns | — |

---

## Key Design Decisions

### Why UUID for order_id?
UUID prevents race conditions when multiple producers generate orders simultaneously. A `max+1` approach would require locking the orders table.

### Why Event Sourcing for Balance/Inventory?
Spark Structured Streaming only supports `append` output mode for joins — updating existing rows is not possible in streaming context. Event sourcing solves this by appending change events instead of updating master records.

### Why createOrReplaceTempView Instead of cache()?
Databricks Serverless does not support `cache()` or `persist()`. `createOrReplaceTempView` provides DataFrame materialization without caching.

### Why Write Orders Last in foreachBatch?
Spark DataFrames are lazy — every action re-executes the full computation plan. If orders were written first, the `left_anti` duplicate check would find those orders already written and return empty DataFrames for all subsequent writes.

### Why availableNow=True Trigger?
Databricks Community Edition does not support long-running streaming jobs. `availableNow=True` processes all available Kafka messages then stops cleanly.

---

## How to Run

### First Time Setup
```
1. Create Confluent Kafka cluster and topics
2. Store credentials in Databricks secret scope
3. Run init/setup_tables
4. Run data/insert_sample_data
```

### Run Pipeline Manually
```
1. Run producer/spark_order_producer
2. Run consumer/stream_order_processor
3. Run utils/add_master_data (action=restock_all)
4. Run utils/reporting_views
```

### Reset Everything
```
Run utils/reset_pipeline
→ clears all transaction tables
→ resets user balances to original values
→ resets inventory to original values
→ deletes checkpoint
```

### Add New User
```
Run utils/add_master_data
Widget: action = add_user
        user_id = U006
        username = frank
        email = frank@gmail.com
        balance = 500
```

### Add New Item + Inventory
```
Run utils/add_master_data
Widget: action = add_item
        item_id = I006
        item_name = smartwatch
        price = 199.99

Then:   action = add_inventory
        item_id = I006
        quantity = 30
```

---

---

## Author
hemasundher0000@gmail.com
