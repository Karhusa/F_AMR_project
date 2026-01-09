
# Final fixes

## 1. Download the file

```{r}
port numpy as np
import pandas as pd
import re

df = pd.read_csv("kesken2.tsv", sep="\t")

```

## 2. Size and names of the columns
```
print(df.shape)
(24605, 37)

print(df.columns.tolist())

['acc', 'age_category', 'age_years', 'avgspotlen', 'bioproject', 'biosample', 'collection_date_sam', 'environmental_package', 'geo_loc_name_country_calc', 'geo_loc_name_country_continent_calc', 'instrument', 'location_resolution', 'mbases', 'platform', 'raw_metadata_Asthma', 'raw_metadata_Carbapenemase_bla_Gene', 'raw_metadata_Colitis', 'raw_metadata_Crohns_disease', 'raw_metadata_Drug_antivirus', 'raw_metadata_GI_infection', 'raw_metadata_Intestinal_disease', 'raw_metadata_MaternalAntimicrobials', 'raw_metadata_TotalAntimicrobialsDays', 'raw_metadata_age_group', 'raw_metadata_diseases', 'raw_metadata_host_disease', 'raw_metadata_multi_drug_resistant_organism_infection', 'raw_metadata_subject_disease_status_full', 'sex', 'spire_sample_name', 'subject_disease_status', 'subject_disease_status_full', 'BMI_range_new', 'antibiotics_list', 'antibiotic_1', 'antibiotic_2', 'antibiotic_3', 'antibiotic_4', 'antibiotic_5', 'antibiotic_6', 'antibiotic_7', 'antibiotic_8', 'antibiotic_9', 'name_of_antibiotic', 'Antibiotics_used', 'IBD', 'UTI_history']

```

## 3. Asthma related columns
```
df["raw_metadata_Asthma"].value_counts(dropna=False)

#raw_metadata_Asthma
#NaN    22874
#No      1729
#Yes        2

cols_with_asthma = [
    col for col in df.columns
    if df[col].astype(str).str.contains("Asthma", case=False, na=False).any()
]

cols_with_asthma

total_asthma = (
    df.astype(str)
      .apply(lambda col: col.str.contains("Asthma", case=False, na=False))
      .sum()
      .sum()
)

total_asthma
# np.int64(4)

df = df.drop(columns=["raw_metadata_Asthma"])
```
## 4. raw_metadata_Carbapenemase_bla_Gene
```
df["raw_metadata_Carbapenemase_bla_Gene"].value_counts(dropna=False)

#raw_metadata_Carbapenemase_bla_Gene
#NaN       24578
#KPC          17
#OXA-48        8
#NDM           2
```

## 5. Gastrointestinal tract diseases/infections columns
```
df["raw_metadata_Intestinal_disease"].value_counts(dropna=False)
df["raw_metadata_Colitis"].value_counts(dropna=False)
df["raw_metadata_Crohns_disease"].value_counts(dropna=False)
df["IBD"].value_counts(dropna=False)

df["GI_disease_history"] = pd.NA

intestinal_mask = (
    df["raw_metadata_Intestinal_disease"]
      .astype(str)
      .str.upper()
      .isin(["Y", "YES", "1"])
)

def add_gi_label(existing, new):
    if pd.isna(existing):
        return new
    if new in existing:
        return existing
    return f"{existing};{new}"

df.loc[intestinal_mask, "GI_disease_history"] = (
    df.loc[intestinal_mask, "GI_disease_history"]
      .apply(add_gi_label, new="Intestinal_disease")
)
colitis_mask = (
    df["raw_metadata_Colitis"]
      .astype(str)
      .str.upper()
      .isin(["Y", "YES", "1"])
)

df.loc[colitis_mask, "GI_disease_history"] = (
    df.loc[colitis_mask, "GI_disease_history"]
      .apply(add_gi_label, new="Colitis")
)

df["GI_disease_history"].value_counts(dropna=False)

#GI_disease_history
#<NA>                          24459
#Intestinal_disease;Colitis       76
#Intestinal_disease               70

crohns_mask = (
    df["raw_metadata_Crohns_disease"]
      .astype(str)
      .str.upper()
      .isin(["Y", "YES", "1"])
)

df.loc[crohns_mask, "GI_disease_history"] = (
    df.loc[crohns_mask, "GI_disease_history"]
      .apply(add_gi_label, new="Crohns_disease")
)
df["GI_disease_history"].value_counts(dropna=False)
#GI_disease_history
#<NA>                                         24459
#Intestinal_disease;Colitis                      75
#Intestinal_disease                              69
#Intestinal_disease;Crohns_disease                1
#Intestinal_disease;Colitis;Crohns_disease        1

ibd_mask = df["IBD"].str.upper() == "YES"

df.loc[ibd_mask, "GI_disease_history"] = (
    df.loc[ibd_mask, "GI_disease_history"]
      .apply(add_gi_label, new="IBD")
)

df["GI_disease_history"].value_counts(dropna=False)

#GI_disease_history
#<NA>                                         24459
#Intestinal_disease;Colitis                      70
#Intestinal_disease                              59
#Intestinal_disease;IBD                          10
#Intestinal_disease;Colitis;IBD                   5
#Intestinal_disease;Crohns_disease;IBD            1
#Intestinal_disease;Colitis;Crohns_disease        1


df = df.drop(columns=["raw_metadata_Intestinal_disease"])
df = df.drop(columns=["raw_metadata_GI_infection"])
df = df.drop(columns=["raw_metadata_Colitis"])
df = df.drop(columns=["raw_metadata_Crohns_disease"])
df = df.drop(columns=["IBD"])

df.to_csv("kesken3.tsv", sep="\t", index=False)
```

