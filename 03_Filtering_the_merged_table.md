
## 1. Load Data

```
import pandas as pd
import numpy as np
import re

df = pd.read_csv("cleaned_merged_final.tsv", sep="\t")
```

## 2. Remove Columns With No Values

```
df = df.dropna(axis=1, how='all')
print(f" Shape: {df.shape}")
# Shape: (24605, 195)
```
## 3. Drop Columns Containing Only NaN or "No"

```
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

```

## 4. BMI Cleaning and Categorization

### 4.1 Inspect BMI Columns

```
bmi_cols = df.columns[df.columns.str.contains("bmi", case=False)]

for col in bmi_cols:
    print(f"{col}: {df[col].unique()}")

#BMI_range: [nan '>30' '<30']
#bmi: [        nan 23.1        20.4        ... 21.33683793 24.27411265    26.21415974]
#bmi_range: [nan '20-25' '16-20' '18.5-28' 'over 25' '25-30']
#raw_metadata_bmi_for_age_z_score: [  nan  0.94  .... ]
```
### 4.2 Convert BMI to Numeric and Create Categories

```
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
### 4.3 Override Using Text-Based bmi_range Values
```
# Override using `bmi_range` text values

bmi_range_map = {
    "16-20": "Underweight (<18.5)",
    "18.5-28": "Normal (18.5-25)",
    "20-25": "Normal (18.5-25)",
    "25-30": "Overweight (25-30)",
    "over 25": "Overweight (25-30)",  # assume not obese unless BMI numeric says so
}

mask = df["bmi_range"].notna()
df.loc[mask, "BMI_range_new"] = (
    df.loc[mask, "bmi_range"].map(bmi_range_map)
)
```
### 4.4 Add Additional Categories
```
if not pd.api.types.is_categorical_dtype(df["BMI_range_new"]):
    df["BMI_range_new"] = df["BMI_range_new"].astype("category")

new_categories = ["Obese (>30)", "Normal/Overweight (<30)"]
existing = df["BMI_range_new"].cat.categories

to_add = [cat for cat in new_categories if cat not in existing]

df["BMI_range_new"] = df["BMI_range_new"].cat.add_categories(to_add)
```
### 4.5 Override Using BMI_range
```
BMI_range_map = {">30": "Obese (>30)", "<30": "Normal/Overweight (<30)"}

mask2 = df["BMI_range"].notna()
df.loc[mask2, "BMI_range_new"] = (
    df.loc[mask2, "BMI_range"].map(BMI_range_map)
)
```
### 4.6 Review and Cleanup
```
print(df["BMI_range_new"].value_counts(dropna=False))
print(df[["bmi", "bmi_range", "BMI_range", "BMI_range_new"]].head(10))

df = df.drop(columns=bmi_cols)
```

## 5. Drop Additional Unnecessary Metadata Columns

```
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

```

## 6. Acid-Related Columns

```
acid_cols = df.columns[df.columns.str.contains("acid", case=False)]
print(acid_cols)

# 'raw_metadata_Alpha_lipoc_acid', 'raw_metadata_Total_bile_acid',
# 'raw_metadata_Uric_acid', 'raw_metadata_valproic_acid'], dtype='object')

df = df.drop(columns=acid_cols)
```

## 7. Cancer-Related Columns

```
cancer_cols = df.columns[df.columns.str.contains("cancer", case=False)]
for col in cancer_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_GI_cancer_past_3_months: [nan 'N' 'Y']

df = df.drop(columns=cancer_cols)
```

## 8. Infection-Related Columns

### 8.1 Drop High-Level Infection Columns
```
infection_cols = [c for c in df.columns if c.startswith("raw_metadata_Infection_")]

for col in infection_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_Infection_Control_Means: [nan 'Cohorted' 'Isolated' 'Outpatient']

df = df.drop(columns=infection_cols)
```
### 8.2 Inspect Remaining Infection Columns
```
infection_cols1 = df.columns[df.columns.str.contains("infection", case=False)]
for col in infection_cols1:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_GI_infection: [nan 'No' '3-4 years ago, stomach bug']
#raw_metadata_OtherInfection: [nan  0.  1.]
#raw_metadata_TrachealInfection: [nan  0.  1.]
#raw_metadata_UrineInfection: [nan  0.  1.]
#raw_metadata_multi_drug_resistant_organism_infection: [nan 'Negative' 'Positive']
```
### 8.3 Drop Selected Columns
```
columns_to_drop = [
    "raw_metadata_GI_infection",
    "raw_metadata_OtherInfection",
    "raw_metadata_TrachealInfection",
]

df = df.drop(columns=columns_to_drop)
```

## 9. Antibiotic Usage Processing

### 9.1 Identify Antibiotic Columns
```
antibiotic_cols_w = [c for c in df.columns if c.startswith("raw_metadata_w_")]
antibiotic_cols_c = [c for c in df.columns if c.startswith("raw_metadata_c_")]
antibiotic_cols_m = [c for c in df.columns if c.startswith("raw_metadata_m_")]

all_antibiotic_cols = antibiotic_cols_w + antibiotic_cols_c + antibiotic_cols_m
```
### 9.2 Create Antibiotic List per Sample
```
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
```
antibiotics_expanded = pd.DataFrame(
    df["antibiotics_list"].to_list(),
    index=df.index
).rename(lambda x: f"antibiotic_{x+1}", axis=1)

df = df.join(antibiotics_expanded)
df["name_of_antibiotic"] = df.apply(get_antibiotics_list, axis=1)

# Sanity check
df["name_of_antibiotic"].apply(tuple).unique()
```
### 9.4 Add Yes/No Column
```
df["Antibiotics_used"] = df["antibiotics_list"].apply(lambda x: "Yes" if len(x) > 0 else "No")
```
### 9.5 Remove Raw Antibiotic Columns
```
df = df.drop(columns=all_antibiotic_cols)
```
## 10. Disease Columns
### 10.1 Inspect Disease Columns
```
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
### 10.2 Drop Selected Disease Columns
```
columns_to_drop = [
    "raw_metadata_Celiac_disease",
    "raw_metadata_Crohns_disease",
    "raw_metadata_Disease_activity_(Y_or_N)",
    "raw_metadata_diagnosed_with_disease",
    "raw_metadata_disease_cause",
    "raw_metadata_disease_group"
]

df = df.drop(columns=columns_to_drop)
```

## 11. Remove Additional Columns

```
columns_to_drop = [
    "raw_metadata_schistosoma_infection_intensity",
    "raw_metadata_weight_for_age_z_score",
]

df = df.drop(columns=columns_to_drop)
```

## 12. IBD Columns

```
ibd_cols = df.columns[df.columns.str.contains("ibd", case=False)]

for col in ibd_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_IBD: [nan 'N' 'Y']
```

## 13. Final Table Size and Export

```
print(f" Shape: {df.shape}")
# Shape: (24605, 64)

df.to_csv("kesken1.tsv", sep="\t", index=False)
```
