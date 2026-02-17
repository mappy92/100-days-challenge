# 🏦 Entity Resolution Pipeline for Fraud & Risk Analytics

## 📌 Project Overview
In banking and credit risk, **Synthetic Identity Fraud** is a major threat. Fraudsters exploit "data fragmentation" by creating multiple personas with slight variations in PII (Personally Identifiable Information) to bypass credit limits and AML checks.

This pipeline implements **Record Linkage** and **Fuzzy Logic** to collapse fragmented customer data into a single "Golden Record." This allows risk models to detect coordinated credit "bust-out" attempts that would otherwise go unnoticed.

---

## 🛠 Features
* **Canonicalization:** Automated string normalization for addresses and names.
* **Blocking Logic:** High-performance indexing using "collisions" (shared phone numbers) to reduce computational complexity.
* **Multi-Metric Fuzzy Matching:** * **Jaro-Winkler** for Name similarity (prefix-biased).
    * **Levenshtein** for Address distance (edit-based).
* **Identity Resolution:** Multi-factor scoring to link fragmented accounts.

---

## 🚀 The Step-by-Step Logic

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

If a pair of records meets the threshold (Score ≥ 2), they