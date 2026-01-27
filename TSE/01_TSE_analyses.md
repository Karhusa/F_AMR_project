ARG Load Analysis by Sex

---
## 1. Packages
```r
library(SummarizedExperiment)
library(dplyr)
library(ggplot2)
library(tidyr)
library(gtsummary)
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
---
## 4.Look through specific columns

### 4.1 Precise_age_category
```
colData_subset$precise_age_category[colData_subset$precise_age_category == "Unknown"] <- NA
table(colData_subset$precise_age_category)
sum(!is.na(colData_subset$precise_age_category))
```
Results:

* Infant 1406
* Toddler 185
* Child 837
* Teeanage 1127
* Young Adult 2583
* Middle-Age Adult 5666
* Older Adult 1941
* Oldest adult 212
* All together 13957


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
```
* Ages 0 to 109, all ages presented
* All together 11189 

### 4.3 sex 

```r
colData_subset$sex[colData_subset$sex == "" | colData_subset$sex == "NA"] <- NA  # convert empty/NA strings to actual NA
table(colData_subset$sex)
sum(!is.na(colData_subset$sex))
```
Results:
* Female 7426
* Male 7249
* All together 14775

### 4.4  BMI_range_new
```r
colData_subset$BMI_range_new <- as.character(colData_subset$BMI_range_new)  
colData_subset$BMI_range_new[colData_subset$BMI_range_new == "" | colData_subset$BMI_range_new == "NA"] <- NA

table(colData_subset$BMI_range_new)

sum(!is.na(colData_subset$BMI_range_new))
```
Results:
* Underweight (<18.5)  467
* Normal (18.5-25)    3294
* Overweight (25-30)  1478
* Obese (>30)         1231
* Normal/Overweight (<30) # We will leave this out
* Alltogether 6526

---
## 5. Analyses of ARG Load by Age and Sex

## 5.1 Boxplot of ARG Load by Category and Sex

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
## 5.2 Save image

```r
setwd("/scratch/project_2008149/USER_WORKSPACES/karhula/DATA")
ggsave("ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```
![ARG Load by Age and Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/ARG_load_by_age_sex.png)

* N values can be removed. I left N values there so that it would be easier to interpret results.

## 5.3 Scatter plot + separate regression lines by sex

```r
colData_sex_clean <- colData_subset %>% filter(!is.na(sex))
N_sex <- colData_sex_clean %>%
  filter(!is.na(age_years), !is.na(log10_ARG_load)) %>%
  count(sex)

ggplot(colData_sex_clean,
       aes(x = age_years, y = log10_ARG_load, color = sex)) +
  geom_point(alpha = 0.25, size = 1) +
  geom_smooth(
    method = "lm",
    se = TRUE,
    linewidth = 1.4,
    alpha = 0.15
  ) +
  scale_color_manual(
    values = c("female" = "#D55E00", "male" = "#0072B2")
  ) +
  labs(
    x = "Age (years)",
    y = "Log10 ARG load",
    title = "ARG load vs Age by Sex",
    color = "Sex"
  ) +
  theme_minimal()
```
### 5.4. Save image
```r
ggsave("Regression_with_table_ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```

![Regression analysis ARG Load by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/Regression_ARG_load_by_age_sex.png)

### 5.5 Linear regression with results
```
model <- lm(log10_ARG_load ~ age_years + sex, data = colData_sex_clean)
summary(model)

model_sum <- summary(model)

beta_age <- model_sum$coefficients["age_years", "Estimate"]
p_age    <- model_sum$coefficients["age_years", "Pr(>|t|)"]

r2 <- model_sum$r.squared
n  <- model_sum$df[1] + model_sum$df[2] + 

ggplot(colData_sex_clean,
       aes(x = age_years, y = log10_ARG_load)) +
  geom_point(alpha = 0.2, color = "grey60") +
  geom_smooth(
    aes(color = sex),
    method = "lm",
    se = TRUE,
    linewidth = 1.5
  ) +
  scale_color_manual(
    values = c("female" = "#D55E00", "male" = "#0072B2")
  ) +
  annotate(
    "text",
    x = Inf, y = Inf,
    hjust = 1.1, vjust = 1.2,
    size = 4,
    label = paste0(
      "Age effect: β = ", round(beta_age, 4),
      "\nP = ", signif(p_age, 3),
      "\nR² = ", round(r2, 3),
      "\nN = ", n
    )
  ) +
  labs(
    x = "Age (years)",
    y = "Log10 ARG load",
    title = "ARG load vs Age by Sex",
    color = "Sex"
  ) +
  theme_minimal()

ggsave("Regression_with_table_ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```

![]()




### 5.7. GAM
```r

library(mgcv)

gam_model <- gam(
  log10_ARG_load ~ s(age_years) + sex,
  data = colData_sex_clean,
  method = "REML"
)

summary(gam_model)

ggplot(colData_sex_clean,
       aes(x = age_years, y = log10_ARG_load)) +
  geom_point(alpha = 0.2, color = "grey70") +
  geom_smooth(
    aes(color = sex),
    method = "gam",
    formula = y ~ s(x),
    se = TRUE,
    linewidth = 1.5
  ) +
  scale_color_manual(
    values = c("female" = "#D55E00", "male" = "#0072B2")
  ) +
  labs(
    x = "Age (years)",
    y = "Log10 ARG load",
    title = "ARG load vs Age by Sex (GAM)",
    color = "Sex"
  ) +
  theme_minimal()
```

Family: gaussian 
Link function: identity 

Formula:
log10_ARG_load ~ s(age_years) + sex

Parametric coefficients:
             Estimate Std. Error t value Pr(>|t|)    
(Intercept)  2.751674   0.004265 645.156   <2e-16 ***
sexmale     -0.013133   0.005999  -2.189   0.0286 *  
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

Approximate significance of smooth terms:
              edf Ref.df     F p-value    
s(age_years) 7.73  8.392 41.25  <2e-16 ***
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

R-sq.(adj) =  0.033   Deviance explained = 3.39%
-REML = 2171.1  Scale est. = 0.089663  n = 10069

* sexmale  Estimate = -0.0131, p = 0.0286 (males have sligthly lower log10 ARG load than females)
* edf: 7.73 (curve?)
* F = 41.25
* p < 2e-16
* Adjusted R2 = 0.33 
* n = 10 069

---

## 6. Analyses of ARG Load by BMI and Sex

```r
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
