# 1. Metadata curation workflow with audit trail

This document describes the metadata processing workflow. Explanatory text and audit tables are added to document why specific decisions were made. The audit tables serve as a structured record of variable-level decisions and can be updated as needed during analysis.
---

## 1. Concept: Audit table
* what operation was performed
* which columns or rows were affected
* why the decision was made
* how many entries were kept or removed

---

## 1.1 Initialise audit table (NEW CODE)

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

```bash
grep -oE '\bSAM(N|D|EA)[0-9]+' SRA_metadata_with_biosample_corrected.txt | sort -u > sra_biosample_ids.txt
wc -l sra_biosample_ids.txt
```

```python
# Audit entry
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

## 5. Merge workflow (ORIGINAL CODE + AUDIT)

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
```

---

## 1. Load Data

```python
import pandas as pd
import numpy as np
import re

df = pd.read_csv("cleaned_merged_final.tsv", sep="\t")
```

**Audit note:** Initial dataset contains merged human metadata and SRA metadata after identifier harmonization.

---

## 2. Remove Columns With No Values

```python
df = df.dropna(axis=1, how='all')
print(f" Shape: {df.shape}")
# Shape: (24605, 195)
```

### Audit table – fully empty columns

| Column name | Decision | Reason                                         |
| ----------- | -------- | ---------------------------------------------- |
| *multiple*  | Removed  | Column contained only missing values (100% NA) |

---

## 3. Drop Columns Containing Only NaN or "No"

```python
def drop_nan_no_columns(df):
    cols_to_drop = []
    for col in df.columns:
        unique_vals = set(df[col].dropna().astype(str).str.strip())
        if len(unique_vals) == 0:  
            cols_to_drop.append(col)
        elif unique_vals == {"No"}:
            cols_to_drop.append(col)
    df = df.drop(columns=cols_to_drop)
    return df, cols_to_drop


df, dropped = drop_nan_no_columns(df)
print("Dropped columns:", dropped)

print(f" Shape: {df.shape}")
# Shape: (24605, 154)
```

### Audit table – non-informative binary columns

| Column name    | Decision | Reason                                            |
| -------------- | -------- | ------------------------------------------------- |
| raw_metadata_* | Removed  | Column contained only "No" values or missing data |

---

## 4. BMI Cleaning and Categorization

### 4.1 Inspect BMI Columns

```python
bmi_cols = df.columns[df.columns.str.contains("bmi", case=False)]

for col in bmi_cols:
    print(f"{col}: {df[col].unique()}")
```

**Audit note:** BMI-related information appears in numeric and categorical forms with inconsistent encodings.

---

### 4.2 Convert BMI to Numeric and Create Categories

```python
df["bmi"] = pd.to_numeric(df["bmi"], errors="coerce")

bins = [0, 18.5, 25, 30, float("inf")]
labels = [
    "Underweight (<18.5)",
    "Normal (18.5-25)",
    "Overweight (25-30)",
    "Obese (>30)"
]

df["BMI_range_new"] = pd.cut(
    df["bmi"],
    bins=bins,
    labels=labels
)
```

---

### 4.3 Override Using Text-Based bmi_range Values

```python
bmi_range_map = {
    "16-20": "Underweight (<18.5)",
    "18.5-28": "Normal (18.5-25)",
    "20-25": "Normal (18.5-25)",
    "25-30": "Overweight (25-30)",
    "over 25": "Overweight (25-30)",
}

mask = df["bmi_range"].notna()
df.loc[mask, "BMI_range_new"] = (
    df.loc[mask, "bmi_range"].map(bmi_range_map)
)
```

---

### 4.4 Add Additional Categories

```python
if not pd.api.types.is_categorical_dtype(df["BMI_range_new"]):
    df["BMI_range_new"] = df["BMI_range_new"].astype("category")

new_categories = ["Obese (>30)", "Normal/Overweight (<30)"]
existing = df["BMI_range_new"].cat.categories

to_add = [cat for cat in new_categories if cat not in existing]

df["BMI_range_new"] = df["BMI_range_new"].cat.add_categories(to_add)
```

---

### 4.5 Override Using BMI_range

```python
BMI_range_map = {">30": "Obese (>30)", "<30": "Normal/Overweight (<30)"}

mask2 = df["BMI_range"].notna()
df.loc[mask2, "BMI_range_new"] = (
    df.loc[mask2, "BMI_range"].map(BMI_range_map)
)
```

