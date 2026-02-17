# Entity Resolution and Outlier Pipeline for Fraud Analytics

## Project Overview
In banking, Synthetic Identity Fraud is a critical threat. Fraudsters exploit data fragmentation by creating multiple personas with slight variations in PII. Simultaneously, extreme transaction volumes (outliers) can distort risk scoring.

This pipeline implements Record Linkage to collapse fragmented identities and Winsorization to handle extreme financial outliers. This ensures risk models see a Golden Record with statistically stable features.

---

## Features
* Canonicalization: Standardizing messy strings (e.g., Street vs St).
* Winsorization: Capping extreme outliers (e.g., a $1M transaction) at the 95th percentile to prevent model distortion.
* Blocking Logic: Scaling performance by only comparing records sharing Collision Points (like phone numbers).
* Fuzzy Comparison:
    * Jaro-Winkler: Optimized for Name similarity.
    * Levenshtein: Optimized for Address edit distance.

---
## The Logic Breakdown

### 1. Winsorization (Outlier Handling)
Standard outlier removal deletes data. Winsorization caps it. If a fraudster moves $1M, the value is capped at the 99th percentile. This preserves the High Spender signal without letting the extreme value break the model's coefficients.

### 2. Blocking (The Efficiency Engine)
Instead of comparing every customer to every other customer, the system groups them by a shared attribute (e.g., Phone). The pipeline only runs expensive fuzzy math within these Blocks to save computational resources.

### 3. Fuzzy Comparison (The Detective)
The pipeline uses Jaro-Winkler for names because it prioritizes matching the start of strings (Jon vs. Johnathan) and Levenshtein for addresses to catch character-level typos.

### 4. Resolution and Golden Record
When scores collide across multiple attributes, the records are linked. The model no longer sees multiple separate people; it sees one identity cluster with a potential risk flag.

## Technical Implementation (Python)

```python
import pandas as pd
import recordlinkage
from recordlinkage.preprocessing import clean
from scipy.stats import mstats

# 1. Load Data with Outliers
df = pd.DataFrame({
    'name': ['Johnathan Smith', 'Jon Smith', 'J. Smith', 'Alice Brown', 'John Smith'],
    'address': ['123 Maple St', '123 Maple Street', '123 Maple St. Apt 2', '456 Oak Rd', '123 Maple Rd'],
    'phone': ['5550101', '5550101', '5550101', '5559999', '5550101'],
    'txn_amt': [1200, 1500, 1100, 500, 1000000] # Extreme Outlier
})

# 2. Winsorization: Cap outliers at the 90th percentile
df['txn_cleaned'] = mstats.winsorize(df['txn_amt'], limits=[0, 0.10])

# 3. Canonicalization
df['c_name'] = clean(df['name'])
df['c_address'] = clean(df['address'])

# 4. Linkage Logic (Blocking by Phone)
indexer = recordlinkage.Index()
indexer.block('phone')
candidate_links = indexer.index(df)

compare = recordlinkage.Compare()
compare.string('c_name', 'c_name', method='jarowinkler', threshold=0.85, label='name')
compare.string('c_address', 'c_address', method='levenshtein', threshold=0.70, label='address')

# 5. Identify Identity Clusters
features = compare.compute(candidate_links, df)
matches = features[features.sum(axis=1) >= 2]
print("Fraud Clusters Found:\n", matches)