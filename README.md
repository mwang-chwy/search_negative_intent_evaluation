# Negative Intent Evaluation Framework

## Overview
This framework evaluates search algorithm performance on negative intent queries (e.g., "cat food without chicken") using LLM-based judgments. It calculates metrics at both retrieval-path (L1) and end-to-end (L2) levels across multiple endpoints (lexical, kNN, l1_hybrid, l2).

**Key Features:**
- ✅ Search result collection from multiple endpoints
- ✅ Automated negative intent detection using LLM
- ✅ Query-product pair evaluation with LLM judgments
- ✅ Comprehensive metrics at search term and aggregated levels
- ✅ Support for multiple variant analysis (control, treatment, etc.)

### Step-by-Step Workflow

#### Step 1: Fetch Search Results *(Control data already queried)*
**Notebook:** `01_fetch_search_results_control.ipynb`

**For Control Variant:**
The control search results have already been collected and stored in:
```
search_results/
├── lexical_search_results.csv
├── knn_search_results.csv
├── l1_hybrid_search_results.csv
└── l2_search_results.csv
```

**What's in the data:**
- Search terms from `negative_intent_candidates(raw_data).csv`
- Top K results from each endpoint (K=36 by default)
- Product details: part_number, sku_name, brand, description, rank, score

**⚠️ Note on Score Field:**
For `l1_hybrid` and `l2` endpoints, the API returns a score of zero for all results. To provide a meaningful relevance proxy in the saved CSV files, the score is calculated as `1/(rank + 1)` based on the result's position:
- Rank 1 → score = 0.5
- Rank 2 → score = 0.333
- Rank 3 → score = 0.25
- etc.

For `knn` and `lexical` endpoints, the actual score from the API response is preserved. This is not the actual model score for l1_hybrid/l2, but rather a rank-based approximation.

⚠️ **For Control:** You can skip this step - data is already available! The control data reflects the latest production search behavior.

**For Treatment/Variant Data:**
⚠️ **Engineering Setup Required** - To collect treatment variant data:
1. Work with engineering team to deploy the variant search API
2. Once deployed, adapt `01_fetch_search_results_control.ipynb` to query the variant endpoint
3. Save results to a new directory or with variant-specific naming
4. Continue with Step 2a using the variant data

---

#### Step 2a: Negative Intent Detection
**Notebook:** `02a_negative_intent_detection.ipynb`

**What it does:**
- Uses LLM to analyze each search term
- Identifies if the query has negative intent (e.g., "without X", "exclude Y")
- Extracts the specific attribute/ingredient the user wants to exclude
- Applies manual fixes to clean up LLM responses

**Output:**
```
llm_evaluations/negative_intent/
├── negative_intent_detection.json              # Raw LLM responses (preserved)
└── negative_intent_detection_manual_fix.json   # Cleaned version (USED IN ALL DOWNSTREAM STEPS)
```

**Manual Fixes Applied:**
The notebook automatically applies a `fix_rules` dictionary to clean up LLM responses:
1. **Multi-line/verbose responses** - Fixes known multi-line outputs:
   - `"non clumping unscented cat litter\n\nthe user is..."` → `"clumping"`
   - `"unscented\ndust"` → `"dust, scent"`
2. **Consolidations** - Merges similar intents (28 rules):
   - `"waste free"` → `"waste"`
   - `"non clumping"` → `"clumping"`
   - `"no melt"` → `"melt"`
   - `"non absorbent"` → `"absorbent"`
   - `"non gmo"` → `"gmo"`
   - `"no bark"` → `"bark"`
   - `"scented"/"unscented"` → `"scent"`
   - `"cat pee"` → `"pee"`
   - `"poop eating"` → `"poop"`
   - `"stuffed"` → `"stuffing"`
   - `"soy and corn"/"corn and soy"` → `"corn, soy"`
   - `"squeaky"` → `"squeak"`
   - `"track"` → `"tracking"`
3. **Typo fixes**:
   - `"gain"` → `"grain"`
   - `"stuff"` → `"stuffing"`
   - `"aller"` → `"allergy"`
4. **Remove too-generic terms** - Set to `None`:
   - `"free"`, `"holes"`, `"hides"`, `"toot"`

⚠️ **IMPORTANT:** All downstream notebooks (02b, 03a, 03b) use `negative_intent_detection_manual_fix.json`, not the raw file.

**How to run:**
```python
# The notebook is pre-configured - just run all cells
# Review the "Manual Fixes" section output to see all applied changes
```

---

#### Step 2b: LLM Query-Product Pair Evaluation  
**Notebook:** `02b_llm_evaluation_pairs.ipynb`

**What it does:**
- For each search term with negative intent, evaluates query-product pairs
- LLM judges if products VIOLATE, are COMPLIANT, or are UNCLEAR
- Merges judgments back with search results

**Output:**
```
llm_evaluations/
├── pair_judgments/
│   ├── evaluations_checkpoint.json  # Progress tracking
│   └── final_evaluations.json       # All judgments
└── merged_csv_results/{VARIANT}/
    ├── lexical_search_results.csv
    ├── knn_search_results.csv
    ├── l1_hybrid_search_results.csv
    └── l2_search_results.csv
```

**How to run:**
```python
# Set your variant (if analyzing multiple experiments)
VARIANT = "control"  # or "treatment_v1", "treatment_v2", etc.

# Run all cells - checkpoint system allows resuming if interrupted
```