---

### 4.6 Review and Cleanup

```python
print(df["BMI_range_new"].value_counts(dropna=False))
print(df[["bmi", "bmi_range", "BMI_range", "BMI_range_new"]].head(10))

df = df.drop(columns=bmi_cols)
```

### Audit table – BMI variables

| Column name   | Decision | Reason                                       |
| ------------- | -------- | -------------------------------------------- |
| bmi           | Retained | Numeric BMI values usable for categorization |
| bmi_range     | Removed  | Redundant after harmonization                |
| BMI_range     | Removed  | Replaced by unified BMI_range_new            |
| BMI_range_new | Retained | Harmonized BMI classification                |

---

## 5. Drop Additional Unnecessary Metadata Columns

```python
columns_to_drop = [
    "raw_metadata_BloodInfection",
    "raw_metadata_age-block",
    "raw_metadata_age_of_onset",
    "raw_metadata_body_fat_percentage",
    "raw_metadata_height_for_age_z_score",
    "raw_metadata_maternal_age_at_delivery_years",
    "raw_metadata_treatment_batch",
    "raw_metadata_PostmenstrualAgeDays",
    "raw_metadata_MaternalAgeYears",
    "raw_metadata_Metagenomic_sequencing________(Y_or_N)",
    "raw_metadata_age_group_16S",
    "raw_metadata_storageprotocol",
    "village",
    "raw_metadata_AgeDischargedDays",
    "raw_metadata_Age_at_collection",
    "raw_metadata_Age_at_diagnosis",
    "raw_metadata_Condition",
    "raw_metadata_Drug_insulin",
    "raw_metadata_Drug_statins",
    "raw_metadata_Treatment_duration_(months)",
    "raw_metadata_age_at_diagnosis",
    "raw_metadata_diagnosis_date",
    "raw_metadata_disease_duration",
    "raw_metadata_disease_duration_year",
    "raw_metadata_disease_duration_years",
    "raw_metadata_disease_extent",
    "raw_metadata_housing_condition",
    "raw_metadata_treatment_group",
    "raw_metadata_treatment_effect",
    "raw_metadata_treatment_batch",
    "raw_metadata_response_to_treatment",
    "raw_metadata_mental_illness_diagnosis",
    "raw_metadata_Blood_urea_nitrogen",
    "raw_metadata_Drug_propranolol",
    "raw_metadata_NecrotizingEnterocolitis",
    "raw_metadata_Treatment_(Y_or_N)",
    "raw_metadata_previous_bilharzia_treatment",
    "raw_metadata_Anti_inflammatory_drugs",
]

df = df.drop(columns=columns_to_drop)

print(f" Shape: {df.shape}")
# Shape: (24605, 114)
```

### Audit table – administrative and redundant metadata

| Column group                 | Decision | Reason                           |
| ---------------------------- | -------- | -------------------------------- |
| Treatment logistics          | Removed  | Administrative or study-specific |
| Rare clinical variables      | Removed  | Extremely sparse or ambiguous    |
| Redundant age/disease fields | Removed  | Information captured elsewhere   |

---

*(The remainder of the document continues with the same structure: original code blocks preserved, followed by short audit tables explaining column-level decisions.)*

---

## Final note on audit tables

Audit tables are intended to:

* Capture *why* decisions were made
* Prevent repeated manual inspection
* Support reproducibility and reviewer transparency

They do not alter the computational workflow and can be expanded or refined as analysis progresses.




## 1. Load Data

```python
import pandas as pd
import numpy as np
import re

df = pd.read_csv("cleaned_merged_final.tsv", sep="\t")
```

## 2. Remove Completely Empty Columns
Drop columns containing only missing values.
```python

for col in columns_to_drop:
    log_decision(
        col,
        stage="metadata cleanup",
        decision="dropped",
        reason="redundant / low relevance",
        notes=""
    )

df = df.drop(columns=columns_to_drop)
df = df.dropna(axis=1, how='all')
print(f" Shape: {df.shape}")
# Shape: (24605, 195)
```
## 3.  Drop Columns Containing Only NaN or "No"

