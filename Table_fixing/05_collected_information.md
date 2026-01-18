## Headline

## 1.Size and shape

```python
import numpy as np
import pandas as pd
import re

df = pd.read_csv("kesken4.tsv", sep="\t")

print("DataFrame shape:", df.shape)
#DataFrame shape: (24605, 39)

print(df.info())
```
---

## 3. Create a new age category: age_category_new

```python

import numpy as np
import pandas as pd
import re

def extract_age(val):
    if pd.isna(val):
        return np.nan
    if isinstance(val, (int, float)):
        return float(val)
    val = str(val)
    match = re.search(r"\(([\d\.]+)\)", val)
    if match:
        return float(match.group(1))
    match = re.search(r"[\d\.]+", val)
    if match:
        return float(match.group())
    return np.nan

def age_category_from_raw(val):
    age = extract_age(val)  
    if pd.isna(age):
        return "Unknown"
    elif age < 1:
        return "Infant"
    elif age < 3:
        return "Toddler"
    elif age < 12:
        return "Child"
    elif age < 20:
        return "Teenage"
    elif age < 35:
        return "Young adult"
    elif age < 65:
        return "Middle-Age Adult"
    elif age < 80:
        return "Older Adult"
    else:  # everything >= 80 goes here
        return "Oldest Adult"


df["age_category_new"] = df["age_years"].apply(age_category_from_raw)

df = df.drop(columns=["age_category"])

df.to_csv("kesken5.tsv", sep="\t", index=False)
```
___

## 4. How many samples we have with age, antibiotics usage and ctr lpatient

```python
sub = df[df["age_category_new"].notna()]

age_abx_counts = (
    sub
    .groupby(["age_category_new", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)

age_abx_counts
```

Results: 

Antibiotics_used    No  Yes
age_category_new           
Child             1969   48
Infant            1235  171
Middle-Age Adult  7265   64
Older Adult       2669  142
Oldest Adult       152   27
Out of range        33    0
Teenage           1334    0
Toddler            272  358
Unknown           6019    0
Young adult       2663  184

---

## 5. Age + Sex + antibiotics
```python
age_sex_abx_counts = (
    sub
    .groupby(["age_category_new", "sex", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)
```
Results:

Antibiotics_used           No  Yes
age_category_new sex              
Child            female  1020   23
                 male     814   25
Infant           female   330   81
                 male     370   52
Middle-Age Adult female  1962   44
                 male    1747   20
Older Adult      female   699   69
                 male    1795   73
Oldest Adult     female    82    9
                 male     100   18
Teenage          female   658    0
                 male     668    0
Toddler          female    47  202
                 male      52  156
Unknown          female   736    0
                 male     315    0
Young adult      female  1370   94
                 male    1054   90

---

## 5. AGE + SEX + UTI + ABX 

```python
age_sex_uti_abx_counts = (
    sub
    .groupby(["age_category_new", "sex", "UTI_history", "Antibiotics_used"])
    .size()
    .unstack(fill_value=0)
)
age_sex_uti_abx_counts
```
Results:

Antibiotics_used                       No  Yes
age_category_new sex    UTI_history           
Child            female No           1020   23
                 male   No            814   25
Infant           female No            328   62
                        Yes             2   19
                 male   No            360   26
                        Yes            10   26
Middle-Age Adult female No           1914   44
                        Yes            48    0
                 male   No           1726   20
                        Yes            21    0
Older Adult      female No            671   69
                        Yes            28    0
                 male   No           1766   73
                        Yes            29    0
Oldest Adult     female No             82    9
                 male   No            100   18
Teenage          female No            646    0
                        Yes            12    0
                 male   No            668    0
Toddler          female No             46  200
                        Yes             1    2
                 male   No             49  150
                        Yes             3    6
Unknown          female No            736    0
                 male   No            315    0
Young adult      female No           1327   94
                        Yes            43    0
                 male   No           1054   90

---

## 6. Age + Sex + BMI + abx 

