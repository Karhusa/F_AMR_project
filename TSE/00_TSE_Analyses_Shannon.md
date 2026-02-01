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
```

![Boxplot of ARG Shannon Diversity by Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/Boxplot_Shannon_diversity_by_sex.png)


### 4.2. Descriptive statistics (Shannon x Sex)

```r
colData_new_subset %>%
  filter(!is.na(sex), !is.na(ARG_div_shan)) %>% group_by(sex) %>%
  summarise(mean_ARGShannon = mean(ARG_div_shan), median_ARGShannon = median(ARG_div_shan), sd_ARGShannon = sd(ARG_div_shan), n = n())
```


| sex    | Mean Shannon | Median Shannon | sd_ARGShannon | N    |
|--------|----------------|-----------------|---------------|------|
| female | 1.90           | 1.94            | 0.515         | 7426 |
| male   | 1.87           | 1.93            | 0.532         | 7349 |

### 4.3. Wilcoxon test

```r
wilcox_res <- wilcox.test(ARG_div_shan ~ sex, data = plot_df)

p_label <- paste0("Wilcoxon p = ", signif(wilcox_res$p.value, 3))
```

### 4.4 Same Boxplot with wilcoxon test value

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
  annotate("text", x = 1.5, y = max(plot_df$ARG_div_shan) + 0.35, label = p_label, size = 4.2, fontface = "italic") +
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

![Boxplot of ARG Shannon Diversity by Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/wilcoxon_Boxplot_Shannon_diversity_by_sex.png)

## 5. Analyses of ARG Load by Age and Sex

### 5.1 Boxplot of ARG Load by Category and Sex

```r
colData_new_subset$sex <- factor(colData_new_subset$sex, levels = c("female", "male"))  # make it a factor

counts <- colData_new_subset %>%
  group_by(precise_age_category, sex) %>%
  summarise(
    N = n(),
    y_pos = max(ARG_div_shan, na.rm = TRUE) + 0.3,
    .groups = "drop"
  )

age_levels <- c(
  "Infant", "Toddler", "Child", "Teenage", 
  "Young adult", "Middle-Age Adult", "Older Adult", "Oldest Adult"
)

colData_new_subset <- colData_new_subset %>% mutate(precise_age_category = factor(precise_age_category, levels = age_levels))

ggplot(colData_new_subset, aes(x = precise_age_category, y = ARG_div_shan, fill = sex)) +
  geom_boxplot(alpha = 0.7, position = position_dodge(width = 0.8)) +
  scale_fill_manual(values = c("female" = "#B3A2C7", "male" = "#A6D854")) +
  geom_text(
    data = counts,
    aes(x = precise_age_category, y = y_pos, label = paste0("N=", N)),
    position = position_dodge(width = 0.8),
    size = 3
  ) +
  labs(
    x = "Age Category",
    y = "ARG Shannon diversity",
    title = "ARG Shannon Diversity by Age Category and Sex",
    fill = "Sex"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))

setwd("/scratch/project_2008149/USER_WORKSPACES/karhula/DATA")

ggsave("Boxplot_Shannon_diversity_by_age_sex.png", width = 8, height = 6, dpi = 300)
```
![Boxplot of ARG Shannon Diversity by Age and Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/Boxplot_Shannon_diversity_by_age_sex.png)

* N values can be removed. I left N values there so that it would be easier to interpret results.

### 6.2 Scatter plot + separate regression lines by sex

```r
colData_sex_clean <- colData_new_subset %>% filter(!is.na(sex))
N_sex <- colData_sex_clean %>%
  filter(!is.na(age_years), !is.na(ARG_div_shan)) %>%
  count(sex)

ggplot(colData_sex_clean,
       aes(x = age_years, y = ARG_div_shan, color = sex)) +
  geom_point(alpha = 0.25, size = 1) +
  geom_smooth(method = "lm", se = TRUE, linewidth = 1.4, alpha = 0.15
  ) +
  scale_color_manual(
    values = c("female" = "#B3A2C7", "male" = "#A6D854")
  ) +
  labs(x = "Age (years)", y = "ARG Shannon diversity", title = "ARG Shannon Diversity by Age and Sex", color = "Sex"
  ) +
  theme_minimal()

