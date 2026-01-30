ARG Shannon Analysis by Sex

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

colData_new_subset <- colData_df %>%
  select(
    acc,
    ARG_div_shan,
    age_years,
    log10_ARG_load, 
    sex, 
    BMI_range_new, 
    Antibiotics_used, 
    UTI_history, 
    precise_age_category, imprecise_age_category
  )

```

## 4. Ananlyses of Shannon diversity index and sex

### 4.1.  Boxplot of Shannon diversity index and sex

```r
colData_new_subset$sex[colData_new_subset$sex == "" | colData_new_subset$sex == "NA"] <- NA  # convert empty/NA strings to actual NA
plot_df <- colData_new_subset %>% filter(!is.na(ARG_div_shan), !is.na(sex))

n_df <- plot_df %>% count(sex)

ggplot(plot_df, aes(x = sex, y = ARG_div_shan, fill = sex)) +
  geom_jitter(
    width = 0.15,
    size = 1.2,
    alpha = 0.25,
    color = "grey30"
  ) +
  geom_boxplot(
    width = 0.55,
    outlier.shape = NA,
    alpha = 0.8,
    color = "grey25"
  ) +
  geom_text(
    data = n_df,
    aes(
      x = sex,
      y = y_max + 0.05,
      label = paste0("N = ", n)
    ),
    inherit.aes = FALSE,
    size = 4
  ) +
  scale_fill_manual(values = c("#B3A2C7", "#A6D854")) +
  labs(
    title = "ARG Shannon diversity by Sex",
    x = "Sex",
    y = "ARG Shannon diversity"
  ) +
  theme_minimal(base_size = 13) +
  theme(
    legend.position = "none",
    plot.title = element_text(face = "bold"),
    axis.title.x = element_text(margin = margin(t = 10)),
    axis.title.y = element_text(margin = margin(r = 10))
  )


ggsave("Boxplot_Shannon_diversity_by_sex.png", width = 8, height = 6, dpi = 300)

´´´
### 4.2. Descriptive statistics (Shannon x Sex)
´´´r
colData_new_subset %>%
  filter(!is.na(sex), !is.na(ARG_div_shan)) %>% group_by(sex) %>%
  summarise(mean_ARGShannon = mean(ARG_div_shan), median_ARGShannon = median(ARG_div_shan), sd_ARGShannon = sd(ARG_div_shan), n = n())
```
| Sex | Mean | Median | SD | N |
|-----|-----:|-------:|---:|--:|
| Female | 2.75 | 2.76 | 0.311 | 7,426 |
| Male   | 2.72 | 2.72 | 0.311 | 7,349 |


| sex    | mean_ARGShannon | median_ARGShannon | sd_ARGShannon | n    |
|--------|----------------|-----------------|---------------|------|
| female | 1.90           | 1.94            | 0.515         | 7426 |
| male   | 1.87           | 1.93            | 0.532         | 7349 |

```
### 4.3. Wilcoxon test

```
# Wilcoxon test
wilcox_res <- wilcox.test(ARG_div_shan ~ sex, data = plot_df)

p_label <- paste0("Wilcoxon p = ", signif(wilcox_res$p.value, 3))

```

### 

```r
ggplot(plot_df, aes(x = sex, y = ARG_div_shan, fill = sex)) +
  geom_jitter(
    width = 0.15,
    size = 1.2,
    alpha = 0.25,
    color = "grey30"
  ) +
  geom_boxplot(
    width = 0.55,
    outlier.shape = NA,
    alpha = 0.8,
    color = "grey25"
  ) +
  geom_text(
    data = n_df,
    aes(
      x = sex,
      y = y_max + 0.05,
      label = paste0("N = ", n)
    ),
    inherit.aes = FALSE,
    size = 4
  ) +
  annotate("text", x = 1.5, y = max(plot_df$log10_ARG_load) + 0.35, label = p_label, size = 4.2, fontface = "italic") +
  scale_fill_manual(values = c("#B3A2C7", "#A6D854")) +
  labs(
    title = "ARG Shannon diversity by Sex",
    x = "Sex",
    y = "ARG Shannon diversity"
  ) +
  theme_minimal(base_size = 13) +
  theme(
    legend.position = "none",
    plot.title = element_text(face = "bold"),
    axis.title.x = element_text(margin = margin(t = 10)),
    axis.title.y = element_text(margin = margin(r = 10))
  )

ggsave("wilcoxon_Boxplot_Shannon_diversity_by_sex.png", width = 8, height = 6, dpi = 300)
```

## 5. Analyses of ARG Load by Age and Sex

### 5.1 Boxplot of ARG Load by Category and Sex

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