## 6. subject_disease_status_full column

### 6.1. inspect the values
```
df["subject_disease_status_full"].value_counts(dropna=False)

#subject_disease_status_full
#NaN                                                            23384
#melanoma                                                         197
#CTR                                                              180
#obesity                                                          166
#iCD                                                              161
#prediabetes                                                       97
#Non-alcoholic fatty liver disease                                 96
#PD                                                                93
#cCD                                                               28
#control                                                           28
#adenocarcinoma                                                    26
#hypertension                                                      23
#adenoma                                                           20
#IC                                                                17
#UC                                                                12
#pre-hypertension                                                  12
#oligohydramnios                                                   11
#Type 2 Diabetes, obesity                                          10
#mild preeclampsia; breech                                         10
#mild preeclampsia                                                 10
#preeclampsia                                                       9
#asthma                                                             4
#Control                                                            4
#Myalgic encephalomyelitis/chronic fatigue syndrome (ME/CFS)        3
#control patient                                                    2
#type 2 diabetes                                                    1
#N                                                                  1

# abbrevations (iCD, cCD, IC and UC) are not clear to interpret so we will search more).
#We may want to use cancers so I'll collect those. Including adenoma (pre-cancer).


```
6.2. Cancer related values
```
Cancer_keywords = [
    "melanoma",
    "adenocarcinoma",
    "adenoma",  
]
df["Cancers_and_adenomas"] = pd.NA

def add_label(existing, new):
    if pd.isna(existing):
        return new
    if new in existing:
        return existing
    return f"{existing};{new}"

for keyword in cancer_keywords:
    mask = df["subject_disease_status_full"].astype(str).str.contains(keyword, case=False, na=False)
    df.loc[mask, "Cancers_and_adenomas"] = df.loc[mask, "Cancers_and_adenomas"].apply(add_label, new=keyword)

df["Cancers_and_adenomas"].value_counts(dropna=False)

```
6.2. Abbrevations
```
Diseases_of_interest = ["iCD", "cCD", "IC", "UC", "PD"]

subset = df[df["subject_disease_status_full"].isin(Diseases_of_interest)]

bioprojects = subset["bioproject"].unique()
print("Unique bioproject numbers for iCD, cCD, IC, UC, PD:")
print(bioprojects)

print(bioprojects)
['PRJNA237362' 'PRJDB15956']

## PRJDB15956 is about parkinsos disease (PD)



```
## 7. subject_disease_status column`df["subject_disease_status"].value_counts(dropna=False)

