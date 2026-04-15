# 📡 Telecom CVM Recommendation Engine

A production-grade **Customer Value Management (CVM) recommendation engine** built for a regional telecom provider. The system generates personalized next-best-offer recommendations for thousands of customers daily by combining collaborative filtering, business rule logic, and customer segmentation.

📓 **[View Interactive Notebook →](https://ummuglsm0134.github.io/customer-retention-recommendation-engine/telecom_cvm_notebook.html)**

---

## Business Problem

Telecom providers lose revenue not just from churn, but from customers who stay on suboptimal plans. Identifying *which package to offer each customer, at the right time* requires understanding both behavioral patterns and business constraints.

This engine answers:
- Which customers are candidates for upsell or service upgrade?
- What packages have the highest affinity given a customer's current services and value tier?
- Which recommendations are technically eligible, margin-positive, and campaign-aligned?

---

## System Architecture

```
SQL Server DW ──► CVM Engine (Python + Ray) ──► Pattern Engine (collaborative filtering)
                                            └──► Rule Engine   (business rules + campaigns)
                                                      │
                                                      ▼
                                             SQL Server output tables
                                                      │
                                                      ▼
                                             C# / SignalR Web API
```

Two complementary engines run in parallel and write results to a shared data warehouse, consumed by a real-time customer service dashboard.

---

## How It Works

### 1 · Customer Segmentation (`CVM_Segment.py`)

Before recommendations are generated, each customer is assigned to a value tier (A–E) based on their monthly margin contribution, using quantile binning on the most recent month-end record.

- **5 segments** (A = highest value, E = lowest)
- Customers who *transition between segments* over time are tracked — these rank switches are the primary signal for the pattern-based engine
- Employee accounts are filtered out before segmentation

### 2 · Pattern-Based Engine — Collaborative Filtering (`package_recommender.py`)

Uses **item-item collaborative filtering** (cosine similarity on binary customer–package matrices) to identify package affinities, then scores each customer's top candidates via user-item recommendation.

- **Stage 1:** Build a customer × package binary matrix per segment → compute pairwise cosine similarities → store top-10 similar packages as a knowledge base
- **Stage 2:** For each customer in segments B–E, score candidate packages using weighted similarity and return the top 6 recommendations
- Segment A customers are excluded — they already hold the highest-value packages

### 3 · Rule-Based Engine — Business Logic (`RuleBasedRecom.py`)

Applies explicit business rules to filter and rank packages based on a customer's current services, technology tier (Copper / Coaxial / Fiber), and gross profit margin thresholds.

| Rule | Logic |
|------|-------|
| **Package rule** | Don't offer packages containing services the customer already has |
| **Standalone rule** | Offer à la carte options to single-product customers |
| **Pick-option rule** | Upsell to a higher tier matching the customer's current usage |
| **Rooftop rule** | Filter by location-based service eligibility |
| **Campaign rule** | Inject seasonal / promotional packages where applicable |

### 4 · Parallel Processing (Ray)

Both engines use **Ray remote tasks** to process customers in batches of 300 asynchronously across CPU cores — reducing end-to-end runtime from ~35 minutes to under 3 minutes.

---

## Results

| Metric | Value |
|--------|-------|
| Customers processed per run | ~8,300+ |
| Recommendations generated | ~46,000+ |
| Segments covered | B, C, D, E (4 of 5) |
| End-to-end runtime | < 3 minutes |
| Recommendation coverage | 88–96% per segment |

---

## Tech Stack

| Layer | Tools |
|-------|-------|
| Data pipeline | Python, pandas, numpy, SQLAlchemy |
| ML / similarity | scipy (cosine), custom item-item + user-item CF |
| Parallelism | Ray |
| Database | SQL Server (ODBC) |
| API layer | C# / ASP.NET, SignalR (real-time delivery) |
| Orchestration | CLI (`main.py`) with `combine`, `pattern_base`, `rule_base` run modes |

---

## Repository Structure

```
├── cvm_engine/
│   ├── cluster/
│   │   └── CVM_Segment.py          # Quantile-based customer segmentation
│   ├── recommender/
│   │   ├── package_recommender.py  # Item-item + user-item collaborative filtering
│   │   ├── PatternBasedRecom.py    # Pattern engine orchestration
│   │   ├── RuleBasedRecom.py       # Rule engine + Ray batch processing
│   │   ├── PatternWrapper.py       # Pattern recommendation wrapper
│   │   ├── PackageProcessor.py     # Package eligibility logic
│   │   ├── StandaloneProcessor.py  # Standalone product rules
│   │   ├── PickOptionProcessor.py  # Tier upgrade logic
│   │   ├── RoofTopProcessor.py     # Location eligibility
│   │   └── CampaignProcessor.py    # Promotional campaign injection
│   └── utils/
│       ├── constants.py            # Config (credentials via env vars)
│       ├── messages.py             # Logging messages
│       └── utils.py                # Shared utilities
├── sqlserver/
│   ├── fetchdata.py                # SQL Server connection + data push
│   └── query_proc.py               # Query processing
├── queries/                        # SQL templates for data extraction
├── main.py                         # Entry point (CLI)
└── requirements.txt
```

---

## Running the Engine

```bash
# Full run — both pattern and rule engines
python main.py --runtype combine

# Pattern-based only
python main.py --runtype pattern_base

# Rule-based only
python main.py --runtype rule_base

# Debug mode (500 customers only)
python main.py --runtype combine --debug

# Staging database
python main.py --runtype combine --staging
```

---

## Security Note

Database credentials are managed via environment variables and are not stored in this repository. Set the following before running:

```bash
export SQL_SERVER="your-server"
export SQL_DATABASE="your-database"
export SQL_USER="your-user"
export SQL_PASSWORD="your-password"
```

---

## Author

**Lia Arslan** — Data Scientist  
[github.com/ummuglsm0134](https://github.com/ummuglsm0134) · [linkedin.com/in/uarslan](https://linkedin.com/in/uarslan)