ggsave("Regression_Shannon_diversity_by_age_sex.png", width = 8, height = 6, dpi = 300)

```

![Regression analysis ARG Shannon diversity by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/Regression_Shannon_diversity_by_age_sex.png)

### 6.3 Linear regression with results
```
model <- lm(ARG_div_shan ~ age_years + sex, data = colData_sex_clean)
summary(model)

model_sum <- summary(model)

beta_age <- model_sum$coefficients["age_years", "Estimate"]
p_age    <- model_sum$coefficients["age_years", "Pr(>|t|)"]
r2 <- model_sum$r.squared
n  <- model_sum$df[1] + model_sum$df[2] 

ggplot(colData_sex_clean, aes(x = age_years, y = ARG_div_shan)
  ) +
  geom_point(alpha = 0.2, color = "grey60") +
  geom_smooth(aes(color = sex), method = "lm", se = TRUE,linewidth = 1.5
  ) +
  scale_color_manual(
    values = c("female" = "#B3A2C7", "male" = "#A6D854")
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
  labs(x = "Age (years)", y = "ARG Shannon diversity", title = "ARG Shannon Diversity by Age and Sex", color = "Sex"
  ) +
  theme_minimal()

ggsave("Regression_with_table_ARG_Shannon_by_age_sex.png", width = 8, height = 6, dpi = 300)
```

![Regression analysis with table ARG Load by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/Regression_with_table_ARG_load_by_age_sex.png)

| Term        | Estimate | Std. Error | t value | Pr(>|t|)    | Significance |
|------------|---------|------------|---------|------------|-------------|
| (Intercept)| 1.7123  | 0.0103     | 165.93  | < 2e-16    | ***         |
| age_years  | 0.00603 | 0.00020    | 29.81   | < 2e-16    | ***         |
| sexmale    | -0.06130| 0.00970    | -6.32   | 2.73e-10   | ***         |


* Residual standard error: 0.4864 on 10066 degrees of freedom  
* Multiple R-squared: 0.0836, Adjusted R-squared: 0.0834  
* F-statistic: 459 on 2 and 10066 DF, p-value: < 2.2e-16  
* 4706 observations deleted due to missingness


### 6.4 Generalized Additive Model (GAM)
* A GAM models the outcome as a sum of smooth functions of predictors rather than simple linear effects.

```r

library(mgcv)

gam_model <- gam(
  ARG_div_shan ~ s(age_years) + sex,
  data = colData_sex_clean,
  method = "REML"
)

summary(gam_model)

ggplot(colData_sex_clean, aes(x = age_years, y = ARG_div_shan)) +
  geom_point(alpha = 0.2, color = "grey70") +
  geom_smooth(
    aes(color = sex),
    method = "gam",
    formula = y ~ s(x),
    se = TRUE,
    linewidth = 1.5
  ) +
  scale_color_manual(values = c("female" = "#B3A2C7", "male" = "#A6D854")) +
  labs(x = "Age (years)", y = "ARG Shannon", title = "ARG Shannon vs Age by Sex (GAM)", color = "Sex") +
  theme_minimal()

ggsave("GAM_ARG_Shannon_by_age_sex.png", width = 8, height = 6, dpi = 300)

```
![Gam analysis ARG Shannon by Age and Sex](https://github.com/Karhusa/Gender_differences_in_AMR/blob/main/Results/GAM_ARG_Shannon_by_age_sex.png)

| Term        | Estimate | Std. Error | t value | Pr(>|t|)    | Significance |
|------------|---------|------------|---------|------------|-------------|
| (Intercept)| 1.9466  | 0.00689    | 282.43  | < 2e-16    | ***         |
| sexmale    | -0.06394| 0.00970    | -6.595  | 4.47e-11   | ***         |

| Term          | edf    | Ref.df | F value | p-value | Significance |
|---------------|-------|--------|---------|---------|-------------|
| s(age_years)  | 7.985 | 8.688  | 115.3   | < 2e-16 | ***         |


* Family: Gaussian, link = identity  
*  R-squared (adj) = 0.093
*  Deviance explained = 9.38%  
* REML = 7000.7  
* Scale estimate = 0.23406  
* Sample size (n) = 10069



---

