
ARG Load Analysis by Sex

```r
library(SummarizedExperiment)
library(dplyr)
library(ggplot2)
library(tidyr)
```
Load TSE
```r
tse_path <- "/scratch/project_2008149/USER_WORKSPACES/karhula/DATA/TSE.rds"
TSE <- readRDS(tse_path)

```
# Extract colData

```r
colData_df <- as.data.frame(colData(TSE))

colData_subset <- colData_df %>%
  select(
    acc,
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

2. AGE

```r
# Clean the sex column
colData_subset$sex <- as.character(colData_subset$sex) 
colData_subset$sex[colData_subset$sex == "" | colData_subset$sex == "NA"] <- NA  # convert empty/NA strings to actual NA
colData_subset$sex <- factor(colData_subset$sex, levels = c("female", "male"))  # make it a factor

# Compute counts per age category × sex
counts <- colData_subset %>%
  group_by(precise_age_category, sex) %>%
  summarise(
    N = n(),
    y_pos = max(log10_ARG_load, na.rm = TRUE) + 0.2,  # position above box
    .groups = "drop"
  )

# Plot with matching colors for boxes and N labels
ggplot(colData_subset, aes(x = precise_age_category, y = log10_ARG_load, fill = sex)) +
  geom_boxplot(alpha = 0.7, position = position_dodge(width = 0.8), color = "black") +
  scale_fill_manual(values = c("female" = "#FF9999", "male" = "#9999FF", "NA" = "grey70"),
                    labels = c("Female", "Male", "Unknown")) +
  geom_text(
    data = counts,
    aes(x = precise_age_category, y = y_pos, label = paste0("N=", N), color = sex),
    position = position_dodge(width = 0.8),
    size = 3,
    show.legend = FALSE  # don't add a separate legend for text
  ) +
  scale_color_manual(values = c("female" = "#FF9999", "male" = "#9999FF", "NA" = "grey70")) +
  labs(
    x = "Age Category",
    y = "Log10 ARG Load",
    title = "ARG Load by Age Category and Sex",
    fill = "Sex"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

![ARG Load by Age and Sex](https://github.com/Karhusa/F_AMR_project/blob/main/Results/ARG_load_by_age_sex.png)


# N values can be removed. I left N values there so that it would be easier to interpret results.


```
