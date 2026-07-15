<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# Apache Ossie - Core Metadata Specification

> **DRAFT version** — in development, schema may change before 0.2.0 is released.

**Version:** 0.2.0.dev0

## Goals

- **Standardization**: Establish uniform language and structure for semantic model definitions, ensuring consistency and ease of interpretation across various tools and systems.
- **Extensibility**: Support domain-specific extensions while maintaining core compatibility.
- **Interoperability**: Enable exchange and reuse across different AI and BI applications.

## Table of Contents

1. [Enumerations](#enumerations)
2. [Semantic Model](#semantic-model)
3. [Datasets](#datasets)
4. [Relationships](#relationships)
5. [Fields](#fields)
6. [Metrics](#metrics)
7. [Examples](#examples)

---

## Enumerations

Standard enumeration values used throughout the specification.

### Dialects

Supported SQL and expression language dialects for metrics and field definitions.

| Dialect | Description |
|---------|-------------|
| `ANSI_SQL` | Standard SQL dialect |
| `SNOWFLAKE` | Snowflake SQL |
| `MDX` | Multi-Dimensional Expressions |
| `TABLEAU` | Tableau calculations |
| `DATABRICKS` | Databricks SQL |
| `MAQL` | GoodData MAQL (Metric Analysis and Query Language) |
| `BIGQUERY` | Google BigQuery (GoogleSQL) |

## Semantic Model

The top-level container that represents a complete semantic model, including datasets, relationships, and  metrics.

### Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the semantic model |
| `description` | string | No | Human-readable description |
| `ai_context` | string/object | No | Additional context for AI tools (e.g., custom instructions) |
| `datasets` | array | Yes | Collection of logical datasets (fact and dimension tables) |
| `relationships` | array | No | Defines how logical datasets are connected |
| `metrics` | array | No | Quantifiable measures defined as aggregate expressions on fields from logical datasets |
| `custom_extensions` | array | No | Vendor-specific attributes for extensibility |

### Example

```yaml
semantic_model:
  - name: sales_analytics
    description: Sales and customer analytics model
    ai_context:
      instructions: "Use this model for sales analysis and customer insights"
    datasets: []
    relationships: []
    metrics: []
    custom_extensions:
      - vendor_name: DBT
        data: '{"project_name": "tpcds_analytics", "models_path": "models/semantic"}'
```

---

## Datasets

Logical datasets represent business entities or concepts (fact and dimension tables). They contain fields and define the structure of the data.

### Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the dataset |
| `source` | string | Yes | Reference to underlying physical table/view (e.g., `database.schema.table`) or query |
| `primary_key` | array | No | Primary key columns that uniquely identify rows (single or composite) |
| `unique_keys` | array of arrays | No | Array of unique key definitions (each can be single or composite) |
| `description` | string | No | Human-readable description |
| `ai_context` | string/object | No | Additional context for AI tools (e.g., synonyms, common terms) |
| `fields` | array | No | Row-level attributes for grouping, filtering, and metric expressions |
| `custom_extensions` | array | No | Vendor-specific attributes |

### Primary Key Examples

```yaml
# Simple primary key
primary_key: [customer_id]

# Composite primary key
primary_key: [order_id, line_number]
```

### Unique Keys Examples

```yaml
# Multiple unique keys (each can be simple or composite)
unique_keys:
  - [email]                    # Simple unique key
  - [first_name, last_name]    # Composite unique key
```

### Example

```yaml
datasets:
  - name: orders
    source: sales.public.orders
    primary_key: [order_id]
    unique_keys:
      - [order_id]
      - [order_number]
    description: Order transactions
    ai_context:
      synonyms:
        - "purchases"
        - "sales"
    fields: []
    custom_extensions:
      - vendor_name: DBT
        data: '{"materialized": "table"}'
```

---

## Relationships

Relationships define how logical datasets are connected through foreign key constraints. They support both simple and composite keys.

### Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the relationship |
| `from` | string | Yes | The logical dataset on the many side of the relationship |
| `to` | string | Yes | The logical dataset on the one side of the relationship |
| `from_columns` | array | Yes | Array of column names in the "from" dataset (foreign key columns) |
| `to_columns` | array | Yes | Array of column names in the "to" dataset (primary or unique key columns) |
| `ai_context` | string/object | No | Additional context for AI tools |
| `custom_extensions` | array | No | Vendor-specific attributes |

### Important Notes

- The order of columns in `from_columns` must correspond to the order in `to_columns`
- Both arrays must have the same number of columns
- For simple relationships, use a single column: `[column1]`
- For composite relationships, use multiple columns: `[column1, column2]`

### Examples

**Simple Relationship:**

```yaml
- name: orders_to_customers
  from: orders
  to: customers
  from_columns: [customer_id]
  to_columns: [id]
```

**Composite Relationship:**

```yaml
# order_lines.product_id = products.id AND order_lines.variant_id = products.variant_id
- name: order_lines_to_products
  from: order_lines
  to: products
  from_columns: [product_id, variant_id]
  to_columns: [id, variant_id]
```

---

## Fields

Fields represent row-level attributes that can be used for grouping, filtering, and in metric expressions. They can be simple column references or computed expressions.

### Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the field within the dataset |
| `expression` | object | Yes | Expression definition with dialect support |
| `dimension` | object | No | Dimension metadata (e.g., `is_time` flag) |
| `label` | string | No | Label for categorization |
| `display_label` | string | No | Human-readable display name for UI and AI interaction |
| `description` | string | No | Human-readable description |
| `ai_context` | string/object | No | Additional context for AI tools (e.g., synonyms) |
| `semantic_type` | string | No | High-level semantic classification (see [Semantic Type](#semantic-type)) |
| `measurement` | object | No | Unit and quantity metadata (see [Measurement](#measurement)) |
| `display_format` | string | No | Excel-compatible format string (e.g., `$#,##0.00`, `0.0%`) |
| `default_aggregation` | string | No | Default aggregation when used as a measure: `sum`, `avg`, `min`, `max`, `count`, `count_distinct` |
| `default_sort` | object | No | Default sorting behavior (see [Default Sort](#default-sort)) |
| `default_time_granularity` | string | No | Default time bucket for temporal fields: `day`, `week`, `month`, `quarter`, `year` |
| `semantic_mappings` | array | No | Links to external ontologies (see [Semantic Mappings](#semantic-mappings)) |
| `hidden` | boolean | No | Whether this field should be hidden from consumer UIs |
| `group_label` | string | No | Organizational grouping label for UI presentation |
| `custom_extensions` | array | No | Vendor-specific attributes |

### Expression Object

The expression object supports multiple SQL dialects for cross-platform compatibility. Each field can define expressions in different dialects.

**Structure:**

```yaml
expression:
  dialects:
    - dialect: ANSI_SQL  # Must be one of the dialects enum values
      expression: "customer_id"  # Scalar SQL expression
```

**Key Points:**

- Use scalar SQL expressions (no aggregations)
- Can be simple column references (e.g., `customer_id`) or computed expressions (e.g., `first_name || ' ' || last_name`)
- Multiple dialect versions can be provided for the same field

### Dimension Object

| Field | Type | Description |
|-------|------|-------------|
| `is_time` | boolean | Indicates if this is a time-based dimension for temporal filtering |

### Examples

**Simple Column Reference for a Dimension:**

```yaml
- name: customer_id
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: customer_id
  description: Customer identifier
  dimension:
    is_time: false
```

**Computed Field:**

```yaml
- name: full_name
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: first_name || ' ' || last_name
  description: Customer full name
  ai_context:
    synonyms:
      - "name"
      - "customer name"
```

**Time Dimension:**

```yaml
- name: order_date
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: order_date
  dimension:
    is_time: true
  description: Date when order was placed
  ai_context:
    synonyms:
      - "purchase date"
      - "transaction date"
```

**Multi-Dialect Field:**

```yaml
- name: email_normalized
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: LOWER(email)
      - dialect: SNOWFLAKE
        expression: LOWER(email)::VARCHAR
      - dialect: BIGQUERY
        expression: SAFE_CAST(LOWER(email) AS STRING)
  description: Normalized email address
```

---

## Metrics

Quantitative measures defined on business data, representing key calculations like sums, averages, ratios, etc. Metrics are defined at the semantic model level and can  span multiple datasets.

### Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the metric |
| `expression` | object | Yes | Expression definition with dialect support |
| `description` | string | No | Human-readable description of what the metric measures |
| `ai_context` | string/object | No | Additional context for AI tools (e.g., synonyms) |
| `display_label` | string | No | Human-readable display name for UI and AI interaction |
| `semantic_type` | string | No | High-level semantic classification (see [Semantic Type](#semantic-type)) |
| `measurement` | object | No | Unit and quantity metadata (see [Measurement](#measurement)) |
| `display_format` | string | No | Excel-compatible format string (e.g., `$#,##0.00`, `0.0%`) |
| `desired_direction` | string | No | KPI polarity: `higher_is_better`, `lower_is_better`, `neutral` |
| `default_sort` | object | No | Default sorting behavior (see [Default Sort](#default-sort)) |
| `semantic_mappings` | array | No | Links to external ontologies (see [Semantic Mappings](#semantic-mappings)) |
| `hidden` | boolean | No | Whether this metric should be hidden from consumer UIs |
| `group_label` | string | No | Organizational grouping label for UI presentation |
| `custom_extensions` | array | No | Vendor-specific attributes |

### Expression Object

The expression object supports multiple dialects

```yaml
expression:
  dialects:
  - dialect: ANSI_SQL  # Default
    expression: "SUM(order.sales) / COUNT(DISTINCT order.customer_id)"
```

### Examples

**Simple Aggregation:**

```yaml
- name: total_revenue
  expression:
    - dialect: ANSI_SQL
      expression: SUM(orders.amount)
  description: Total revenue across all orders
  ai_context:
    synonyms:
      - "total sales"
      - "revenue"
```

**Cross-Dataset Metric:**

```yaml
- name: avg_orders
  expression:
    - dialect: ANSI_SQL
      expression: SUM(orders.amount) / COUNT(DISTINCT customers.id)
  description: Average orders
  ai_context:
    synonyms:
      - "Order Average by customer"
```

---

## Extended Metadata Types

The following types are used by the extended metadata fields on both fields and metrics. All extended metadata is **optional** and **non-executional** — it does not affect query execution but enables consumers (BI tools, AI agents, developers) to correctly interpret, render, and present data.

### Semantic Type

High-level classification of a field or metric value.

| Value | Description |
|-------|-------------|
| `categorical` | Unordered categorical values (e.g., status, color) |
| `quantitative` | Numeric values representing quantities |
| `monetary` | Currency/financial values |
| `temporal` | Time or date values |
| `geographic` | Location-related values (country, lat/lng, region) |
| `ordinal` | Ordered categorical values (e.g., Low/Medium/High, ratings) |
| `identifier` | Unique identifiers (e.g., IDs, codes) |

### Measurement

Describes what a numeric value represents. Enables unit-aware reasoning and formatting.

| Field | Type | Description |
|-------|------|-------------|
| `quantity_kind` | string | The kind of quantity (e.g., `currency`, `length`, `weight`, `temperature`, `percentage`, `duration`) |
| `unit` | string | Specific unit. For currency, use ISO 4217 codes (e.g., `usd`, `eur`, `gbp`). For others, use UCUM or descriptive strings (e.g., `meters`, `kg`, `celsius`) |
| `unit_system` | string | Unit system: `si`, `imperial`, `custom` |

**Example:**

```yaml
measurement:
  quantity_kind: currency
  unit: usd
  unit_system: custom
```

### Default Sort

Defines default sorting behavior for a field or metric.

| Field | Type | Description |
|-------|------|-------------|
| `direction` | string | Sort direction: `asc`, `desc` |
| `nulls` | string | Null positioning: `first`, `last` |
| `by_field` | string | Sort by a different field (e.g., sort month names by month number) |

**Example:**

```yaml
default_sort:
  direction: desc
  nulls: last
```

### Semantic Mappings

Links a field or metric to external ontologies or standards. Enables semantic interoperability and knowledge graph integration.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | string | Yes | Ontology or standard name (e.g., `schema.org`, `FIBO`) |
| `identifier` | string | Yes | URI or identifier within that ontology |

**Example:**

```yaml
semantic_mappings:
  - source: schema.org
    identifier: https://schema.org/MonetaryAmount
```

### Display Format

The `display_format` string follows Excel-compatible custom number format conventions. Consumers MAY support a subset, but interoperability is improved when adhering to common patterns.

**Common Patterns:**

| Pattern | Description | Example Output |
|---------|-------------|----------------|
| `$#,##0.00` | Currency with 2 decimals | $1,234.56 |
| `#,##0` | Integer with grouping | 12,345 |
| `0.0%` | Percentage with 1 decimal | 12.3% |
| `#,##0.00;(#,##0.00)` | Positive/negative | 1,234.56 or (1,234.56) |
| `0.00E+00` | Scientific notation | 1.23E+04 |
| `yyyy-mm-dd` | Date format | 2024-01-15 |

---

## Extended Metadata Examples

**Field with full extended metadata:**

```yaml
- name: sales_amount
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: sales_amount
  display_label: "Sales Amount"
  semantic_type: monetary
  measurement:
    quantity_kind: currency
    unit: usd
  display_format: "$#,##0.00"
  default_aggregation: sum
  default_sort:
    direction: desc
    nulls: last
  semantic_mappings:
    - source: schema.org
      identifier: https://schema.org/MonetaryAmount
  group_label: "Revenue"
```

**Metric with extended metadata:**

```yaml
- name: total_sales
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(orders.sales_amount)
  display_label: "Total Sales"
  description: Total revenue from all completed orders
  semantic_type: monetary
  measurement:
    quantity_kind: currency
    unit: usd
  display_format: "$#,##0.00"
  desired_direction: higher_is_better
  default_sort:
    direction: desc
  group_label: "Revenue"
```

**Temporal field with time granularity:**

```yaml
- name: order_date
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: order_date
  dimension:
    is_time: true
  display_label: "Order Date"
  semantic_type: temporal
  default_time_granularity: month
  default_sort:
    direction: desc
```

**Hidden field used only in expressions:**

```yaml
- name: internal_cost_basis
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: raw_cost * adjustment_factor
  hidden: true
  description: Internal cost calculation used by margin metrics
```

---

## Custom Extensions

Custom extensions allow vendors to add platform-specific metadata without breaking core compatibility. Each extension includes a vendor name and arbitrary JSON data.

### Schema

```yaml
custom_extensions:
  - vendor_name: string  # Free-form string identifying the vendor
    data: string         # JSON string containing vendor-specific data
```

### Vendor Names

The `vendor_name` field is a free-form string, allowing any vendor or organization to
define custom extensions without requiring changes to the core specification.

The following are well-known examples:

| Vendor | Description |
|--------|-------------|
| `COMMON` | Common/standard extensions |
| `SNOWFLAKE` | Snowflake-specific attributes |
| `SALESFORCE` | Salesforce/Tableau-specific attributes |
| `DBT` | dbt-specific attributes |
| `DATABRICKS` | Databricks-specific attributes |
| `GOODDATA` | GoodData-specific attributes |
| `HONEYDEW` | Honeydew-specific attributes |

### Examples

**Snowflake Extension:**

```yaml
- vendor_name: SNOWFLAKE
  data: '{
    "warehouse": "ANALYTICS_WH",
    "database": "PROD",
    "schema": "PUBLIC"
  }'
```

**Salesforce Extension:**

```yaml
- vendor_name: SALESFORCE
  data: '{
    "tableau_workbook_id": "sales_dashboard",
    "einstein_enabled": true,
    "crm_sync": {
      "enabled": true,
      "sync_frequency": "daily"
    }
  }'
```

**DBT Extension:**

```yaml
- vendor_name: DBT
  data: '{
    "project_name": "analytics",
    "materialized": "table",
    "tags": ["daily", "core"]
  }'
```

**Databricks Extension:**

```yaml
- vendor_name: Databricks
  data: '{
    "default_catalog": "finance",
    "default_schema": "gold"
  }'
```

---

## Complete Example

Here's a complete semantic model example showing all components working together:

```yaml
semantic_model:
  - name: ecommerce_analytics
    description: E-commerce sales and customer analytics
    ai_context:
      instructions: "Use this model for analyzing sales trends, customer behavior, and product performance"

    datasets:
      - name: orders
        source: sales.public.orders
        primary_key: [order_id]
        description: Customer orders
        fields:
          - name: order_id
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: order_id
            description: Order identifier

          - name: customer_id
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: customer_id
            description: Customer identifier

          - name: order_date
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: order_date
            dimension:
              is_time: true
            description: Order date

          - name: amount
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: amount
            description: Order amount

      - name: customers
        source: sales.public.customers
        primary_key: [id]
        description: Customer information
        fields:
          - name: id
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: id
            description: Customer identifier

          - name: email
            expression:
              dialects:
                - dialect: ANSI_SQL
                  expression: email
            description: Customer email

    relationships:
      - name: orders_to_customers
        from: orders
        to: customers
        from_columns: [customer_id]
        to_columns: [id]

    metrics:
      - name: total_revenue
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: SUM(orders.amount)
        description: Total revenue from all orders
        ai_context:
          synonyms:
            - "total sales"
            - "revenue"

      - name: customer_count
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: COUNT(DISTINCT customers.id)
        description: Total number of customers
        ai_context:
          synonyms:
            - "total customers"
            - "customer base"

    custom_extensions:
      - vendor_name: SNOWFLAKE
        data: '{"warehouse": "ANALYTICS_WH"}'
```

---

## AI Context Structure

The `ai_context` field can be either a simple string or a structured object with specific keys:

**Simple String:**

```yaml
ai_context: "orders, purchases, sales"
```

**Structured Object:**

```yaml
ai_context:
  instructions: "Use this for sales analysis"
  synonyms:
    - "orders"
    - "purchases"
    - "sales"
  examples:
    - "Show total sales last month"
    - "What's the revenue by region?"
```

### Recommended AI Context Fields

| Field | Type | Description |
|-------|------|-------------|
| `instructions` | string | Instructions for AI on how to use this entity |
| `synonyms` | array | Alternative names and terms |
| `examples` | array | Sample questions or use cases |

---

## Version History

- **0.2.0.dev0** (Unreleased): In-development next minor release. Schema is mutable; do not depend on this version in production.
- **0.1.1** (2025-12-11): Initial release
  - Core semantic model structure
  - Support for datasets, relationships, fields, and metrics
  - Multi-dialect metric expressions
  - Vendor extensibility framework
  - Context for agents

---

## License

See LICENSE file for details.