setwd("/scratch/project_2008149/USER_WORKSPACES/karhula/DATA")
ggsave("ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```
![ARG Load by Age and Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/ARG_load_by_age_sex.png)

* N values can be removed. I left N values there so that it would be easier to interpret results.

### 6.2 Scatter plot + separate regression lines by sex

```r
colData_sex_clean <- colData_subset %>% filter(!is.na(sex))
N_sex <- colData_sex_clean %>%
  filter(!is.na(age_years), !is.na(log10_ARG_load)) %>%
  count(sex)

ggplot(colData_sex_clean,
       aes(x = age_years, y = log10_ARG_load, color = sex)) +
  geom_point(alpha = 0.25, size = 1) +
  geom_smooth(method = "lm", se = TRUE, linewidth = 1.4, alpha = 0.15
  ) +
  scale_color_manual(
    values = c("female" = "#D55E00", "male" = "#0072B2")
  ) +
  labs(x = "Age (years)", y = "Log10 ARG load", title = "ARG load vs Age by Sex", color = "Sex"
  ) +
  theme_minimal()
```

![Regression analysis ARG Load by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/Regression_ARG_load_by_age_sex.png)

### 6.3 Linear regression with results
```
model <- lm(log10_ARG_load ~ age_years + sex, data = colData_sex_clean)
summary(model)

model_sum <- summary(model)

beta_age <- model_sum$coefficients["age_years", "Estimate"]
p_age    <- model_sum$coefficients["age_years", "Pr(>|t|)"]

r2 <- model_sum$r.squared
n  <- model_sum$df[1] + model_sum$df[2] + 

ggplot(colData_sex_clean, aes(x = age_years, y = log10_ARG_load)
  ) +
  geom_point(alpha = 0.2, color = "grey60") +
  geom_smooth(aes(color = sex), method = "lm", se = TRUE,linewidth = 1.5
  ) +
  scale_color_manual(
    values = c("female" = "#D55E00", "male" = "#0072B2")
  ) +
  annotate("text", x = Inf, y = Inf,
    hjust = 1.1, vjust = 1.2,
    size = 4,
    label = paste0(
      "Age effect: β = ", round(beta_age, 4),
      "\nP = ", signif(p_age, 3),
      "\nR² = ", round(r2, 3),
      "\nN = ", n
    )
  ) +
  labs(x = "Age (years)", y = "Log10 ARG load", title = "ARG load vs Age by Sex", color = "Sex"
  ) +
  theme_minimal()

ggsave("Regression_with_table_ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```

![Regression analysis with table ARG Load by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/Regression_with_table_ARG_load_by_age_sex.png)

| Section                 | Term        | Estimate / Value | Std. Error | Statistic   | p-value | Signif. |
|-------------------------|------------|----------------|------------|------------|---------|---------|
| Parametric coefficients | (Intercept)| 2.6921235      | 0.0064149  | t = 419.667 | <2e-16 | ***     |
| Parametric coefficients | age_years  | 0.0014798      | 0.0001257  | t = 11.775  | <2e-16 | ***     |
| Parametric coefficients | sexmale    | -0.0084796     | 0.0060318  | t = -1.406  | 0.16   | –       |



| Metric | Value |
|-------|-------|
| Residual standard error | 0.3024 |
| Degrees of freedom | 10066 |
| Observations removed (missingness) | 4706 |
| Multiple R-squared | 0.01369 |
| Adjusted R-squared | 0.0135 |
| F-statistic | 69.87 (2, 10066 DF) |
| Model p-value | < 2.2e-16 |

### 6.4 Generalized Additive Model (GAM)
* A GAM models the outcome as a sum of smooth functions of predictors rather than simple linear effects.

```r

library(mgcv)

gam_model <- gam(
  log10_ARG_load ~ s(age_years) + sex,
  data = colData_sex_clean,
  method = "REML"
)

summary(gam_model)

ggplot(colData_sex_clean, aes(x = age_years, y = log10_ARG_load)) +
  geom_point(alpha = 0.2, color = "grey70") +
  geom_smooth(
    aes(color = sex),
    method = "gam",
    formula = y ~ s(x),
    se = TRUE,
    linewidth = 1.5
  ) +
  scale_color_manual(values = c("female" = "#D55E00", "male" = "#0072B2")) +
  labs(x = "Age (years)", y = "Log10 ARG load", title = "ARG load vs Age by Sex (GAM)", color = "Sex") +
  theme_minimal()

ggsave("GAM_ARG_load_by_age_sex.png", width = 8, height = 6, dpi = 300)
```
![Regression analysis with table ARG Load by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/GAM_ARG_load_by_age_sex.png)

| Section | Term | Estimate / Value | Std. Error | Statistic | p-value | Signif. |
|--------|------|------------------|------------|-----------|---------|---------|
| Model information | Family | Gaussian | – | – | – | – |
| Model information | Link function | Identity | – | – | – | – |
| Model information | Formula | log10_ARG_load ~ s(age_years) + sex | – | – | – | – |
| Model information | Sample size (n) | 10069 | – | – | – | – |
| Parametric coefficients | (Intercept) | 2.751674 | 0.004265 | t = 645.156 | <2e-16 | *** |
| Parametric coefficients | sexmale | -0.013133 | 0.005999 | t = -2.189 | 0.0286 | * |
| Smooth terms | s(age_years) | – | – | F = 41.25 (edf = 7.73, Ref.df = 8.392) | <2e-16 | *** |
| Model fit | Adjusted R-squared | 0.033 | – | – | – | – |
| Model fit | Deviance explained | 3.39% | – | – | – | – |
| Model fit | REML | 2171.1 | – | – | – | – |
| Model fit | Scale estimate | 0.089663 | – | – | – | – |

---

