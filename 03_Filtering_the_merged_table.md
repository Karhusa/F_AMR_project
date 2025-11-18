
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

for col in bmi_cols:
    print(f"{col}: {df[col].unique()}")

#BMI_range: [nan '>30' '<30']
#bmi: [        nan 23.1        20.4        ... 21.33683793 24.27411265    26.21415974]
#bmi_range: [nan '20-25' '16-20' '18.5-28' 'over 25' '25-30']
#raw_metadata_bmi_for_age_z_score: [  nan  0.94  .... ]

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

# Remove columns that are no longer needed

columns_to_drop = [
    "bmi_range",
    "bmi",
    "BMI_range",
    "raw_metadata_bmi_for_age_z_score"
]

df = df.drop(columns=columns_to_drop)
```

## 4. Antibiotics:

### 4.1. Columns with name of the antibiotic

Look through what kind of values there are in the columns and collect the names of the antibiotics and put in a more cleaner form

```
antibiotic_cols_w = [c for c in df.columns if c.startswith("raw_metadata_w_")]
antibiotic_cols_c = [c for c in df.columns if c.startswith("raw_metadata_c_")]
antibiotic_cols_m = [c for c in df.columns if c.startswith("raw_metadata_m_")]

all_antibiotic_cols = antibiotic_cols_w + antibiotic_cols_c + antibiotic_cols_m

# Create a list of used antibiotics for each patient
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

antibiotics_expanded = pd.DataFrame(
    df["antibiotics_list"].to_list(),
    index=df.index
).rename(lambda x: f"antibiotic_{x+1}", axis=1)

df = df.join(antibiotics_expanded)

df["name_of_antibiotic"] = df.apply(get_antibiotics_list, axis=1)

# Sanity check
df["name_of_antibiotic"].apply(tuple).unique()

#Create a new column categorized as Yes or No
df["Antibiotics_used"] = df["antibiotics_list"].apply(lambda x: "Yes" if len(x) > 0 else "No")

# Remove original antibiotic columns
df = df.drop(columns=all_antibiotic_cols)

df.to_csv("kesken1.tsv", sep="\t", index=False)
```

### 4.2. Other antibiotic related columns

```
antibiotic_cols = df.columns[df.columns.str.contains("antibiotic", case=False)]

for col in antibiotic_cols:
    print(f"\n=== {col} ===")
    print(df[col].value_counts(dropna=False).head(10))

# look through the columns and remove unnecessary columns with non-important information
exclude_cols = ['antibiotics_list', 'name_of_antibiotic', 'Antibiotics_used'] + [f'antibiotic_{i}' for i in range(1, 10)]

filtered_antibiotic_cols = [col for col in antibiotic_cols if col not in exclude_cols]

print(filtered_antibiotic_cols)

#'Drug_antibiotic_last3y', 'days_since_antibiotics', #'range_days_since_antibiotics', #'raw_metadata_Antibiotics_Last3months', #'raw_metadata_Antibiotics_current', #'raw_metadata_Antibiotics_past_3_months', 'raw_metadata_antibiotic_use', #'raw_metadata_antibiotics', 'raw_metadata_antibiotics_at_birth', #'raw_metadata_antibiotics_with_admission_days', 'raw_metadata_total_antibiotic_days']

# Initialize numeric days column
df["days_since_antibiotics_combined"] = np.nan

# 3. Update based on days_since_antibiotics
df.loc[df["days_since_antibiotics"].notna(), "days_since_antibiotics_combined"] = df.loc[df["days_since_antibiotics"].notna(), "days_since_antibiotics"]

# 4. Update based on Drug_antibiotic_last3y == "2 months"
df.loc[df["Drug_antibiotic_last3y"] == "2 months", "days_since_antibiotics_combined"] = 60

# 5. Update based on range_days_since_antibiotics (take approximate midpoint)
def range_to_midpoint(x):
    if pd.isna(x):
        return np.nan
    if x == "0-30":
        return 15
    elif x == "0-90":
        return 45
    elif x == "30-180":
        return 105
    elif x == "0-180":
        return 90
    return np.nan

df.loc[df["range_days_since_antibiotics"].notna(), "days_since_antibiotics_combined"] = \
    df.loc[df["range_days_since_antibiotics"].notna(), "range_days_since_antibiotics"].apply(range_to_midpoint)

# 6. Update based on raw_metadata_Antibiotics_current == "Y" (currently on antibiotics)
df.loc[df["raw_metadata_Antibiotics_current"] == "Y", "days_since_antibiotics_combined"] = 0