---

#### Step 3: Calculate Metrics
**Notebook:** `03a_calculate_metrics_control.ipynb` (template version available)

**What it does:**
- Calculates violation rates, compliant rates, unclear rates
- Metrics at 3 K-values: K@4, K@10, K@36
- Per-search-term level details AND aggregated metrics
- Breakdown by negative intent type for all endpoints

**Output:**
```
metrics_output/{VARIANT}/
├── {VARIANT}_aggregated_metrics.csv      # Summary metrics per endpoint
└── {VARIANT}_search_term_metrics.csv     # Detailed per-search-term metrics
```

**How to run:**
```python
# Change VARIANT if needed
VARIANT = "control"  # Update for your variant

# Run all cells
```

**Key Metrics Explained:**

| Metric | Description | Formula |
|--------|-------------|---------|
| `violation_rate@K` | % of top-K results that violate negative intent | violations / total_results |
| `compliant_rate@K` | % of top-K results that respect negative intent | compliant / total_results |
| `unclear_rate@K` | % of top-K results with unclear judgment | unclear / total_results |

**Two levels of aggregation:**
1. **Search term level**: Metrics calculated per individual search term
2. **Aggregated level**: Average metrics across all search terms

---

#### Step 3b: Negative Intent Volume Analysis (Optional)
**Notebook:** `03b_negative_intent_volume_analysis.ipynb`

**What it does:**
- Combines search-term metrics with search volume data to calculate **weighted violation rates**
- Provides volume-weighted analysis by endpoint, K-value, and negative intent type
- Shows which negative intents contribute most to violations based on actual search traffic
- Displays sample search terms for each negative intent category

**Key Features:**
- **Weighted metrics**: Violations/compliance rates weighted by search volume, giving more importance to high-traffic queries
- **Intent-level breakdown**: See which specific negative intents (e.g., "grain", "chicken", "scent") have highest violation rates
- **Sample term inspection**: View top 5 search terms by volume for each negative intent
- **Above/below average comparison**: Automatically compares each intent's violation rate to the overall average

**Output:**
```
metrics_output/{VARIANT}/
└── {VARIANT}_negative_intent_volume_analysis.csv  # Weighted metrics by intent
```

**How to run:**
```python
# Set your variant and default K
VARIANT = "control"
DEFAULT_K = 36

# Run all cells - notebook uses the search_term_metrics.csv from Step 3a
```

**Configuration:**
- `DEFAULT_K`: The K value to use for detailed breakdowns (default: 36)
- `K_VALUES`: List of K values to calculate metrics for (default: [4, 10, 36])
- Minimum volume threshold for display: `terms_with_volume > 8`

**Output columns:**
- `endpoint`: The search endpoint (knn, lexical, l1_hybrid, l2)
- `k`: The K value
- `negative_intent`: The specific intent or "all negative intent" for aggregates
- `terms_with_volume`: Number of search terms with volume data
- `total_volume`: Total search volume for this intent
- `weighted_violation_rate`: Volume-weighted violation rate
- `weighted_compliant_rate`: Volume-weighted compliance rate
- `weighted_unclear_rate`: Volume-weighted unclear rate

**Why use this?**
Standard metrics treat all search terms equally, but "dog food without chicken" (high volume) matters more than "cat treats without taurine" (low volume). This notebook provides business-relevant metrics weighted by actual user impact.

---

## 🗂️ Directory Structure

```
negative_intent/
├── � **Input Data**
│   ├── negative_intent_candidates(raw_data).csv  # Initial candidate queries
│   └── search_results/                           # Search results from endpoints (Step 1 output)
│       ├── lexical_search_results.csv
│       ├── knn_search_results.csv  
│       ├── l1_hybrid_search_results.csv
│       └── l2_search_results.csv
│
├── 📁 **LLM Evaluations** (Step 2 outputs)
│   └── llm_evaluations/
│       ├── negative_intent/                                # Step 2a: Intent detection
│       │   ├── negative_intent_detection.json              # Raw LLM responses (preserved)
│       │   └── negative_intent_detection_manual_fix.json   # Cleaned (used downstream)
│       ├── pair_judgments/                                 # Step 2b: Pair evaluation  
│       │   ├── evaluations_checkpoint.json
│       │   └── final_evaluations.json
│       └── merged_csv_results/{VARIANT}/                   # Search results + judgments
│           ├── lexical_search_results.csv
│           ├── knn_search_results.csv
│           ├── l1_hybrid_search_results.csv
│           └── l2_search_results.csv
│
├── 📈 **Metrics Output** (Step 3 outputs)
│   └── metrics_output/{VARIANT}/
│       ├── {VARIANT}_aggregated_metrics.csv                # Summary per endpoint
│       ├── {VARIANT}_search_term_metrics.csv               # Detailed per search term
│       └── {VARIANT}_negative_intent_volume_analysis.csv   # Volume-weighted by intent (Step 3b)
│
└── 📓 **Notebooks**
    ├── 01_fetch_search_results_control.ipynb     # Step 1 (already run)
    ├── 02a_negative_intent_detection.ipynb       # Step 2a
    ├── 02b_llm_evaluation_pairs.ipynb            # Step 2b  
    ├── 03a_calculate_metrics_control.ipynb       # Step 3
    └── 03b_negative_intent_volume_analysis.ipynb # Step 3b (optional)
```