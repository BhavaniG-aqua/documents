# Afresh Technologies – Complete Business Overview

## 1. Company Overview

### What is Afresh?

Afresh Technologies is the leading AI-powered platform company purpose-built for grocery retail, specifically focused on the **fresh food supply chain**. Founded nearly a decade ago, Afresh addresses one of the most complex problems in retail: managing perishable inventory where shelf life is short, demand is unpredictable, and data is often messy and incomplete.

The company has developed the world's first AI engine designed specifically for fresh grocery, leveraging machine learning models (including hidden Markov models and neural networks) trained on grocery and fresh-specific data to optimize decisions across the entire fresh supply chain.

### Key Metrics & Market Position

- **Live in 12,500+ departments** across 40 US states
- **+3% sales increase** for retail partners
- **+25% shrink reduction** (waste reduction)
- **80% stockout reduction**
- **+20% labor efficiency improvement**
- **+7% faster inventory turns**
- **94% adherence rate** from store managers using the platform
- **Raised $34M** to scale AI across the grocery industry (as of 2025)

### Major Retail Partners

- **Albertsons Companies** – one of the largest US grocery chains
- **Meijer** – Midwest-based supercenter chain
- **Wakefern Food Corp** – retailer-owned cooperative (ShopRite, Price Rite Marketplace, The Fresh Grocer, Morton Williams, Fairway Market)
- **Stater Bros.** – Southern California chain with 169 stores

### Platform Expansion

Originally built for the fresh perimeter (produce, meat, seafood, deli, bakery), Afresh has expanded its platform to cover **every department** across the grocery enterprise, including center store, frozen, general merchandise, and health & beauty. The company also launched the industry's first **AI-powered fresh buying solution** for distribution centers (DCs).

---

## 2. The Afresh Platform

### Core Capabilities

The Afresh Platform is an interconnected suite of AI/ML solutions that optimize decision-making across:

| Function | Description |
|----------|-------------|
| **Store Ordering** | AI-guided replenishment recommendations for store managers |
| **Inventory Management** | Real-time perpetual inventory estimation accounting for spoilage and misscans |
| **Production Planning** | Optimization of in-store prepared food production |
| **DC Forecasting** | Distribution center demand forecasting for fresh buying |
| **Merchandising** | Data-driven insights for product placement and assortment |
| **Operations** | Workflow optimization and labor efficiency improvements |

### Technical Differentiation

Traditional inventory systems rely on static counts and simple demand forecasting. Afresh's approach:

1. **Accounts for perishability** – Models spoilage, expiration, and quality degradation
2. **Handles imperfect data** – Designed for the messiness inherent to grocery data (misscans, theft, damage)
3. **Uses probabilistic models** – Hidden Markov models to estimate true inventory state
4. **Neural network forecasting** – Demand prediction that accounts for seasonality, promotions, weather, and local events
5. **Continuous learning** – Every new retailer, category, and daily decision strengthens the models

### The App Experience

Store managers interact with Afresh through a **mobile app** that provides:
- Daily order recommendations
- Order guide management (what items to order from which vendors)
- Inventory visibility
- Production planning for prepared foods (deli, bakery)
- Workflow execution and task management

---

## 3. Data Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUSTOMER SOURCE SYSTEMS                        │
│   (POS, ERP, Warehouse Management, Vendor Systems, SFTP feeds)   │
└──────────────────────────────┬──────────────────────────────────┘
                               │ (SFTP / API / Database Export)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATA INGESTION LAYER                           │
│         (Raw data landing → Customer-specific schemas)           │
└──────────────────────────────┬──────────────────────────────────┘
                               │ (dbt Transformation)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│              CANONICAL INTEGRATION SCHEMA (20+ tables)            │
