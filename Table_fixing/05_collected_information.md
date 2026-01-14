
## 1.Size and shape

```python
print("DataFrame shape:", df.shape)
#DataFrame shape: (24605, 39)

print(df.info())
```
---

## 2. How many samples we have with age, antibiotics usage and ctrlpatient

```python
sub = df[df["age_category"].notna()]

age_abx_counts = (
    sub
    .groupby(["age_category", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_abx_counts

age_abx_counts
Antibiotics_used             No  Yes
age_category                        
adolescent                  793    0
adolescent, adult             3    0
adult                     16253  417
baby                       1518  521
baby, child                 228    0
baby, child, adolescent      53    0
child                       791   56
child, adolescent            74    0
child, adolescent, adult   1165    0
```

## 3. Update the age_category column

Some values of the age_categoryncolumn include multiple categories
--> Lets fix this
```
child_age_years = {
    "1 to 13 (7)",
    "below 12 (12)",
    "2 to 5 (4)"
}

df.loc[df["age_years"].isin(child_age_years), "age_category"] = "child"



df.to_csv("kesken5.tsv", sep="\t", index=False)
```
age_sex_abx_counts = (
    sub
    .groupby(["age_category", "sex", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_sex_abx_counts

Antibiotics_used                   No  Yes
age_category             sex              
adolescent               female   331    0
                         male     458    0
adolescent, adult        male       3    0
adult                    female  5038  216
                         male    5105  201
baby                     female   365  282
                         male     417  201
child                    female   428   24
                         male     450   32
child, adolescent, adult female   718    0
                         male     444    0


age_sex_abx_counts = (
    sub
    .groupby(["age_category", "sex", "UTI_history", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_sex_uti_abx_counts


age_sex_uti_abx
Antibiotics_used                               No  Yes
age_category             sex    UTI_history           
adolescent               female No            331    0
                         male   No            458    0
adolescent, adult        male   No              3    0
adult                    female No           4907  216
                                Yes           131    0
                         male   No           5055  201
                                Yes            50    0
baby                     female No            362  261
                                Yes             3   21
                         male   No            404  169
                                Yes            13   32
child                    female No            428   24
                         male   No            450   32
child, adolescent, adult female No            718    0
                         male   No            444    0



age_sex_bmi_abx = (
    sub
    .groupby(["age_category", "sex", "BMI_range_new", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_sex_bmi_abx

age_sex_bmi_abx.reset_index().to_csv(
    "age_sex_bmi_antibiotics_counts.tsv",
    sep="\t",
    index=False
)

```
## Results

```

age_sex_bmi_abx
Antibiotics_used                                    No  Yes
age_category      sex    BMI_range_new                     
adolescent        female Normal (18.5-25)           14    0
                         Obese (>30)                87    0
                         Overweight (25-30)          5    0
                         Underweight (<18.5)         1    0
                  male   Normal (18.5-25)            9    0
                         Obese (>30)                87    0
                         Overweight (25-30)          4    0
                         Underweight (<18.5)        14    0
adolescent, adult male   Normal (18.5-25)            2    0
                         Underweight (<18.5)         1    0
adult             female Normal (18.5-25)         1539   30
                         Normal/Overweight (<30)    32    0
                         Obese (>30)               499   39
                         Overweight (25-30)        516   35
                         Underweight (<18.5)       204    4
                  male   Normal (18.5-25)         1552   30
                         Normal/Overweight (<30)    24    0
                         Obese (>30)               456   36
                         Overweight (25-30)        770   45
                         Underweight (<18.5)        73    0
baby              male   Normal (18.5-25)            0    1
                         Underweight (<18.5)         1    2
child             female Normal (18.5-25)           10    0
                         Obese (>30)                 2    0
                         Underweight (<18.5)        62   24
                  male   Normal (18.5-25)           11    6
                         Obese (>30)                 2    0
                         Overweight (25-30)          4    0
                         Underweight (<18.5)        55   24

```

age_sex_GI_abx = (
    sub
    .groupby(["age_category", "sex", "GI_disease_history", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_sex_GI_abx

age_sex_GI_abx.reset_index().to_csv(
    "age_sex_GI_antibiotics_counts.tsv",
    sep="\t",
    index=False
)

```


## Results:

```
age_sex_GI_abx
Antibiotics_used                                                No  Yes
age_category sex    GI_disease_history                                 
adolescent   female Crohn's disease                             96    0
                    helminthiasis                                7    0
                    ulcerative colitis                          45    0
             male   Crohn's disease                            222    0
                    helminthiasis                                7    0
                    ulcerative colitis                          55    0
adult        female Clostridium difficile infection             10    0
                    Colitis;Intestinal_disease                  15   13
                    Colitis;Intestinal_disease;IBD               3    0
                    Crohn's disease                            167    0
                    Intestinal_disease                          21   18
                    Intestinal_disease;IBD                       4    1
                    cholera                                     22    0
                    helminthiasis                               88    0
                    ulcerative colitis                         187    0
             male   Clostridium difficile infection              3    0
                    Colitis;Intestinal_disease                  26   16
                    Colitis;Intestinal_disease;Crohns_disease    0    1
                    Colitis;Intestinal_disease;IBD               2    0
                    Crohn's disease                             97    0
                    Intestinal_disease                          13    7
                    Intestinal_disease;Crohns_disease;IBD        0    1
                    Intestinal_disease;IBD                       3    2
                    cholera                                     14    0
                    helminthiasis                               67    0
                    ulcerative colitis                         107    0
child        female Crohn's disease                             42    0
                    cholera                                      9    0
                    helminthiasis                               68    0
                    indeterminate colitis                       17    0
                    schistosomiasis                              8    4
                    ulcerative colitis                          28    0
             male   Crohn's disease                            104    0
                    cholera                                      7    0
                    helminthiasis                               61    0
                    schistosomiasis                              2    4
                    ulcerative colitis                          16    0

```

```
Cancers_and_adenomas

age_sex_CA_abx = (
    sub
    .groupby(["age_category", "sex", "Cancers_and_adenomas", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_sex_CA_abx

age_sex_CA_abx.reset_index().to_csv(
    "age_sex_CA_antibiotics_counts.tsv",
    sep="\t",
    index=False
)

```
Results: 

```
age_category	sex	Cancers_and_adenomas	No	Yes
adolescent, adult	male	melanoma	1	0
adult	female	melanoma	131	4
adult	female	adenocarcinoma;colorectal cancer	4	0
adult	female	adenoma	10	0
adult	female	adenoma;advanced colorectal adenoma	1	0
adult	female	adenoma;familial adenomatous polyposis	12	0
adult	female	adenoma;nonadvanced colorectal adenoma	19	0
adult	female	breast cancer	118	0
adult	female	colorectal cancer	132	0
adult	female	non-small cell lung cancer	8	0
adult	male	melanoma	291	17
adult	male	adenocarcinoma;colorectal cancer	22	0
adult	male	adenoma	10	0
adult	male	adenoma;familial adenomatous polyposis	10	0
adult	male	adenoma;nonadvanced colorectal adenoma	48	0
adult	male	colorectal cancer	205	0
adult	male	non-small cell lung cancer	21	0<img width="391" height="393" alt="image" src="https://github.com/user-attachments/assets/edb65da1-190d-43fc-a250-a59c7908cccc" />

```



Preparing the TSE object
Set acc numbers as the rownames for colData
```
df = df.set_index("acc", drop=True)
df.index.is_unique

df_tse = df_tse.replace("", pd.NA)

```





```

Remove list-like columns from colData
df = df.drop(columns=["antibiotics_list", "name_of_antibiotic"])
```


```
categorical_cols = [
    "sex", "age_category", "Antibiotics_used", "UTI_history",
    "HIV_history", "COVID19_history", "CPE_infection_history",
    "Multidrug_resistant_infection", "Antivirals_used"
]

for c in categorical_cols:
    if c in df.columns:
        df[c] = df[c].astype("category")


```

Make sure only NA, no empty strings.

´´´

df = df.replace("", pd.NA)

```

Check.

```
df.index.is_unique
df.dtypes
df.isna().sum().sort_values(ascending=False).head(10)

````
