
## 1. Download the data

```
import pandas as pd
import numpy as np
import re

df = pd.read_csv("cleaned_merged_final.tsv", sep="\t")
```

## 2. Remove columns with no values

```
df = df.dropna(axis=1, how='all')
print(f" Shape: {df.shape}")
#Shape: (24605, 195)
```
## 3. BMI

```
bmi_cols = df.columns[df.columns.str.contains("bmi", case=False)]
print(bmi_cols)

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

if not pd.api.types.is_categorical_dtype(df["BMI_range_new"]):
    df["BMI_range_new"] = df["BMI_range_new"].astype("category")

new_categories = ["Obese (>30)", "Normal/Overweight (<30)"]
existing = df["BMI_range_new"].cat.categories

to_add = [cat for cat in new_categories if cat not in existing]

df["BMI_range_new"] = df["BMI_range_new"].cat.add_categories(to_add)

BMI_range_map = {
    ">30": "Obese (>30)",
    "<30": "Normal/Overweight (<30)"
}

mask2 = df["BMI_range"].notna()
df.loc[mask2, "BMI_range_new"] = (
    df.loc[mask2, "BMI_range"].map(BMI_range_map)
)

# Review
print(df["BMI_range_new"].value_counts(dropna=False))
print(df[["bmi", "bmi_range", "BMI_range", "BMI_range_new"]].head(10))

df["raw_metadata_bmi_for_age_z_score"].unique()
# remove column

```

## 3. Age

# ! add this: df["raw_metadata_age_group"].unique()

```
age_cols = df.columns[df.columns.str.contains("age", case=False)]
print(age_cols)

df["age_days"].unique()
# remove column
df["age_years"].unique()
df["age_category"].unique()
df["age_range"].unique()

def unify_age_range(rng):
    if pd.isna(rng):
        return np.nan, np.nan
    rng = str(rng).lower().strip()
    
    if rng in ['baby', 'child', 'adolescent', 'adult']:
        return rng, rng
    
    match = re.findall(r'\d*\.?\d+', rng)
    if len(match) == 2:
        return float(match[0]), float(match[1])
    
    match_over = re.search(r'over (\d*\.?\d+)', rng)
    if match_over:
        return float(match_over.group(1)), np.inf
    
    match_below = re.search(r'(?:below|under) (\d*\.?\d+)', rng)
    if match_below:
        return 0, float(match_below.group(1))
    
    if len(match) == 1:
        return float(match[0]), float(match[0])
    
    return np.nan, np.nan

df[['age_start', 'age_end']] = df['age_range'].apply(lambda x: pd.Series(unify_age_range(x)))

# Step 2 — Numeric age from range

def range_to_midpoint(start, end):
    if isinstance(start, str):  # textual category
        return np.nan
    if end == np.inf:
        return start + 1
    return (start + end) / 2

df['age_numeric_from_range'] = df.apply(lambda row: range_to_midpoint(row['age_start'], row['age_end']), axis=1)

# Step 3 — Combine numeric age

df['age_numeric'] = df['age_years']

mask_days = df['age_numeric'].isna() & df['age_days'].notna()
df.loc[mask_days, 'age_numeric'] = df.loc[mask_days, 'age_days'] / 365

mask_range = df['age_numeric'].isna() & df['age_numeric_from_range'].notna()
df.loc[mask_range, 'age_numeric'] = df.loc[mask_range, 'age_numeric_from_range']


# Step 4 — Assign labeled age category

def numeric_age_label(age):
    if pd.isna(age):
        return 'Unknown age'
    if age < 1.0:
        return 'Baby (0–1y)'
    elif age < 10:
        return 'Child (1–9y)'
    elif age < 20:
        return 'Adolescent (10–19y)'
    else:
        return 'Adult (≥19y)'

df['age_category_numeric'] = df['age_numeric'].apply(numeric_age_label)

# Step 5 — Merge with existing textual categories

category_map = {
    'baby': 'Baby (0–1y)',
    'child': 'Child (1–9y)',
    'adolescent': 'Adolescent (10–19y)',
    'adult': 'Adult (≥19y)'
}

def clean_age_category_label(existing_cat, numeric_cat):
    if pd.isna(existing_cat):
        return numeric_cat
    categories = [c.strip() for c in str(existing_cat).split(',')]
    categories_mapped = [category_map.get(c, c) for c in categories]
    if numeric_cat in categories_mapped:
        return numeric_cat
    order = ['Baby (0–1y)', 'Child (1–9y)', 'Adolescent (10–19y)', 'Adult (≥19y)']
    sorted_cats = sorted(categories_mapped, key=lambda x: order.index(x))
    return sorted_cats[0]

df['age_category_new'] = df.apply(
    lambda row: clean_age_category_label(row['age_category'], row['age_category_numeric']), axis=1
)

```

