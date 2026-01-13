# Metadata curation with executable audit table

 The audit table can be regenerated at any time and exported as a TSV/CSV.

---

## 0. Concept: Audit table

* what operation was performed
* which columns or rows were affected
* why the decision was made
* how many entries were kept or removed
---

## 1. Initialise audit table

```python
import pandas as pd

# Initialise audit table
audit_log = []

def log_audit(step, action, target, reason, before=None, after=None):
    audit_log.append({
        "step": step,
        "action": action,
        "target": target,
        "reason": reason,
        "n_before": before,
        "n_after": after
    })
```

---

## 2. Download and inspect SRA metadata with BioSample IDs

```bash
head -n 1 SRA_metadata_with_biosample_corrected.txt | tr ',' '\n' | nl
```
Columns present:

1. acc
2. biosample
3. geo_loc_name_country_calc
4. geo_loc_name_country_continent_calc
5. platform
6. instrument
7. bioproject
8. avgspotlen
9. mbases
10. collection_date_sam

```bash
grep -oE '\bSAM(N|D|EA)[0-9]+' SRA_metadata_with_biosample_corrected.txt | sort -u > sra_biosample_ids.txt
wc -l sra_biosample_ids.txt
```

```python
log_audit(
    step="SRA metadata inspection",
    action="extract unique BioSample IDs",
    target="biosample",
    reason="Identify sample identifiers for later matching",
    after=54178
)
```

---

## 3. Inspect human_all_wide metadata

```bash
gzcat human_all_wide_2025-10-19.tsv.gz | wc -l
gzcat human_all_wide_2025-10-19.tsv.gz | head -n 1 | awk -F'\t' '{print NF}'
```

```bash
gzcat human_all_wide_2025-10-19.tsv.gz | grep -oE '\bSAM(N|D|EA)[0-9]+' | sort -u > biosample_ids.txt
```

```python
log_audit(
    step="Human metadata inspection",
    action="extract unique BioSample IDs",
    target="spire_sample_name",
    reason="Determine overlap with SRA metadata",
    after=81436
)
```

---

## 4. Identify overlapping BioSample IDs

```bash
sort biosample_ids.txt > biosample_ids_sorted.txt
sort sra_biosample_ids.txt > sra_biosample_ids_sorted.txt
comm -12 biosample_ids_sorted.txt sra_biosample_ids_sorted.txt > matched_biosamples.txt
wc -l matched_biosamples.txt
```

```python
log_audit(
    step="Sample matching",
    action="intersection",
    target="BioSample IDs",
    reason="Restrict analysis to samples present in both datasets",
    after=20339
)
```

---

## 5. Merge workflow

* Subset both datasets using shared BioSample IDs
* Merge on BioSample ID
* Retain clinically relevant metadata fields
* Remove irrelevant or sensitive columns
* Handle large files via chunked reading

```python
import re

human_file = "human_all_wide_2025-10-19.tsv.gz"
sra_file = "SRA_metadata_with_biosample_corrected.txt"
matched_file = "matched_biosamples.txt"
final_output = "cleaned_merged_final.tsv"

id_col_human = "spire_sample_name"
id_col_sra = "biosample"

chunksize = 20000
```
### 4.3 Define Keywords for Column Filtering

These keywords retain columns related to:
* Demographics (age, sex, BMI)
* Diseases and conditions
* Antibiotic and drug exposure

```python

keywords = [
    "bmi", "body mass", "antibiotic", "antimicrobial", "drug",
    "sex", "gender", "age", "disease", "diagnosis", "condition",
    "infection", "immune", "inflammation", "therapy", "treatment",
    "cycline", "cillin", "phenicol", "bactam", "beta", "lactamase", "inhibitor",
    "cef", "loracarbef", "flomoxef", "latamoxef", "onam", "penem", "cilastatin",
    "cephalexin", "dazole", "mycin", "prim", "sulfa", "tylosin", "tilmicosin",
    "tylvacosin", "tildipirosin", "macrolide", "quinupristin", "dalfopristin",
    "xacin", "acid", "flumequine", "olaquindox", "sulfonamide", "tetracycline",
    "nitrofuran", "amphenicol", "polymyxin", "quinolone",
    "aminoglycoside", "teicoplanin", "vancin", "colistin", "polymyxin b",
    "nitro", "fura", "nifru", "micin", "tiamulin", "valnemulin",
    "xibornol", "clofoctol", "methenamine", "zolid", "lefamulins",
    "gepotidacin", "bacitracin", "novobiocin",
    "asthma", "cancer", "uti", "ibd", "crohn", "intestine",
    "ulcerative", "colitis"
]
keywords = [kw.lower() for kw in keywords]
```
## 4.4 Define Exclusion Keywords