# 7. Update based on raw_metadata_Antibiotics_past_3_months == "Y" (midpoint 0-90)
df.loc[df["raw_metadata_Antibiotics_past_3_months"] == "Y", "days_since_antibiotics_combined"] = 45

# 8. Update based on raw_metadata_antibiotics_at_birth == "YES" (set -1 days as flag)
df.loc[df["raw_metadata_antibiotics_at_birth"] == "YES", "days_since_antibiotics_combined"] = -1

# 9. Update based on raw_metadata_total_antibiotic_days > 0 (use total days as approximate)
df.loc[df["raw_metadata_total_antibiotic_days"] > 0, "days_since_antibiotics_combined"] = df.loc[df["raw_metadata_total_antibiotic_days"] > 0, "raw_metadata_total_antibiotic_days"]

# 10. Set Antibiotics_used = "Yes" if any of the sources indicate use
df.loc[
    (df["days_since_antibiotics_combined"].notna()) & (df["days_since_antibiotics_combined"] != -1),
    "Antibiotics_used"
] = "Yes"

# Optional: if you want "Yes" also for at birth
df.loc[df["days_since_antibiotics_combined"] == -1, "Antibiotics_used"] = "Yes"

# Remove columns that are no longer needed

columns_to_drop = [
    "Drug_antibiotic_last3y",
    "days_since_antibiotics",
    "range_days_since_antibiotics",
    "raw_metadata_Antibiotics_current",
    "raw_metadata_Antibiotics_past_3_months",
    "raw_metadata_antibiotic_use",
    "raw_metadata_antibiotics",
    "raw_metadata_antibiotics_with_admission_days",
]

df = df.drop(columns=columns_to_drop)

df.to_csv("kesken1.tsv", sep="\t", index=False)

```

## 5. Drop unnecessary columns

```

columns_to_drop = [
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
    "raw_metadata_Arthritis_uncertain_diagnosis",
    "raw_metadata_Autism_Asperger",
    "raw_metadata_Blood_urea_nitrogen",
    "raw_metadata_Drug_propranolol",
    "raw_metadata_NecrotizingEnterocolitis",
    "raw_metadata_Other_auto_immune",
    "raw_metadata_Other_skin_condition",
    "raw_metadata_Primary_biliary_cirrhosis",
    "raw_metadata_Treatment_(Y_or_N)",
    "raw_metadata_necrotizing_enterocolitis",
    "raw_metadata_previous_bilharzia_treatment"
    
]

df = df.drop(columns=columns_to_drop)

df.to_csv("kesken1.tsv", sep="\t", index=False)
```

## 6. acid

```
acid_cols = df.columns[df.columns.str.contains("acid", case=False)]
print(acid_cols)

# 'raw_metadata_Alpha_lipoc_acid', 'raw_metadata_Total_bile_acid',
# 'raw_metadata_Uric_acid', 'raw_metadata_valproic_acid'], dtype='object')

columns_to_drop = [
    "raw_metadata_Alpha_lipoc_acid",
    "raw_metadata_Total_bile_acid",
    "raw_metadata_Uric_acid",
    "raw_metadata_valproic_acid"
]

df = df.drop(columns=columns_to_drop)

```


## 6. Infections (kesken)

```
df = pd.read_csv("kesken1.tsv", sep="\t")

infection_cols = [c for c in df.columns if c.startswith("raw_metadata_Infection_")]

for col in infection_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_Infection_CRE: [nan 'No']
#raw_metadata_Infection_Control_Means: [nan 'Cohorted' 'Isolated' 'Outpatient']
#raw_metadata_Infection_ESBLs: [nan 'No']
#raw_metadata_Infection_MRSA: [nan 'No']
#raw_metadata_Infection_Other: [nan 'No']
#raw_metadata_Infection_VRE: [nan 'No']


df = df.drop(columns=infection_cols)

df.to_csv("kesken1.tsv", sep="\t", index=False)

```


```

infection_cols1 = df.columns[df.columns.str.contains("infection", case=False)]
for col in infection_cols1:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_GI_infection: [nan 'No' '3-4 years ago, stomach bug']
#raw_metadata_OtherInfection: [nan  0.  1.]
#raw_metadata_TrachealInfection: [nan  0.  1.]
#raw_metadata_UrineInfection: [nan  0.  1.]
#raw_metadata_multi_drug_resistant_organism_infection: [nan 'Negative' 'Positive']
#raw_metadata_schistosoma_infection_intensity: [  nan  4.    0.    4.33  5.67  9.67  2.    2.5   0.33 16.   10.    #5.33 31.5   5.   74.   26.33  0.67  4.67]

