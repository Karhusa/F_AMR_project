## 1. Download and check SRA_metadata_with_biosample_corrected.txt

```
head -n 1 SRA_metadata_with_biosample_corrected.txt | tr ',' '\n' | nl
     1  acc
     2  biosample
     3  geo_loc_name_country_calc
     4  geo_loc_name_country_continent_calc
     5  platform
     6  instrument
     7  bioproject
     8  avgspotlen
     9  mbases
    10  collection_date_sam

grep -oE '\bSAM(N|D|EA)[0-9]+' SRA_metadata_with_biosample_corrected.txt | sort -u > sra_biosample_ids.txt

wc -l sra_biosample_ids.txt
# count: 54178 sra_biosample_ids.txt
```

## 2. Download and check human_all_wide_2025-10-19.tsv.gz

```
bash
gzcat human_all_wide_2025-10-19.tsv.gz | wc -l
# rows 85471
gzcat human_all_wide_2025-10-19.tsv.gz | head -n 1 | awk -F'\t' '{print NF}'
# columns: 2836

# Does this file have sample id numbers?
gzcat human_all_wide_2025-10-19.tsv.gz | grep -E '\bSAM(N|D|EA)[0-9]+' | wc -l
# yes: 81436

#Extract the sample ids to a separate list:
gzcat human_all_wide_2025-10-19.tsv.gz | grep -oE '\bSAM(N|D|EA)[0-9]+' | sort -u > biosample_ids.txt

```

## 3. Find matching sample ID's from human_all_wide_2025-10-19.tsv.gz and SRA_metadata_with_biosample_corrected.txt
```
bash

sort biosample_ids.txt > biosample_ids_sorted.txt
sort sra_biosample_ids.txt > sra_biosample_ids_sorted.txt

comm -12 biosample_ids_sorted.txt sra_biosample_ids_sorted.txt > matched_biosamples.txt

wc -l matched_biosamples.txt
# matches: 20339
```
## 4. Workflow for merging SRA_metadata_with_biosample_corrected.txt and human_all_wide_2025-10-19.tsv.gz by the sample id's

```
python

import pandas as pd
import re

# Set tables

human_file = "human_all_wide_2025-10-19.tsv.gz"
sra_file = "SRA_metadata_with_biosample_corrected.txt"
matched_file = "matched_biosamples.txt"
final_output = "cleaned_merged_final.tsv"

id_col_human = "spire_sample_name"
id_col_sra = "biosample"

chunksize = 20000  # safe for large files

keywords = [
    "bmi", "body mass", "antibiotic", "antimicrobial", "drug",
    "sex", "gender", "age", "disease", "diagnosis", "condition",
    "infection", "immune", "inflammation", "therapy", "treatment",
    "cycline", "cillin", "phenicol", "bactam", "beta", "lactamase", "inhibitor",
    "cef", "loracarbef", "flomoxef", "latamoxef", "onam", "penem", "cilastatin",
    "cephalexin", "dazole", "mycin", "prim", "sulfa", "tylosin", "tilmicosin",
    "tylvacosin", "tildipirosin", "macrolide", "quinupristin", "dalfopristin",
    "xacin", "acid", "flumequine", "olaquindox", "sulfonamide", "tetracycline",
    "nitrofuran", "amphenicol", "polymyxin", "quinolone",
    "aminoglycoside", "teicoplanin", "vancin", "colistin", "polymyxin b",
    "nitro", "fura", "nifru", "micin", "tiamulin", "valnemulin",
    "xibornol", "clofoctol", "methenamine", "zolid", "lefamulins",
    "gepotidacin", "bacitracin", "novobiocin",
    "asthma", "cancer", "uti", "ibd", "crohn", "intestine",
    "ulcerative", "colitis"
]
keywords = [kw.lower() for kw in keywords]

exclude_keywords = [
    "sexual", "private", "identifier", "confidential", "contact",
    "gestational", "beverages", "stage", "vitamin", "alcohol",
    "average", "home", "sports", "strength", "siblings", "blockage"
]

# 1. Load matched BioSamples

matched_biosamples = pd.read_csv(
    matched_file, header=None, names=[id_col_sra], dtype=str
)
matched_set = set(matched_biosamples[id_col_sra])
print(f"Matched BioSamples: {len(matched_set)}")
# 20339

# 2. Subset human metadata by chunk
human_subset_file = "tmp_human_subset.tsv"
first = True

for chunk in pd.read_csv(human_file, sep="\t", dtype=str, chunksize=chunksize):
    filtered = chunk[chunk[id_col_human].isin(matched_set)]
    filtered.to_csv(
        human_subset_file,
        sep="\t",
        index=False,
        mode="w" if first else "a",
        header=first
    )
    first = False

human = pd.read_csv(human_subset_file, sep="\t", dtype=str)
print(f"Human subset: {human.shape}")
# (20339, 2836)

# 3. Subset SRA metadata
sra = pd.read_csv(sra_file, sep=",", dtype=str)
sra = sra[sra[id_col_sra].isin(matched_set)]
print(f"SRA subset: {sra.shape}")
# (24605, 10)

# 4. Merge
merged = pd.merge(
    human, sra,
    left_on=id_col_human,
    right_on=id_col_sra,
    how="inner",
    validate="many_to_many"
)
print(f"Merged: {merged.shape}")
# (24605, 2846)

# 5. Column filtering
def keep_keyword(c):
    c_low = c.lower()
    return any(kw in c_low for kw in keywords)

def drop_excluded(c):
    c_low = c.lower()
    return any(bad in c_low for bad in exclude_keywords)

always_keep = set(sra.columns.tolist() + [id_col_human])
keyword_keep = {c for c in merged.columns if keep_keyword(c)}

cols_to_keep = always_keep | keyword_keep
cols_to_keep = sorted(
    c for c in cols_to_keep
    if c in merged.columns and not drop_excluded(c)
)

merged_clean = merged[cols_to_keep]
print(f"Final filtered columns: {len(cols_to_keep)}")
# 344

# 6. Save final file
merged_clean.to_csv(final_output, sep="\t", index=False)
```