```

df["subject_disease_status"].value_counts(dropna=False)u
bject_disease_status
#COHORT                                                      10713
#CTR                                                          3651
#NaN                                                          3096
#Crohn's disease                                               936
#Parkinson's disease                                           721
#ulcerative colitis                                            640
#colorectal cancer                                             456
#melanoma                                                      444
#control patient                                               402
#helminthiasis                                                 298
#COVID-19                                                      294
#Clostridium difficile infection                               292
#postmenopausal osteopenia                                     160
#gestational diabetes mellitus                                 133
#type 2 diabetes                                               131
#critically ill patient                                        124
#end-stage renal disease                                       124
#HIV                                                           120
#metabolic syndrome                                            119
#breast cancer                                                 118
#prediabetes                                                    97
#ankylosing spondylitis                                         91
#cholera                                                        89
#metabolic dysfunction-associated steatohepatitis               87
#liver cirrhosis                                                87
#irritable bowel syndrome                                       77
#atopy                                                          73
#postmenopausal osteoporosis                                    72
#nonadvanced colorectal adenoma                                 67
#urinary tract infection                                        67
#autism spectrum disorder                                       66
#konzo                                                          60
#diarrhea                                                       52
#Vogt-Koyanagi-Harada disease                                   51
#polycystic ovary syndrome                                      50
#slow transit constipation                                      48
#chronic kidney disease                                         45
#type 1 diabetes                                                44
#metastatic renal cell carcinoma                                43
#hypertension                                                   34
#impaired glucose tolerance                                     33
#rheumatoid arthritis                                           32
#indeterminate colitis                                          29
#metabolic dysfunction-associated steatotic liver disease       29
#non-small cell lung cancer                                     29
#adenoma                                                        27
#Carbapenemase-producing Enterobacteriaceae infection           27
#chronic obstructive pulmonary disease                          27
#familial adenomatous polyposis                                 22
#Behcet's disease                                               21
#schistosomiasis                                                18
#pre-hypertension                                               12
#stunted growth                                                 12
#short bowel syndrome                                           11
#myalgic encephalomyelitis/chronic fatigue syndrome              3
#advanced colorectal adenoma                                     1
#Name: count, dtype: int64

```
### 7.1. Cancer related values 
```
new_cancer_keywords = [
    "colorectal cancer",
    "melanoma",
    "breast cancer",
    "metastatic renal cell carcinoma",
    "non-small cell lung cancer",
    "adenoma",
    "nonadvanced colorectal adenoma",
    "advanced colorectal adenoma",
    "familial adenomatous polyposis"
]

def add_label(existing, new):
    if pd.isna(existing):
        return new
    if new in existing:
        return existing
    return f"{existing};{new}"

for keyword in new_cancer_keywords:
    mask = df["subject_disease_status"].astype(str).str.contains(keyword, case=False, na=False)
    df.loc[mask, "Cancers_and_adenomas"] = df.loc[mask, "Cancers_and_adenomas"].apply(add_label, new=keyword)

df["Cancers_and_adenomas"].value_counts(dropna=False)

#Cancers_and_adenomas
#<NA>                                      23398
#melanoma                                    444
#colorectal cancer                           430
#breast cancer                               118
#adenoma;nonadvanced colorectal adenoma       67
#metastatic renal cell carcinoma              43
#non-small cell lung cancer                   29
#adenoma                                      27
#adenocarcinoma;colorectal cancer             26
#adenoma;familial adenomatous polyposis       22
#adenoma;advanced colorectal adenoma           1
```
### 7.2. GI-tract related values 

```
gi_inflam_infect_keywords = [
    "Crohn's disease",
    "ulcerative colitis",
    "indeterminate colitis",
    "Clostridium difficile infection",
    "cholera",
    "schistosomiasis",
    "helminthiasis"
]
def add_label(existing, new):
    if pd.isna(existing):
        return new
    if new in existing:
        return existing
    return f"{existing};{new}"

for keyword in gi_inflam_infect_keywords:
    mask = df["subject_disease_status"].astype(str).str.contains(keyword, case=False, na=False)
    df.loc[mask, "GI_disease_history"] = df.loc[mask, "GI_disease_history"].apply(add_label, new=keyword)

df["GI_disease_history"].value_counts(dropna=False)
GI_disease_history
#<NA>                                         22157
#Crohn's disease                                936
#ulcerative colitis                             640
#helminthiasis                                  298
#Clostridium difficile infection                292
#cholera                                         89
#Intestinal_disease;Colitis                      70
#Intestinal_disease                              59
#indeterminate colitis                           29
#schistosomiasis                                 18
#Intestinal_disease;IBD                          10
#Intestinal_disease;Colitis;IBD                   5
#Intestinal_disease;Crohns_disease;IBD            1
#Intestinal_disease;Colitis;Crohns_disease        1

```
### 7.3. UTI related values
```
uti_mask = df["subject_disease_status"].astype(str).str.contains("urinary tract infection", case=False, na=False)

df.loc[uti_mask, "UTI_history"] = "Yes"

df["UTI_history"].value_counts(dropna=False)

#UTI_history
#No     24355
#Yes      250
```

## 7.4 Other viral/bacterial related diseases
```
df["HIV_history"] = pd.NA
df["COVID19_history"] = pd.NA
df["CPE_infection_history"] = pd.NA  # CPE = Carbapenemase-producing Enterobacteriaceae

# HIV
df.loc[df["subject_disease_status"].astype(str).str.contains("HIV", case=False, na=False), "HIV_history"] = "Yes"

# COVID-19
df.loc[df["subject_disease_status"].astype(str).str.contains("COVID-19", case=False, na=False), "COVID19_history"] = "Yes"

# CPE infection
df.loc[df["subject_disease_status"].astype(str).str.contains("Carbapenemase-producing Enterobacteriaceae infection", case=False, na=False), "CPE_infection_history"] = "Yes"

for col in ["HIV_history", "COVID19_history", "CPE_infection_history"]:
    df[col] = df[col].fillna("No").astype("category")
```

