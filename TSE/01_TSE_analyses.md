ARG Load Analysis by Sex

---
## 1. Packages
```r
library(SummarizedExperiment)
library(dplyr)
library(ggplot2)
library(tidyr)
```
## 2. Load TSE

```r
tse_path <- "/scratch/project_2008149/USER_WORKSPACES/karhula/DATA/TSE.rds"
TSE <- readRDS(tse_path)
```
## 3. Extract colData

```r
colData_df <- as.data.frame(colData(TSE))

colData_subset <- colData_df %>%
  select(
    acc,
    age_years,
    log10_ARG_load, 
    sex, 
    BMI_range_new, 
    Antibiotics_used, 
    UTI_history, 
    GI_disease_history, 
    Cancers_and_adenomas,
    Vancomycin, Ampicillin, Cefotaxime, TicarcillinClavulanate, Gentamicin,
    Clindamycin, Cefazolin, Oxacillin, Cefoxitin, Cefepime, 
    Meropenem, PenicillinG, Amoxacillin, AmpicillinSulbactam, SulfamethoxazoleTrimethoprim,
    precise_age_category, imprecise_age_category
  )
```

## 4.Look through specific columns

### 4.1 Precise_age_category
```
table(colData_subset$precise_age_category)

 #          Child           Infant Middle-Age Adult      Older Adult     Oldest Adult 
 #            837             1406             5666             1941              212 
 #        Teenage          Toddler          Unknown      Young adult 
 #           1127              185            10648             2583 

colData_subset$precise_age_category[colData_subset$precise_age_category == "Unknown"] <- NA

table(colData_subset$precise_age_category)

#           Child           Infant Middle-Age Adult      Older Adult     Oldest Adult 
#             837             1406             5666             1941              212 
#         Teenage          Toddler      Young adult 
#            1127              185             2583 
sum(!is.na(colData_subset$precise_age_category))
# 13957
```

### 4.2 age_years

```r

table(colData_subset$age_years)

# Many values (some are babys ages like 0.00XXXXXXXXXXXXX, so rounding up might be the best way to make sense to these numbers)

colData_subset$age_years <- round(colData_subset$age_years)

#   0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18 
# 1098  235   34   48   48   53   53   59  113   80  117  123  100   82  101  160  232  119  159 
#  19   20   21   22   23   24   25   26   27   28   29   30   31   32   33   34   35   36   37 
# 170  114  268  276  268  189  203  187   91  289  141  129   91  137   76   82   96  138  161 
#  38   39   40   41   42   43   44   45   46   47   48   49   50   51   52   53   54   55   56 
#  62   37  102   59   58   78   86  101   84  110   90   59  130  151   71   98   92  138  108 
#  57   58   59   60   61   62   63   64   65   66   67   68   69   70   71   72   73   74   75 
# 107  148  160  179  173  279  144  167  194  145  146  156  173  169  130   92  119  125   89 
#  76   77   78   79   80   81   82   83   84   85   86   87   88   90   91   92   98   99  100 
#  88   81   63   38   28   30   19   13   13    8   22    8    3    1    6    1    1    4    5 
# 101  102  104  105  106  107  108  109 
#   3    1    2   10    4    4    1    3 

sum(!is.na(colData_subset$age_years))
#11189

```
### 4.3 sex 

```r
table(colData_subset$sex)
#       female   male 
#  9830   7426   7349

colData_subset$sex[colData_subset$sex == "" | colData_subset$sex == "NA"] <- NA  # convert empty/NA strings to actual NA

table(colData_subset$sex)
#female   male 
#  7426   7349 

sum(!is.na(colData_subset$sex))
# 14775
```

### 4.4  BMI_range_new
```
table(colData_subset$BMI_range_new)

                               Normal (18.5-25) Normal/Overweight (<30)             Obese (>30) 
                  18079                    3294                      56                    1231 
     Overweight (25-30)     Underweight (<18.5) 
                   1478                     467 

# ensure it's character
colData_subset$BMI_range_new <- as.character(colData_subset$BMI_range_new)  
colData_subset$BMI_range_new[colData_subset$BMI_range_new == "" | colData_subset$BMI_range_new == "NA"] <- NA

table(colData_subset$BMI_range_new)

#       Normal (18.5-25) Normal/Overweight (<30)             Obese (>30) 
#                   3294                      56                    1231 
#     Overweight (25-30)     Underweight (<18.5) 
#                   1478                     467 

sum(!is.na(colData_subset$BMI_range_new))
#6526

```

## 5. Boxplot of ARG Load by Category and Sex

```r
colData_subset$sex <- factor(colData_subset$sex, levels = c("female", "male"))  # make it a factor

# Compute counts per age category × sex
counts <- colData_subset %>%
  group_by(precise_age_category, sex) %>%
  summarise(
    N = n(),
    y_pos = max(log10_ARG_load, na.rm = TRUE) + 0.2,  # position above box
    .groups = "drop"
  )

age_levels <- c(
  "Infant", "Toddler", "Child", "Teenage", 
  "Young adult", "Middle-Age Adult", "Older Adult", "Oldest Adult"
)

colData_subset <- colData_subset %>%
  mutate(precise_age_category = factor(precise_age_category, levels = age_levels))

ggplot(colData_subset, aes(x = precise_age_category, y = log10_ARG_load, fill = sex)) +
  geom_boxplot(alpha = 0.7, position = position_dodge(width = 0.8)) +
  scale_fill_manual(values = c("female" = "#FF9999", "male" = "#9999FF")) +
  geom_text(
    data = counts,
    aes(x = precise_age_category, y = y_pos, label = paste0("N=", N)),
    position = position_dodge(width = 0.8),
    size = 3
  ) +
  labs(
    x = "Age Category",
    y = "Log10 ARG Load",
    title = "ARG Load by Age Category and Sex",
    fill = "Sex"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))

```
## 5.1 Save image

```
setwd("/scratch/project_2008149/USER_WORKSPACES/karhula/DATA")

ggsave("ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```

![ARG Load by Age and Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/ARG_load_by_age_sex.png)


# N values can be removed. I left N values there so that it would be easier to interpret results.


```

colData_subset_clean <- colData_subset %>%
  filter(BMI_range_new != "Normal/Overweight (<30)")

# Reorder BMI categories
colData_subset_clean$BMI_range_new <- factor(
  colData_subset_clean$BMI_range_new,
  levels = c("Underweight (<18.5)", "Normal (18.5-25)", 
             "Overweight (25-30)", "Obese (>30)")
)

# Count N per category + sex
counts <- colData_subset_clean %>%
  group_by(BMI_range_new, sex) %>%
  summarise(N = n(), .groups = "drop") %>%
  mutate(y_pos = max(colData_subset_clean$log10_ARG_load, na.rm = TRUE) + 0.1)

# Plot
ggplot(colData_subset_clean, aes(x = BMI_range_new, y = log10_ARG_load, fill = sex)) +
  geom_boxplot(alpha = 0.7, position = position_dodge(width = 0.8), na.rm = TRUE) +
  scale_fill_manual(values = c("female" = "#FF9999", "male" = "#9999FF")) +
  geom_text(
    data = counts,
    aes(x = BMI_range_new, y = y_pos, label = paste0("N=", N), color = sex),
    position = position_dodge(width = 0.8),
    size = 3
  ) +
  labs(
    x = "BMI Category",
    y = "Log10 ARG Load",
    title = "ARG Load by BMI Category and Sex",
    fill = "Sex"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