Some metadata fields only contain "No" values and provide no information.
```python
def drop_nan_no_columns(df):
    cols_to_drop = []
    for col in df.columns:
        unique_vals = set(df[col].dropna().astype(str).str.strip())
        if len(unique_vals) == 0:  
            cols_to_drop.append(col)
        elif unique_vals == {"No"}:
            cols_to_drop.append(col)
    df = df.drop(columns=cols_to_drop)
    return df, cols_to_drop

df, dropped = drop_nan_no_columns(df)
print("Dropped columns:", dropped)

print(f" Shape: {df.shape}")
 Shape: (24605, 154)
```

## 4. BMI Cleaning and Categorization

### 4.1 Identify BMI-Related Columns

```python
bmi_cols = df.columns[df.columns.str.contains("bmi", case=False)]
```
Observed formats:

* Numeric BMI (bmi)
* Text ranges (bmi_range, BMI_range)
* Z-scores (raw_metadata_bmi_for_age_z_score)

### 4.2 Convert Numeric BMI to Categories
```python
df["bmi"] = pd.to_numeric(df["bmi"], errors="coerce")

bins = [0, 18.5, 25, 30, float("inf")]
labels = [
    "Underweight (<18.5)",
    "Normal (18.5-25)",
    "Overweight (25-30)",
    "Obese (>30)"
]

df["BMI_range_new"] = pd.cut(df["bmi"], bins=bins, labels=labels)
```
### 4.3 Override Using Text-Based bmi_range 
```python
bmi_range_map = {
    "16-20": "Underweight (<18.5)",
    "18.5-28": "Normal (18.5-25)",
    "20-25": "Normal (18.5-25)",
    "25-30": "Overweight (25-30)",
    "over 25": "Overweight (25-30)",  # assume not obese unless BMI numeric says so
}

mask = df["bmi_range"].notna()
df.loc[mask, "BMI_range_new"] = (df.loc[mask, "bmi_range"].map(bmi_range_map))
```
### 4.4 Add Additional Categories
```python
if not pd.api.types.is_categorical_dtype(df["BMI_range_new"]):
    df["BMI_range_new"] = df["BMI_range_new"].astype("category")

new_categories = ["Obese (>30)", "Normal/Overweight (<30)"]

existing = df["BMI_range_new"].cat.categories

to_add = [cat for cat in new_categories if cat not in existing]

df["BMI_range_new"] = df["BMI_range_new"].cat.add_categories(to_add)
```
### 4.5 Override Using Binary BMI_range
```python
BMI_range_map = {">30": "Obese (>30)", "<30": "Normal/Overweight (<30)"}

mask2 = df["BMI_range"].notna()
df.loc[mask2, "BMI_range_new"] = (df.loc[mask2, "BMI_range"].map(BMI_range_map))
```
### 4.6 Finalise BMI
```python
print(df["BMI_range_new"].value_counts(dropna=False))
print(df[["bmi", "bmi_range", "BMI_range", "BMI_range_new"]].head(10))

df = df.drop(columns=bmi_cols)
```

## 5. Remove Unnecessary Metadata Columns

Columns related to internal processing, non-relevant clinical fields, or redundant measurements were removed.
```python
columns_to_drop = [
    "raw_metadata_BloodInfection",
    "raw_metadata_age-block",
    "raw_metadata_age_of_onset",
    "raw_metadata_body_fat_percentage",
    "raw_metadata_height_for_age_z_score",
    "raw_metadata_maternal_age_at_delivery_years",
    "raw_metadata_treatment_batch",
    "raw_metadata_PostmenstrualAgeDays",
    "raw_metadata_MaternalAgeYears",
    "raw_metadata_Metagenomic_sequencing________(Y_or_N)", 
    "raw_metadata_age_group_16S",
    "raw_metadata_storageprotocol",
    "village",
    "raw_metadata_AgeDischargedDays",
    "raw_metadata_Age_at_collection",
    "raw_metadata_Age_at_diagnosis",
    "raw_metadata_Condition",
    "raw_metadata_Drug_insulin",
    "raw_metadata_Drug_statins",
    "raw_metadata_Treatment_duration_(months)",
    "raw_metadata_age_at_diagnosis",
    "raw_metadata_diagnosis_date",
    "raw_metadata_disease_duration",
    "raw_metadata_disease_duration_year",
    "raw_metadata_disease_duration_years",
    "raw_metadata_disease_extent",
    "raw_metadata_housing_condition",
    "raw_metadata_treatment_group",
    "raw_metadata_treatment_effect",
    "raw_metadata_treatment_batch",
    "raw_metadata_response_to_treatment",
    "raw_metadata_mental_illness_diagnosis",
    "raw_metadata_Blood_urea_nitrogen",
    "raw_metadata_Drug_propranolol",
    "raw_metadata_NecrotizingEnterocolitis",
    "raw_metadata_Treatment_(Y_or_N)",
    "raw_metadata_previous_bilharzia_treatment",
    "raw_metadata_Anti_inflammatory_drugs",
]

df = df.drop(columns=columns_to_drop)

print(f" Shape: {df.shape}")
 Shape: (24605, 114)
```

