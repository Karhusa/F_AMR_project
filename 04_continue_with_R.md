
1. Load the data

```{r}

library(dplyr)
library(stringr)

setwd()
df <- read.table("kesken1.tsv", sep = "\t", header = TRUE)

```

2. Find uti columns
```{r}
uti_cols <- grep("uti", names(df), ignore.case = TRUE, value = TRUE)
uti_cols

for(col in uti_cols) {
  cat("Column:", col, "\n")
  print(unique(df[[col]]))
  cat("\n")
}

# Create new column indicating if patient ever had a UTI
df$UTI_history <- ifelse(
  (!is.na(df$raw_metadata_UTIs) & df$raw_metadata_UTIs > 0) |
  (!is.na(df$raw_metadata_diagnosed_utis) & df$raw_metadata_diagnosed_utis == 1) |
  (!is.na(df$raw_metadata_ecoli_utis) & df$raw_metadata_ecoli_utis == 1) |
  (df$raw_metadata_history_of_recurrent_uti == "Recurrent UTIs"),
  "Yes",
  "No"
)

# 2. Remove the original columns used to create it
columns_to_remove <- c(
  "raw_metadata_UTIs",
  "raw_metadata_diagnosed_utis",
  "raw_metadata_ecoli_utis",
  "raw_metadata_history_of_recurrent_uti"
)

df <- df[ , !(names(df) %in% columns_to_remove)]

# 3. Check the new column
table(df$UTI_history, useNA = "ifany")
#   No   Yes 
#12989   109 
```


Age

```
age_cols <- grep("age", names(df), ignore.case = TRUE, value = TRUE)
age_cols

df <- df %>%
  mutate(
    Age_numeric_new = case_when(
      !is.na(age_years) ~ age_years,
      !is.na(age_days)  ~ age_days / 365,  # convert days to years
      TRUE ~ NA_real_
    ),
    # 2. If age_range has a numeric range or single number, take midpoint
    Age_numeric_new = ifelse(
      is.na(Age_numeric_new) & !is.na(age_range),
      sapply(str_extract_all(age_range, "\\d+\\.?\\d*"), function(x) {
        if(length(x) == 2) return(mean(as.numeric(x)))
        if(length(x) == 1) return(as.numeric(x))
        NA_real_
      }),
      Age_numeric_new
    ),
    # 3. Define age categories
    Age_category_new = case_when(
      Age_numeric_new < 1 ~ "baby (<1 years)",
      Age_numeric_new >= 1 & Age_numeric_new < 18 ~ "child (1-17 years)",
      Age_numeric_new >= 18 & Age_numeric_new <= 65 ~ "adult (18-65 years)",
      Age_numeric_new > 65 ~ "senior (>65 years)",
      TRUE ~ NA_character_
    )
  )

df$Age_numeric_new[is.na(df$Age_numeric_new)] <- case_when(
  df$raw_metadata_age_group[is.na(df$Age_numeric_new)] == "baby" ~ 0.5,
  df$raw_metadata_age_group[is.na(df$Age_numeric_new)] == "schoolage" ~ 6,
  df$raw_metadata_age_group[is.na(df$Age_numeric_new)] == "adult" ~ 30,
  TRUE ~ NA_real_
)

# Remove original age-related columns
columns_to_remove <- c("age_category", "age_days", "age_range", "age_years", "raw_metadata_age_group")
df <- df[ , !(names(df) %in% columns_to_remove)]

# Check the new columns
head(df[, c("Age_numeric_new", "Age_category_new")])

```


## 4. Antibiotics:

### 4.2. Other antibiotic related columns

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
"raw_metadata_antibiotics_with_admission_days",
"raw_metadata_total_antibiotic_days"

]

df = df.drop(columns=columns_to_drop)
```