--> We can remove other 

```


## 7. Cancer

```
cancer_cols = df.columns[df.columns.str.contains("cancer", case=False)]
for col in cancer_cols:
    print(f"{col}: {df[col].unique()}")

#raw_metadata_Cancer_Hodgkin_s_lymphoma: [nan 'No']
#raw_metadata_Cancer_Non_Hodkin_s_lymphoma: [nan 'No']
#raw_metadata_Cancer_breast: [nan 'No']
#raw_metadata_Cancer_cholangiocarcinoma: [nan 'No']
#raw_metadata_Cancer_colon_or_rectum: [nan 'No']
#raw_metadata_Cancer_liver: [nan 'No']
#raw_metadata_Cancer_lung: [nan 'No']
#raw_metadata_Cancer_lymphoma_not_otherwise_specified: [nan 'No']
#raw_metadata_Cancer_ovarian: [nan 'No']
#raw_metadata_Cancer_prostate: [nan 'No']
#raw_metadata_Colon_Cancer: [nan 'No']
#raw_metadata_GI_cancer_past_3_months: [nan 'N' 'Y']
#raw_metadata_Thyroid_disease_uncertain_diagnosis_not_cancer: [nan 'No']
#raw_metadata_cancer_treatment: [nan 'No']

df = df.drop(columns=cancer_cols)

df.to_csv("kesken2.tsv", sep="\t", index=False)
```

## Disease

```
disease_cols = df.columns[df.columns.str.contains("disease", case=False)]
for col in disease_cols:
    print(f"{col}: {df[col].unique()}")


#raw_metadata_Celiac_disease: [nan 'No' 'N' 'Y']
#raw_metadata_Connective_tissue_disease: [nan 'No']
#raw_metadata_Crohns_disease: [nan 'N' 'Y']
#raw_metadata_Disease_activity_(Y_or_N): [nan 'Y' 'N']
#raw_metadata_Grave_s_disease: [nan 'No']
#raw_metadata_Heart_disease: [nan 'No']
#raw_metadata_Intestinal_disease: [nan 'N' 'Y']
#raw_metadata_Liver_disease: [nan 'No']
#raw_metadata_Lyme_disease: [nan 'No']
#raw_metadata_Parkinson_disease: [nan 'No']
#raw_metadata_Pectic_ulcer_disease: [nan 'No']
#raw_metadata_Thyroid_disease: [nan 'No']
#raw_metadata_diagnosed_with_disease: [nan 'No' 'Yes' 'not provided']
#raw_metadata_disease_cause: [nan 'HBV,alcohol' 'HBV' 'alcohol' 'Hepatitis C virus related'
 'HBV,Hepatitis E virus related']
#raw_metadata_disease_group: [nan 'Healthy' 'Stage_III_IV' 'Stage_I_II' 'MP' 'Stage_0' 'HS']
raw_metadata_diseases:


columns_to_drop = [
    "raw_metadata_Celiac_disease",
    "raw_metadata_Connective_tissue_disease",
    "raw_metadata_Crohns_disease",
    "raw_metadata_Disease_activity_(Y_or_N)",
    "raw_metadata_Grave_s_disease",
    "raw_metadata_Heart_disease",
    "raw_metadata_Lyme_disease",
    "raw_metadata_Parkinson_disease",
    "raw_metadata_Pectic_ulcer_disease",
    "raw_metadata_Thyroid_disease",
    "raw_metadata_diagnosed_with_disease",
    "raw_metadata_disease_cause"
]


df = df.drop(columns=disease_cols)
```



## 9. Age


```
age_cols = df.columns[df.columns.str.contains("age", case=False)]
print(age_cols)

#'age_category', 'age_days', 'age_range', 'age_years',
#       'environmental_package', 'raw_metadata_age_group',
#       'raw_metadata_weight_for_age_z_score'],
#      dtype='object')


df["raw_metadata_age_group"].unique()
df["age_days"].unique()
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



#remove separately
#df = df.drop(columns=["raw_metadata_housing_condition"])

print(f" Shape: {df.shape}")
#24605, 163

```

## Save file for later use so that you can reload and modify it again

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



Mental health
```



```

Muuta

```
def get_cancer_list(row):
    cancers = []
    for col in cancer_cols:
        if row[col] == 1: 
            # remove prefix and clean underscores
            name = col.replace("raw_metadata_Cancer_", "").replace("_", " ")
            cancers.append(name)
    return ", ".join(cancers) if cancers else "None"

df["Cancer"] = df.apply(get_cancer_list, axis=1)
```