## Remove columns that are no longer needed

```
columns_to_drop = [
    "bmi_range",
    "bmi",
    "BMI_range",
    "age_days",
    "age_range",
    "age_category",
    "age_years",
    "raw_metadata_Antibiotics_Last3months",
    "raw_metadata_Autoimmune_hemolytic_anemia",
    "raw_metadata_Autoimmune_hepatitis",
    "raw_metadata_BloodInfection",
    "raw_metadata_Sex_Trade_Worker",
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
    "Drug_antibiotic_last3y",
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
]

df = df.drop(columns=columns_to_drop)

#remove separately
#df = df.drop(columns=["raw_metadata_housing_condition"])

print(f" Shape: {df.shape}")
#24605, 163
```

## Save file for later use so that you can reload and modify it again

```
df.to_csv("kesken1.tsv", sep="\t", index=False)
df = pd.read_csv("kesken1.tsv", sep="\t")
```
## Antibiotics:

## Columns with name of the antibiotic

```
antibiotic_w_cols = [c for c in df.columns if c.startswith("raw_metadata_w_")]
antibiotic_c_cols = [c for c in df.columns if c.startswith("raw_metadata_c_")]
antibiotic_m_cols = [c for c in df.columns if c.startswith("raw_metadata_m_")]

all_antibiotic_cols = antibiotic_w_cols + antibiotic_c_cols + antibiotic_m_cols

# 2️⃣ Convert antibiotic names into a list for each patient
def get_antibiotics(row):
    antibiotics = []
    for col in all_antibiotic_cols:
        val = row[col]
        if pd.notnull(val) and float(val) > 0: 
            name = col.replace("raw_metadata_w_", "") \
                      .replace("raw_metadata_c_", "") \
                      .replace("raw_metadata_m_", "") \
                      .replace("_", " ")
            antibiotics.append(name)
    return antibiotics

df["antibiotics_list"] = df.apply(get_antibiotics, axis=1)

# 3️⃣ Expand into multiple columns: antibiotic_1, antibiotic_2, ...
max_antibiotics = df["antibiotics_list"].apply(len).max()

for i in range(max_antibiotics):
    df[f"antibiotic_{i+1}"] = df["antibiotics_list"].apply(
        lambda x: x[i] if len(x) > i else None
    )

# 4️⃣ Remove original antibiotic columns — optional but cleaner
df = df.drop(columns=all_antibiotic_cols)

df.to_csv("kesken3.tsv", sep="\t", index=False)

```

### Look through the columns
```
antibiotic_cols = df.columns[df.columns.str.contains("antibiotic", case=False)]

for col in antibiotic_cols:
    print(col)
#days_since_antibiotics
#range_days_since_antibiotics
#raw_metadata_Antibiotics_current
#raw_metadata_Antibiotics_past_3_months
#raw_metadata_antibiotic_use
#raw_metadata_antibiotics
#raw_metadata_antibiotics_at_birth
#raw_metadata_antibiotics_with_admission_days
#raw_metadata_total_antibiotic_days

df["days_since_antibiotics"].unique()
df["range_days_since_antibiotics"].unique()
df["raw_metadata_Antibiotics_current"].unique()
df["raw_metadata_Antibiotics_past_3_months"].unique()
df["raw_metadata_antibiotic_use"].unique()
#remove column
df["raw_metadata_antibiotics"].unique()
#remove column
df["raw_metadata_antibiotics_at_birth"].unique()
#odottaa
df["raw_metadata_antibiotics_with_admission_days"].unique()
#odottaa
df["raw_metadata_total_antibiotic_days"].unique()
```