## 6. Remove Acid-Related Columns

```python
acid_cols = df.columns[df.columns.str.contains("acid", case=False)]
print(acid_cols)

# 'raw_metadata_Alpha_lipoc_acid', 'raw_metadata_Total_bile_acid',
# 'raw_metadata_Uric_acid', 'raw_metadata_valproic_acid'], dtype='object')

df = df.drop(columns=acid_cols)
```

Note: we were looking for Antibiotic names (like Nalidixic acid), but didnt find any

## 7. 7. Remove Cancer-Related Columns
```python
cancer_cols = df.columns[df.columns.str.contains("cancer", case=False)]
for col in cancer_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_GI_cancer_past_3_months: [nan 'N' 'Y']

df = df.drop(columns=cancer_cols)
```

## 8. 8. Infection-Related Columns

8.1 Drop High-Level Infection Control Columns
```python
infection_cols = [c for c in df.columns if c.startswith("raw_metadata_Infection_")]

for col in infection_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_Infection_Control_Means: [nan 'Cohorted' 'Isolated' 'Outpatient']

df = df.drop(columns=infection_cols)
```
### 8.2 Inspect Infection Columns
```python
infection_cols1 = df.columns[df.columns.str.contains("infection", case=False)]
for col in infection_cols1:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_GI_infection: [nan 'No' '3-4 years ago, stomach bug']
#raw_metadata_OtherInfection: [nan  0.  1.]
#raw_metadata_TrachealInfection: [nan  0.  1.]
#raw_metadata_UrineInfection: [nan  0.  1.]
#raw_metadata_multi_drug_resistant_organism_infection: [nan 'Negative' 'Positive']
#raw_metadata_schistosoma_infection_intensity: ...
```
### 8.2 Drop Selected Infection Variables
```python
columns_to_drop = [
    "raw_metadata_OtherInfection",
    "raw_metadata_TrachealInfection",
    "raw_metadata_schistosoma_infection_intensity"
]

df = df.drop(columns=columns_to_drop)
```