│    (Standardized data model all customers must map to)           │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                  AFRESH PRODUCT DATABASE (Postgres)               │
│      (Application data + user-generated orders/workflows)        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AFRESH AI/ML PLATFORM                          │
│  (Demand forecasting, inventory estimation, recommendations)     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     STORE MANAGER APP                             │
│    (Order recommendations, inventory views, task execution)      │
└─────────────────────────────────────────────────────────────────┘
```

### The Canonical Data Model

Afresh defines a **standardized data model** (canonical schema) that every new customer's data must be transformed into. This ensures the AI/ML models can work consistently regardless of how different retailers structure their data internally.

The canonical model contains **22+ integration tables** organized into these domains:

#### Item/Product Domain
| Table | Purpose |
|-------|---------|
| `ORDERABLE_ITEMS` | Core catalog of items that can be ordered from vendors/DCs |
| `RETAIL_ITEMS` | Items sellable in stores (from POS, may differ from orderable) |
| `ITEM_MAPPING_LINKS` | Links orderable ↔ retail IDs daily to generate Afresh Item IDs |
| `RETAIL_ITEMS_METADATA` | Metadata for conflict resolution when mapping items |
| `RETAIL_ITEM_TAXONOMIES` | 3-level hierarchy: department → class → subclass |
| `TAXONOMIES` | Master list of all valid taxonomy values |

#### Ordering & Workflow Domain
| Table | Purpose |
|-------|---------|
| `ORDER_GUIDE_ITEMS` | What items are available to order per store-workflow per day |
| `ORDERABLE_ITEMS_BLOCKED` | Items blocked from ordering (but still visible) |
| `ORDER_DELIVERY_SCHEDULES` | When stores receive deliveries (explicit date feeds) |
| `ORDER_DELIVERY_ITEM_GROUPINGS` | Groups items by vendor/region for delivery scheduling |

#### Financial Domain
| Table | Purpose |
|-------|---------|
| `BASE_PRICES` | Standard list prices for retail items |
| `BASE_PRICE_TYPES` | Customer-defined price types with priorities |
| `PROMOTION_PRICES` | Promotional/sale prices distinct from base |
| `PROMOTION_PRICE_TYPES` | Promotion type definitions with metadata |
| `ORDERABLE_ITEMS_COST` | Historical per-store costs for ordering items (time series) |

#### Logistics Domain
| Table | Purpose |
|-------|---------|
| `SHIPMENTS` | All received shipments (warehouse + DSD) at store-item-day level |
| `PLANNED_SHIPMENTS` | Anticipated future shipments |
| `RETAIL_ITEM_SALES` | Daily sales data per retail item per store |

#### Store Domain
| Table | Purpose |
|-------|---------|
| `STORES` | Store metadata (address, coordinates, timezone, region, banner) |

#### Transformation/Recipe Domain
| Table | Purpose |
|-------|---------|
| `IN_STORE_PREPARED_ITEMS` | Prepared items and their ingredient recipes |
| `ORDERABLE_ITEM_TRANSFORMATIONS` | Source → ingredient mappings with yield percentages |
| `ITEM_UNIT_CONVERSIONS` | Conversion factors (e.g., cups → pounds) |

### Key Data Concepts

- **Orderable Item**: A unit of goods that can be ordered from a vendor/DC. Has a case size, UOM, vendor ID.
- **Retail Item**: A sellable item in stores (from POS). Has customer IDs and metadata.
- **Afresh Item ID**: Generated internally by building a graph of ID relationships from `ITEM_MAPPING_LINKS`. This unifies different item ID types into a single identity.
- **Scope**: Many tables use scope (store-specific vs. chain-wide) to handle items that may have different attributes per store.
- **Time Series Data**: Cost, sales, shipments are tracked as time series to enable trend analysis.

---

## 4. Customer Onboarding Process

### Why Onboarding is Complex

Every grocery retailer has different:
- Source systems (ERP, POS, warehouse management, vendor portals)
- Data formats and naming conventions
- Item hierarchies and ID systems
- Delivery schedules and logistics workflows
- Business rules and exceptions

Afresh must transform each customer's unique data landscape into its standardized canonical model. This is a **significant data engineering effort** that historically takes months of manual work.

### The Onboarding Lifecycle

```
Phase 1: DISCOVERY (Weeks 1-2)
├── Business stakeholder interviews
├── Source system inventory
├── Data feed identification (SFTP, API, DB exports)
├── Sample data collection
└── Understand customer-specific business rules

Phase 2: DESIGN (Weeks 2-4)
├── Integration Requirement Document (IRD)
├── Source-to-target mapping specification
├── Data quality rules definition
├── Transformation logic documentation
└── Architecture and TCO walkthrough

Phase 3: DEVELOPMENT (Weeks 4-12+)
├── dbt transformation code development
├── Data quality check implementation
├── End-to-end pipeline construction
├── Iterative testing with customer
└── Code review and refinement

Phase 4: TESTING & VALIDATION (Weeks 10-14)
├── Data quality validation
├── Ground rules validation
├── Production milestone walkthrough
├── Bug fixing and edge case handling
└── Performance optimization