## Create metadata
```
df["antibiotics_current"] = (
    df["raw_metadata_Antibiotics_current"]
    .replace({"Y": "Yes", "N": "No"})
)

df["antibiotics_past_3_months"] = (
    df["raw_metadata_Antibiotics_past_3_months"]
    .replace({"Y": "Yes", "N": "No"})
)

df["antibiotics_at_birth"] = (
    df["raw_metadata_antibiotics_at_birth"]
    .replace({"Y": "Yes", "N": "No"})
)

# Keep raw continuous data
df["total_antibiotic_days"] = df["raw_metadata_total_antibiotic_days"]

# Create cleaned time-based categories

def categorize_time(days):
    if pd.isna(days):
        return "Unknown"
    days = float(days)
    if days <= 7:
        return "0–7 days"
    elif days <= 14:
        return "8–14 days"
    elif days <= 30:
        return "15–30 days"
    elif days <= 60:
        return "31–60 days"
    elif days <= 90:
        return "61–90 days"
    elif days <= 180:
        return "91–180 days"
    else:
        return ">180 days"

df["antibiotic_time_final"] = df["days_since_antibiotics"].apply(categorize_time)

def determine_antibiotic_use(row):
    total_days = row.get("total_antibiotic_days")

    # Any clear evidence of antibiotics = Yes
    if row.get("antibiotics_current") == "Yes":
        return "Yes"
    if row.get("antibiotics_past_3_months") == "Yes":
        return "Yes"
    if pd.notnull(total_days) and total_days > 0:
        return "Yes"
    if row.get("antibiotics_at_birth") == "Yes":
        return "Yes"

    # Explicit No (all metadata disagree with use)
    if (
        row.get("antibiotics_current") == "No"
        and row.get("antibiotics_past_3_months") == "No"
        and row.get("antibiotics_at_birth") == "No"
    ):
        return "No"

    return "Unknown"

df["antibiotic_time_final"] = df["total_antibiotic_days"].apply(categorize_time)


def merge_antibiotic_exposure(row):
    current = row.get("antibiotics_current")
    past3m = row.get("antibiotics_past_3_months")
    time = row.get("antibiotic_time_final")
    birth = row.get("antibiotics_at_birth")

    if current == "Yes":
        return "Current antibiotics"
    if birth == "Yes":
        return "Early-life antibiotics"
    if past3m == "Yes" or time in [
        "0–7 days", "8–14 days", "15–30 days", "31–60 days", "61–90 days"
    ]:
        return "Recent antibiotics"
    if (
        current == "No"
        and past3m == "No"
        and birth == "No"
    ):
        return "No antibiotics"
    return "Unknown"

df["antibiotic_exposure_group"] = df.apply(merge_antibiotic_exposure, axis=1)

# Ensure logical ordering for plotting
df["antibiotic_exposure_group"] = pd.Categorical(
    df["antibiotic_exposure_group"],
    categories=[
        "Current antibiotics",
        "Recent antibiotics",
        "Early-life antibiotics",
        "No antibiotics",
        "Unknown"
    ],
    ordered=True
)

df.to_csv("kesken2.tsv", sep="\t", index=False)
```
## Remove other unnecessary columns

```
columns_to_drop = [
"days_since_antibiotics",
"range_days_since_antibiotics",
"raw_metadata_Antibiotics_current",
"raw_metadata_Antibiotics_past_3_months",
"raw_metadata_antibiotic_use",
"raw_metadata_antibiotics",
"raw_metadata_antibiotics_with_admission_days",
"raw_metadata_total_antibiotic_days"

]

df = df.drop(columns=columns_to_drop)
```


## Infections

```
df = pd.read_csv("kesken2.tsv", sep="\t")

infection_cols = [c for c in df.columns if c.startswith("raw_metadata_Infection_")]
print(infection_cols)
#['raw_metadata_Infection_CRE', 'raw_metadata_Infection_Control_Means', 'raw_metadata_Infection_ESBLs', 'raw_metadata_Infection_MRSA', 'raw_metadata_Infection_Other', 'raw_metadata_Infection_VRE']

df["raw_metadata_Infection_CRE"].unique()

df["raw_metadata_Infection_Control_Means"].unique()
array([nan, 'No'], dtype=object)

df["raw_metadata_Infection_ESBLs"].unique()
array([nan, 'Cohorted', 'Isolated', 'Outpatient'], dtype=object)

df["raw_metadata_Infection_MRSA"].unique()
array([nan, 'No'], dtype=object)

df["raw_metadata_Infection_Other"].unique()
array([nan, 'Cohorted', 'Isolated', 'Outpatient'], dtype=object)

df["raw_metadata_Infection_VRE"].unique()
array([nan, 'No'], dtype=object)

df = df.drop(columns=infection_cols)


df.to_csv("kesken3.tsv", sep="\t", index=False)

```
## Cancer

```
cancer_cols = [c for c in df.columns if c.startswith("raw_metadata_Cancer_")]

def get_cancer_list(row):
    cancers = []
    for col in cancer_cols:
        if row[col] == 1: 
            # remove prefix and clean underscores
            name = col.replace("raw_metadata_Cancer_", "").replace("_", " ")
            cancers.append(name)
    return ", ".join(cancers) if cancers else "None"

df["Cancer"] = df.apply(get_cancer_list, axis=1)


df = df.drop(columns=cancer_cols)


df.to_csv("kesken3.tsv", sep="\t", index=False)

```