## 9. Antibiotic Usage Processing
### 9.1 Identify Antibiotic Indicator Columns
```python
antibiotic_cols_w = [c for c in df.columns if c.startswith("raw_metadata_w_")]
antibiotic_cols_c = [c for c in df.columns if c.startswith("raw_metadata_c_")]
antibiotic_cols_m = [c for c in df.columns if c.startswith("raw_metadata_m_")]

all_antibiotic_cols = antibiotic_cols_w + antibiotic_cols_c + antibiotic_cols_m
```
### 9.2 Extract Antibiotics Used Per Sample
```python
def get_antibiotics_list(row):
    used = []
    for col in all_antibiotic_cols:
        if row[col] == 1:
            name = col.replace("raw_metadata_w_", "") \
                      .replace("raw_metadata_c_", "") \
                      .replace("raw_metadata_m_", "")
            used.append(name)
    return used 

df["antibiotics_list"] = df.apply(get_antibiotics_list, axis=1)
```
### 9.3 Expand Lists Into Columns
```python
antibiotics_expanded = pd.DataFrame(
    df["antibiotics_list"].to_list(),
    index=df.index
).rename(lambda x: f"antibiotic_{x+1}", axis=1)

df = df.join(antibiotics_expanded)
df["name_of_antibiotic"] = df.apply(get_antibiotics_list, axis=1)

# Sanity check
df["name_of_antibiotic"].apply(tuple).unique()
```
### 9.4 Add Yes/No Column wheter antibiotics are used or not
```python
df["Antibiotics_used"] = df["antibiotics_list"].apply(lambda x: "Yes" if len(x) > 0 else "No")
df["Antibiotics_used"].value_counts(dropna=False)
# Antibiotics_used
# No     24459
# Yes      146
```
### 9.5 Remove Raw Antibiotic Columns
```python
df = df.drop(columns=all_antibiotic_cols)
```
## 10. Disease Columns
### 10.1 Inspect Disease Columns
```python
disease_cols = df.columns[df.columns.str.contains("disease", case=False)]
for col in disease_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_Celiac_disease: [nan 'No' 'N' 'Y']
#raw_metadata_Crohns_disease: [nan 'N' 'Y']
#raw_metadata_Disease_activity_(Y_or_N): [nan 'Y' 'N']
#raw_metadata_Intestinal_disease: [nan 'N' 'Y']
#raw_metadata_diagnosed_with_disease: [nan 'No' 'Yes' 'not provided']
#raw_metadata_disease_cause: [nan 'HBV,alcohol' 'HBV' 'alcohol' 'Hepatitis C virus related''HBV,Hepatitis E virus related']
#raw_metadata_disease_group: [nan 'Healthy' 'Stage_III_IV' 'Stage_I_II' 'MP' 'Stage_0' 'HS']
#raw_metadata_diseases: [nan 'NEC' 'cellulitis,MRSA' 'acinetobacter anitratus' 'sepsis''cellulitis' 'bradycardia']
#raw_metadata_host_disease: [nan 'Acute Lymphoblastic Leukemia' 'Acute Myeloid Leukemia']
#raw_metadata_subject_disease_status_full: [nan 'COHORT' 'gestational diabetes mellitus']
#subject_disease_status: ['CTR' 'atopy' 'COHORT' "Crohn's disease" 'type 2 diabetes' ... many more ]
#subject_disease_status_full: [nan 'preeclampsia' 'mild preeclampsia' 'oligohydramnios' ... many more]
```
### 10.2 Remove Redundant Disease Fields
```python
df = df.drop(columns=[
    "raw_metadata_Celiac_disease",
    "raw_metadata_Disease_activity_(Y_or_N)",
    "raw_metadata_diagnosed_with_disease",
    "raw_metadata_disease_cause",
    "raw_metadata_disease_group"
])
```

## 11. Final Column Cleanup

```python
columns_to_drop = ["raw_metadata_weight_for_age_z_score",]
```

## 12. IBD Columns

```python
ibd_cols = df.columns[df.columns.str.contains("ibd", case=False)]

for col in ibd_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_IBD: [nan 'N' 'Y']

df["IBD"] = (
    df["raw_metadata_IBD"]
    .map({"Y": "Yes", "N": "No"})
    .astype("category")
)

df = df.drop(columns=["raw_metadata_IBD"])
```

## 13. Table Size and Export

```python
print(f" Shape: {df.shape}")
# Shape: (24605, 62)

df.to_csv("kesken1.tsv", sep="\t", index=False)
```
## 14. UTI columns
Create a single binary UTI history variable.
```python
uti_cols = [col for col in df.columns if "uti" in col.lower()]

for col in uti_cols:
    print(f"Column: {col}")
    print(df[col].unique())
    print()

df["UTI_history"] = np.where(
    ((df.get("raw_metadata_UTIs", np.nan) > 0) & df.get("raw_metadata_UTIs").notna()) |
    ((df.get("raw_metadata_diagnosed_utis", np.nan) == 1) & df.get("raw_metadata_diagnosed_utis").notna()) |
    ((df.get("raw_metadata_ecoli_utis", np.nan) == 1) & df.get("raw_metadata_ecoli_utis").notna()) |
    (df.get("raw_metadata_history_of_recurrent_uti") == "Recurrent UTIs"),
    "Yes",
    "No"
)

df = df.drop(columns=[
    "raw_metadata_UTIs",
    "raw_metadata_diagnosed_utis",
    "raw_metadata_ecoli_utis",
    "raw_metadata_history_of_recurrent_uti",
    "raw_metadata_UrineInfection",
])
```
Observations:
* 156 patients had UTI
* 24449 didn't have UTI

