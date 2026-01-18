
ARG Load Analysis by Sex
```r
library(SummarizedExperiment)
library(dplyr)
library(ggplot2)

```
Load TSE
```r
tse_path <- "/scratch/project_2008149/USER_WORKSPACES/karhula/DATA/TSE.rds"
TSE <- readRDS(tse_path)

```
# Extract colData

```r
colData_df <- as.data.frame(colData(TSE))

# Keep relevant columns
colData_age <- colData_df %>%
  select(acc, sex_combined, host_age_years, ARG_load_rpk)

colData_age2 <- colData_df %>%
  select(acc, sex, age_category_new, ARG_load_rpk)

# Quick check
head(colData_age)
summary(colData_age)

head(colData_age2)
summary(colData_age2)

```

Visualize colData_age
```r
ggplot(colData_age, aes(x = host_age_years, y = ARG_load_rpk, color = sex_combined)) +
  geom_point(alpha = 0.4) +
  geom_smooth(method = "loess") +   
  theme_minimal() +
  labs(
    x = "Age (years)",
    y = "ARG load (RPK)",
    color = "Sex",
    title = "ARG load across age by sex"
  )



```


```r

ggplot(colData_age2, aes(x = age_category_new, y = ARG_load_rpk, color = sex)) +
  geom_point(alpha = 0.4) +
  geom_smooth(method = "loess") +    # local regression to see trend
  theme_minimal() +
  labs(
    x = "Age (years)",
    y = "ARG load (RPK)",
    color = "Sex",
    title = "ARG load across age by sex"
  )

```

```