Columns containing these terms are removed.

```
exclude_keywords = [
    "sexual", "private", "identifier", "confidential", "contact",
    "gestational", "beverages", "stage", "vitamin", "alcohol",
    "average", "home", "sports", "strength", "siblings", "blockage"
]
```

### 5.1 Load matched BioSamples

```python
matched_biosamples = pd.read_csv(
    matched_file, header=None, names=[id_col_sra], dtype=str
)
matched_set = set(matched_biosamples[id_col_sra])

log_audit(
    step="Load matched samples",
    action="load",
    target="matched_biosamples",
    reason="Define reference set for subsetting",
    after=len(matched_set)
)
```

---

### 5.2 Subset human metadata

```python
human_subset_file = "tmp_human_subset.tsv"
first = True

for chunk in pd.read_csv(human_file, sep="\t", dtype=str, chunksize=chunksize):
    filtered = chunk[chunk[id_col_human].isin(matched_set)]
    filtered.to_csv(
        human_subset_file,
        sep="\t",
        index=False,
        mode="w" if first else "a",
        header=first
    )
    first = False

human = pd.read_csv(human_subset_file, sep="\t", dtype=str)
```

```python
log_audit(
    step="Human metadata filtering",
    action="row filter",
    target="human_all_wide",
    reason="Keep only samples present in SRA",
    after=human.shape[0]
)
```

---

### 5.3 Subset SRA metadata

```python
sra = pd.read_csv(sra_file, sep=",", dtype=str)
before = sra.shape[0]
sra = sra[sra[id_col_sra].isin(matched_set)]
```

```python
log_audit(
    step="SRA metadata filtering",
    action="row filter",
    target="SRA_metadata",
    reason="Keep only samples present in human metadata",
    before=before,
    after=sra.shape[0]
)
```

---

### 5.4 Merge datasets

```python
merged = pd.merge(
    human, sra,
    left_on=id_col_human,
    right_on=id_col_sra,
    how="inner",
    validate="many_to_many"
)
```

```python
log_audit(
    step="Merge",
    action="inner join",
    target="human + SRA",
    reason="Combine biological and sequencing metadata",
    after=merged.shape[0]
)
```

---

## 6. Column filtering with documented rationale

```python
keywords = [kw.lower() for kw in keywords]

def keep_keyword(c):
    return any(kw in c.lower() for kw in keywords)

def drop_excluded(c):
    return any(bad in c.lower() for bad in exclude_keywords)

always_keep = set(sra.columns.tolist() + [id_col_human])
keyword_keep = {c for c in merged.columns if keep_keyword(c)}

cols_to_keep = sorted(
    c for c in (always_keep | keyword_keep)
    if c in merged.columns and not drop_excluded(c)
)

merged_clean = merged[cols_to_keep]
```

```python
log_audit(
    step="Column filtering",
    action="column selection",
    target="merged dataset",
    reason="Retain demographic, clinical, and treatment-related metadata",
    before=merged.shape[1],
    after=len(cols_to_keep)
)
```

---

## 7. Save outputs

```python
merged_clean.to_csv(final_output, sep="\t", index=False)
```

```python
# Convert audit log to table and save
audit_table = pd.DataFrame(audit_log)
audit_table.to_csv("metadata_audit_table.tsv", sep="\t", index=False)
```

---

## Outputs produced

* `cleaned_merged_final.tsv` — curated metadata for analysis
* `metadata_audit_table.tsv` — **fully reproducible audit trail of all decisions**

This file can be cited directly in Methods or provided as Supplementary Material.