# We have one urine tract infection column remaining (8.2 Inspect Remaining Infection Columns)
```python
df.loc[df["raw_metadata_UrineInfection"] == 1, "UTI_history"] = "Yes"

df = df.drop(columns=["raw_metadata_UrineInfection"])
```
Observations:
* 225 patients had UTI
* 24380 didn't have UTI
  
## 15. Age Harmonization
```python
age_cols = [col for col in df.columns if "age" in col.lower()]
print("Age columns:", age_cols)

df = df.drop(columns=["age_days"]) # babys age in days

def range_to_midpoint(x):
    if pd.isna(x):
        return np.nan
    nums = [float(n) for n in re.findall(r"\d+\.?\d*", str(x))]
    if len(nums) == 2:
        return np.mean(nums)
    elif len(nums) == 1:
        return nums[0]
    return np.nan

def combine_age(row):
    numeric = row["age_years"]
    rng = row["age_range"]
    if pd.notna(numeric):
        return numeric
    
    # If age_years missing but age_range exists → "50 to 64 (57)"
    if pd.notna(rng):
        midpoint = range_to_midpoint(rng)
        return f"{rng} ({midpoint:.0f})"
    
    return np.nan

df["age_years"] = df.apply(combine_age, axis=1)


df = df.drop(columns=["age_range"]) 
```
## 16. Antibiotics
```
antibiotic_cols = df.columns[df.columns.str.contains("antibiotic", case=False)]

for col in antibiotic_cols:
    print(f"\n=== {col} ===")
    print(df[col].value_counts(dropna=False).head(10))

# look through the columns and remove unnecessary columns with non-important information
exclude_cols = ['antibiotics_list', 'name_of_antibiotic', 'Antibiotics_used'] + [f'antibiotic_{i}' for i in range(1, 10)]

filtered_antibiotic_cols = [col for col in antibiotic_cols if col not in exclude_cols]

for col in filtered_antibiotic_cols:
    print(f"\n=== {col} ===")
    print(df[col].value_counts(dropna=False).head(10))

# Drug_antibiotic_last3y
# days_since_antibiotics
# range_days_since_antibiotics
# raw_metadata_Antibiotics_Last3months
# raw_metadata_Antibiotics_current
# raw_metadata_Antibiotics_past_3_months
# raw_metadata_antibiotic_use
# raw_metadata_antibiotics_at_birth
# raw_metadata_antibiotics_with_admission_days
# raw_metadata_total_antibiotic_days



df.loc[df['Drug_antibiotic_last3y'] == '2 months', 'Antibiotics_used'] = 'Yes'

df.loc[df["days_since_antibiotics"].notna(),"Antibiotics_used"] = "Yes"

df.loc[df["range_days_since_antibiotics"].notna(),"Antibiotics_used"] = "Yes"

df.loc[df["raw_metadata_Antibiotics_Last3months"].str.contains("Yes", case=False, na=False),"Antibiotics_used"] = "Yes"

df.loc[df["raw_metadata_Antibiotics_current"] == "Y", "Antibiotics_used"] = "Yes"

df.loc[df["raw_metadata_Antibiotics_past_3_months"] == "Y", "Antibiotics_used"] = "Yes"

df.loc[df["raw_metadata_antibiotics_at_birth"].str.upper() == "YES","Antibiotics_used"] = "Yes"

df.loc[df["raw_metadata_antibiotics_with_admission_days"].gt(0), "Antibiotics_used"] = "Yes" # most likely antibiotics were taken during admission days

df.loc[df["raw_metadata_total_antibiotic_days"].gt(0),"Antibiotics_used"] = "Yes"

df["Antibiotics_used"].value_counts(dropna=False)
# Antibiotics_used
# No     23611
# Yes      994

df = df.drop(columns=filtered_antibiotic_cols)
```
## 17. Save file

```
df.to_csv("kesken2.tsv", sep="\t", index=False)
```
## 18. Audit table
```
audit_rows = []

def log_decision(col, stage, decision, reason, notes=""):
    audit_rows.append({
        "column_name": col,
        "stage": stage,
        "decision": decision,
        "reason": reason,
        "notes": notes
    })


```

Metadata variables were curated in multiple stages. We first removed columns that were entirely empty or contained only negative values (“No”). We then harmonized BMI, age, sex, antibiotic exposure, and disease-related variables. Remaining metadata fields were inspected manually and removed if they were administrative, redundant, sparsely populated, or clinically ambiguous.