### 7.5. Remove original column and save
```
df = df.drop(columns=["subject_disease_status"])
df.to_csv("kesken4.tsv", sep="\t", index=False)
```
## 8 raw_metadata_subject_disease_status_full column
```
df["raw_metadata_subject_disease_status_full"].value_counts(dropna=False)

#raw_metadata_subject_disease_status_full
#NaN                              24314
#COHORT                             158
#gestational diabetes mellitus      133
#Name: count, dtype: int64

df = df.drop(columns=["raw_metadata_subject_disease_status_full"])
```
## 9. raw_metadata_host_disease column
```
df["raw_metadata_host_disease"].value_counts(dropna=False)

df["raw_metadata_host_disease"].value_counts(dropna=False)
raw_metadata_host_disease
NaN                             24585
Acute Lymphoblastic Leukemia       14
Acute Myeloid Leukemia              6
Name: count, dtype: int64

mask = df["raw_metadata_host_disease"].notna()

df.loc[mask, "Cancers_and_adenomas"] = (
    df.loc[mask, "Cancers_and_adenomas"]
    .apply(add_label, new="Leukemia")
)

df = df.drop(columns=["raw_metadata_host_disease"])

df.to_csv("kesken4.tsv", sep="\t", index=False)
```
## 10. raw_metadata_diseases column
```
df["raw_metadata_diseases"].value_counts(dropna=False)

#raw_metadata_diseases
#NaN                        24593
#NEC                            7
#cellulitis,MRSA                1
#acinetobacter anitratus        1
#sepsis                         1
#cellulitis                     1
#bradycardia                    1

df = df.drop(columns=["raw_metadata_diseases"])
```

## 11. raw_metadata_multi_drug_resistant_organism_infection column
```
df["raw_metadata_multi_drug_resistant_organism_infection"].value_counts(dropna=False)

#raw_metadata_multi_drug_resistant_organism_infection
#NaN         24479
#Positive       88
#Negative       38
#Name: count, dtype: int64

df["Multidrug_resistant_infection"] = pd.NA

df.loc[
    df["raw_metadata_multi_drug_resistant_organism_infection"] == "Positive",
    "Multidrug_resistant_infection"
] = "Yes"

df.loc[
    df["raw_metadata_multi_drug_resistant_organism_infection"] == "Negative",
    "Multidrug_resistant_infection"
] = "No"

df["Multidrug_resistant_infection"] = df["Multidrug_resistant_infection"].astype("category")
#Multidrug_resistant_infection
#NaN    24479
#Yes       88
#No        38
#Name: count, dtype: int64

df = df.drop(columns=["raw_metadata_multi_drug_resistant_organism_infection"])

```
## 12. raw_metadata_Drug_antivirus, raw_metadata_MaternalAntimicrobials, and raw_metadata_TotalAntimicrobialsDays columns
```
df["raw_metadata_Drug_antivirus"].value_counts(dropna=False)

#raw_metadata_Drug_antivirus
#NaN    24591
#Y         14

df["Antivirals_used"] = df["raw_metadata_Drug_antivirus"].apply(lambda x: "Yes" if x == "Y" else "No")

df["Antivirals_used"].value_counts(dropna=False)

#Antivirals_used
#No     24591
#Yes       14


df["raw_metadata_MaternalAntimicrobials"].value_counts(dropna=False)

# raw_metadata_MaternalAntimicrobials
# NaN                                  24243
# cefazolin (Ancef)                      235
# PCN G                                   71
# clindamycin                             19
# ampicillin                              17
# amp-sulbactam (Unasyn)                   8
# amoxicillin                              7
# Retrovirals for HIV  (AZT, HAART)        5

df["raw_metadata_TotalAntimicrobialsDays"].value_counts(dropna=False)

subset = df[df["raw_metadata_MaternalAntimicrobials"].notna() | df["raw_metadata_TotalAntimicrobialsDays"].notna()]

unique_projects = subset["bioproject"].unique()

print(unique_projects)
['PRJNA489090']

print("Number of unique bioprojects:", len(unique_projects))
Number of unique bioprojects: 1

## -> Both columns (raw_metadata_MaternalAntimicrobials and raw_metadata_TotalAntimicrobialsDays) are from the same study so most likely maternal antiviral related.

df = df.drop(columns=["raw_metadata_Drug_antivirus"])
df = df.drop(columns=["raw_metadata_MaternalAntimicrobials"])
df = df.drop(columns=["raw_metadata_TotalAntimicrobialsDays"])

```
## 13. raw_metadata_age_group column
```
["raw_metadata_age_group"].value_counts(dropna=False)

raw_metadata_age_group
NaN          24600
adult            3
schoolage        2
Name: count, dtype: int64

df["age_category"] = df["raw_metadata_age_group"].replace({
    "adult": "Adult",
    "schoolage": "Child"
})


df = df.drop(columns=["raw_metadata_age_group"])
```


```
DataFrame shape: (24605, 39)
```

df.to_csv("kesken4.tsv", sep="\t", index=False)