Phase 5: HANDOVER (Week 14+)
├── Production deployment
├── Documentation delivery
├── Knowledge transfer sessions
├── Future enhancement proposal
└── Ongoing support setup
```

### Ground Rules

Afresh maintains **50+ business validation rules** ("ground rules") organized by subject area:

- **Items**: Unique IDs, required fields (vendor ID, case size, UOM, descriptions), item relationships
- **Stores**: Data contracts, timeliness requirements
- **Delivery Schedules**: Store-vendor-date completeness, 14-day future coverage
- **Shipments**: Store/item/vendor/quantity required, handling of shorts and negative shipments, deduplication rules
- **Pricing**: Base price and cost requirements

These rules are validated both programmatically (SQL queries) and semantically (LLM-based reasoning).

---

## 5. The Engagement (ZEB × Afresh)

### Context

This workspace documents an engagement between **ZEB** (a consulting/technology firm) and **Afresh Technologies**. The engagement runs **two parallel initiatives**:

### Initiative 1: Data Engineering Track

**Goal**: Complete a real customer onboarding using Afresh's existing (and new) tooling.

**Team**:
- **Pandi** – Data Engineering Lead (ZEB)
- **Andy** – Data Engineering (ZEB)
- **Yoga** – Data Engineering (ZEB)
- **Jake** – Main POC (Afresh)
- **Kevin Matthew** – Solution Engineering Lead (Afresh)
- **Daniel Alpert** – Data Engineering Lead (Afresh)

**Deliverables**:
- Source data analysis and documentation
- Source-to-canonical mapping specification
- dbt transformation code (using AI-generated code as starting point)
- Data quality validation and testing
- Production-ready pipeline

### Initiative 2: AI/ML Innovation Track

**Goal**: Optimize and automate the customer onboarding process using LLM/AI, reducing the months-long manual effort.

**Team**:
- **Harish** – Solution Architect (ZEB)
- **Sidh** – AI/ML Practices Lead (ZEB, offshore)
- **Naga** – AI/ML Practices (ZEB, offshore)
- **Mukhilesh** – AI/ML Engineer (ZEB)
- **Bamai (Bhavani)** – AI/ML Engineer (ZEB)
- **David** – ML Platform Manager (Afresh)
- **Thurva** – ML Engineer (Afresh)

**Deliverables**:
- Customer onboarding automation framework (the POC code)
- AI-powered data mapping generation
- Automated dbt code generation
- Data quality validation automation
- Detailed roadmap for AI incorporation
- MLOps assessment and optimization recommendations

### Communication & Tools

- **Communication**: Slack (dedicated channel)
- **Documentation**: Notion (Afresh's internal wiki)
- **Diagramming**: Miro
- **Development**: Databricks (Azure-hosted), GitHub (`afresh-technologies/integration-assistant`)
- **Data Transformation**: dbt (Data Build Tool) on Databricks
- **Meeting Cadence**: Daily syncs initially, then split by track

### The "BGC" Customer

The workspace references **"BGC"** as the customer being onboarded during this engagement. Their source data includes:
- base_prices, categories, delivery_schedules, delivery_windows
- discards, hotspots, in_store_prepared_items, item_unit_conversions
- items, orderable_item_transformations, period_end_inventory
- planned_shipments, promotions, recipe_ingredients, recipe_sellable
- sales, shipments, store_to_store_transfers, stores

This data lives in `havasu_local.zeb_bgc_raw` and transforms into `havasu_local.zeb_bgc_integ`.

---

## 6. Technology Stack

| Component | Technology |
|-----------|------------|
| **Cloud** | Azure |
| **Data Platform** | Databricks (Unity Catalog) |
| **Data Transformation** | dbt (Data Build Tool) with dbt-databricks adapter |
| **Data Format** | Delta Lake |
| **AI/LLM Models** | Claude Sonnet 4, Claude 3.7 Sonnet, GPT-4 OSS 120B, GPT-4 OSS 20B |
| **Vector Search** | Databricks Vector Search |
| **Source Control** | GitHub (`afresh-technologies/integration-assistant`) |
| **Orchestration** | Databricks Jobs (serverless compute) |
| **SQL Warehouse** | Databricks SQL Warehouse |
| **Production Database** | PostgreSQL (application data) |

---

## 7. Why This Matters

The customer onboarding bottleneck is Afresh's biggest constraint to growth. Each new retailer requires:
- Months of data engineering work
- Deep understanding of each retailer's unique data systems
- Manual mapping of hundreds of columns to the canonical model
- Iterative testing and validation
- Significant human expertise

By automating this with AI (the POC code in this workspace), Afresh aims to:
1. **Reduce onboarding time** from months to weeks
2. **Scale faster** to serve more retailers
3. **Reduce human error** in data mapping and transformation
4. **Standardize quality** across onboarding projects
5. **Free up data engineers** to focus on complex edge cases rather than routine mappings

---

*Sources: [Afresh Technology Platform](https://www.afresh.com/technology), [Afresh Company Overview](https://www.afresh.com/company/about), [Afresh $34M Fundraise Announcement](https://www.afresh.com/resources/afresh-raises-34m), [Afresh Platform Expansion](https://www.afresh.com/resources/afresh-expands-ai-platform-to-cover-every-item-in-stores), workspace project documentation. Content was rephrased for compliance with licensing restrictions.*