```pyhton
age_sex_bmi_abx = (
    sub
    .groupby(["age_category_new", "sex", "BMI_range_new", "Antibiotics_used"])
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

Antibiotics_used                                  No  Yes
age_category_new sex    BMI_range_new                    
Child            female Normal (18.5-25)           9    0
                        Obese (>30)                2    0
                        Underweight (<18.5)       55   23
                 male   Normal (18.5-25)           9    2
                        Obese (>30)                2    0
                        Overweight (25-30)         4    0
                        Underweight (<18.5)       53   22
Middle-Age Adult female Normal (18.5-25)         673    8
                        Obese (>30)              201   14
                        Overweight (25-30)       256    7
                        Underweight (<18.5)       92    0
                 male   Normal (18.5-25)         635    6
                        Obese (>30)              235    7
                        Overweight (25-30)       363    7
                        Underweight (<18.5)       34    0
Older Adult      female Normal (18.5-25)         286   19
                        Obese (>30)               94   22
                        Overweight (25-30)       184   25
                        Underweight (<18.5)       27    3
                 male   Normal (18.5-25)         287   16
                        Obese (>30)              100   28
                        Overweight (25-30)       230   27
                        Underweight (<18.5)       10    0
Oldest Adult     female Normal (18.5-25)          20    2
                        Obese (>30)               10    3
                        Overweight (25-30)        18    3
                        Underweight (<18.5)        1    1
                 male   Normal (18.5-25)          30    7
                        Obese (>30)                3    1
                        Overweight (25-30)        21   10
                        Underweight (<18.5)        1    0
Teenage          female Normal (18.5-25)          69    0
                        Normal/Overweight (<30)    8    0
                        Obese (>30)              177    0
                        Overweight (25-30)         8    0
                        Underweight (<18.5)       12    0
                 male   Normal (18.5-25)          55    0
                        Obese (>30)              136    0
                        Overweight (25-30)         9    0
                        Underweight (<18.5)       23    0
Toddler          female Normal (18.5-25)           1    0
                        Underweight (<18.5)        7    1
                 male   Normal (18.5-25)           2    5
                        Underweight (<18.5)        3    4
Unknown          female Normal (18.5-25)           2    0
                        Obese (>30)               82    0
                        Overweight (25-30)         2    0
                 male   Obese (>30)               41    0
Young adult      female Normal (18.5-25)         503    1
                        Normal/Overweight (<30)   24    0
                        Obese (>30)               23    0
                        Overweight (25-30)        53    0
                        Underweight (<18.5)       73    0
                 male   Normal (18.5-25)         556    1
                        Normal/Overweight (<30)   24    0
                        Obese (>30)               28    0
                        Overweight (25-30)       151    1
                        Underweight (<18.5)       20    0

---

## /. AGE + SEX + GI + ABX
```python
age_sex_GI_abx = (
    sub
    .groupby(["age_category_new", "sex", "GI_disease_history", "Antibiotics_used"])
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

Antibiotics_used                                                    No  Yes
age_category_new sex    GI_disease_history                                 
Child            female Crohn's disease                             42    0
                        cholera                                      9    0
                        indeterminate colitis                       17    0
                        schistosomiasis                              7    4
                        ulcerative colitis                          28    0
                 male   Crohn's disease                            104    0
                        cholera                                      7    0
                        schistosomiasis                              2    4
                        ulcerative colitis                          16    0
Middle-Age Adult female Clostridium difficile infection              5    0
                        Colitis;Intestinal_disease                   6    4
                        Colitis;Intestinal_disease;IBD               2    0
                        Crohn's disease                             78    0
                        Intestinal_disease                           8    4
                        Intestinal_disease;IBD                       2    0
                        cholera                                     15    0
                        ulcerative colitis                         120    0
                 male   Clostridium difficile infection              3    0
                        Colitis;Intestinal_disease                   6    1
                        Crohn's disease                             55    0
                        Intestinal_disease                           7    1
                        Intestinal_disease;Crohns_disease;IBD        0    1
                        Intestinal_disease;IBD                       1    0
                        cholera                                      6    0
                        ulcerative colitis                          42    0
Older Adult      female Clostridium difficile infection              5    0
                        Colitis;Intestinal_disease                   9    9
                        Colitis;Intestinal_disease;IBD               1    0
                        Crohn's disease                              8    0
                        Intestinal_disease                          13   12
                        Intestinal_disease;IBD                       2    1
                        ulcerative colitis                           8    0
                 male   Colitis;Intestinal_disease                  18   14
                        Colitis;Intestinal_disease;Crohns_disease    0    1
                        Colitis;Intestinal_disease;IBD               2    0
                        Intestinal_disease                           4    4
                        Intestinal_disease;IBD                       2    2
                        ulcerative colitis                          18    0
Oldest Adult     female Intestinal_disease                           0    2
                 male   Colitis;Intestinal_disease                   2    1
                        Intestinal_disease                           2    2
Teenage          female Crohn's disease                            117    0
                        helminthiasis                               75    0
                        ulcerative colitis                          45    0
                 male   Crohn's disease                            223    0
                        helminthiasis                               68    0
                        ulcerative colitis                          55    0
Toddler          female schistosomiasis                              1    0
Unknown          female Crohn's disease                              9    0
                        helminthiasis                               88    0
                 male   helminthiasis                               67    0
Young adult      female Crohn's disease                             60    0
                        cholera                                      7    0
                        ulcerative colitis                          59    0
                 male   Crohn's disease                             41    0
                        cholera                                      8    0
                        ulcerative colitis                          47    0

---

## ## 5. AGE + SEX + CA + ABX

```python
Cancers_and_adenomas

age_sex_CA_abx = (
    sub
    .groupby(["age_category_new", "sex", "Cancers_and_adenomas", "Antibiotics_used"])
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
Antibiotics_used                                                 No  Yes
age_category_new sex    Cancers_and_adenomas                            
Child            female Leukemia                                  2    0
                 male   Leukemia                                 14    0
Middle-Age Adult female adenoma                                   6    0
                        adenoma;familial adenomatous polyposis   11    0
                        adenoma;nonadvanced colorectal adenoma    9    0
                        colorectal cancer                        64    0
                        melanoma                                 95    0
                        non-small cell lung cancer                5    0
                 male   adenoma                                   6    0
                        adenoma;familial adenomatous polyposis    7    0
                        adenoma;nonadvanced colorectal adenoma   27    0
                        colorectal cancer                       114    0
                        melanoma                                126    6
                        non-small cell lung cancer               11    0
                        adenocarcinoma;colorectal cancer          7    0
Older Adult      female adenoma                                   4    0
                        adenoma;familial adenomatous polyposis    1    0
                        adenoma;nonadvanced colorectal adenoma   10    0
                        colorectal cancer                        60    0
                        melanoma                                 27    3
                        non-small cell lung cancer                3    0
                        adenocarcinoma;colorectal cancer          1    0
                        adenoma;advanced colorectal adenoma       1    0
                 male   adenoma                                   4    0
                        adenoma;familial adenomatous polyposis    3    0
                        adenoma;nonadvanced colorectal adenoma   21    0
                        colorectal cancer                        89    0
                        melanoma                                120    4
                        non-small cell lung cancer               10    0
                        adenocarcinoma;colorectal cancer         13    0
Oldest Adult     female colorectal cancer                         7    0
                        melanoma                                  6    1
                        adenocarcinoma;colorectal cancer          3    0
                 male   melanoma                                 44    7
                        adenocarcinoma;colorectal cancer          2    0
Teenage          female Leukemia                                  3    0
                 male   Leukemia                                  1    0
                        melanoma                                  1    0
Unknown          female melanoma                                  2    0
                        breast cancer                           118    0
Young adult      female colorectal cancer                         1    0
                        melanoma                                  1    0
                 male   colorectal cancer                         2    0
                        melanoma                                  1    0

---

## 9. Preparing the TSE object

```python
# 1. Set acc numbers as the rownames for colData
df = df.set_index("acc", drop=True)
df.index.is_unique

# 2. Replace empty strings with NA
df = df.replace("", pd.NA)

# 3. Drop list-like columns
df = df.drop(columns=["antibiotics_list", "name_of_antibiotic"], errors='ignore')

# 4. Convert categorical columns
categorical_cols = [
    "sex", "age_category", "Antibiotics_used", "UTI_history",
    "HIV_history", "COVID19_history", "CPE_infection_history",
    "Multidrug_resistant_infection", "Antivirals_used"
]
for c in categorical_cols:
    if c in df.columns:
        df[c] = df[c].astype("category")

# 4. Quick checks
print(df.index.is_unique)          # True
print(df.dtypes)                   # Check types
print(df.isna().sum().sort_values(ascending=False).head(10))  # Missing values
´´´

df = df.replace("", pd.NA)

```
Lets modify the antibiotic columns:

```

antibiotic_cols = [f'antibiotic_{i}' for i in range(1, 10)]

all_antibiotics = pd.Series(df[antibiotic_cols].values.ravel()).dropna().unique()

for ab in all_antibiotics:
    df[ab] = 'No'

for col in antibiotic_cols:
    for ab in df[col].dropna().unique():
        df.loc[df[col] == ab, ab] = 'Yes'

df = df.drop(columns=antibiotic_cols, errors='ignore')

df.to_csv("keskenab.tsv", sep="\t", index=True)
