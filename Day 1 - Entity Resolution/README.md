# Entity Resolution Pipeline for Fraud & Risk Analytics

## Project Overview
In banking and credit risk, **Synthetic Identity Fraud** is a major threat. Fraudsters exploit "data fragmentation" by creating multiple personas with slight variations in PII (Personally Identifiable Information) to bypass credit limits and AML checks.

This pipeline implements **Record Linkage** and **Fuzzy Logic** to collapse fragmented customer data into a single "Golden Record." This allows risk models to detect coordinated credit "bust-out" attempts that would otherwise go unnoticed.

## Features
* **Canonicalization:** Automated string normalization for addresses and names.
* **Blocking Logic:** High-performance indexing using "collisions" (shared phone numbers) to reduce computational complexity.
* **Multi-Metric Fuzzy Matching:** * **Jaro-Winkler** for Name similarity (prefix-biased).
    * **Levenshtein** for Address distance (edit-based).
* **Identity Resolution:** Multi-factor scoring to link fragmented accounts.

## The Step-by-Step Logic

### 1. Data Canonicalization (The Standardizer)

Raw data is messy. This step removes noise like "St." vs "Street" and converts everything to lowercase, ensuring the "base" data is comparable.
* *Example:* `123 Maple St.` vs `123 MAPLE STREET` ➔ `123 maple st`

### 2. Blocking (Efficiency Engine)
Instead of comparing 1 million customers to 1 million others ($10^{12}$ comparisons), we group records by a shared attribute, like a phone number. We only run expensive fuzzy math on records within that "block."

### 3. Fuzzy Comparison (The Detective)

We use mathematical algorithms to determine similarity:
* **Jaro-Winkler:** Perfect for names where "Jon" and "Johnathan" share the same root.
* **Levenshtein:** Measures the number of edits between addresses.

### 4. Resolution & Golden Record

If a pair of records meets the threshold (Score ≥ 2), they are flagged as a **Synthetic Identity Cluster**. Your model now sees the "True Identity" rather than four separate applicants.

## Technical Implementation (Python)

```python
import pandas as pd
import recordlinkage
from recordlinkage.preprocessing import clean

# 1. Load Data
df = pd.DataFrame({
    'name': ['Johnathan Smith', 'Jon Smith', 'J. Smith', 'Alice Brown'],
    'address': ['123 Maple St', '123 Maple Street', '123 Maple St. Apt 2', '456 Oak Rd'],
    'phone': ['5550101', '5550101', '5550101', '5559999']
})

# 2. Pre-processing
df['c_name'] = clean(df['name'])
df['c_address'] = clean(df['address'])

# 3. Linkage Logic
indexer = recordlinkage.Index()
indexer.block('phone')
candidate_links = indexer.index(df)

compare = recordlinkage.Compare()
compare.string('c_name', 'c_name', method='jarowinkler', threshold=0.85, label='name')
compare.string('c_address', 'c_address', method='levenshtein', threshold=0.70, label='address')

# 4. Results
features = compare.compute(candidate_links, df)
matches = features[features.sum(axis=1) >= 2]
print(matches)