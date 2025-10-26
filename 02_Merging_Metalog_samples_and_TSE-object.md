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

# filter biosample numbers
grep -oE '\bSAM(N|D|EA)[0-9]+' SRA_metadata_with_biosample_corrected.txt | sort -u > sra_biosample_ids.txt

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
## 4. Make a new file from compressed human_all_wide_2025-10-19.tsv.gz

```
gzcat human_all_wide_2025-10-19.tsv.gz | head -n 1 > human_subset.tsv

```

## 4. Make a subset of matching ID's from human_all_wide_2025-10-19.tsv.gz and SRA_metadata_with_biosample_corrected.txt

```
python

python3

import pandas as pd

# 1: Load list of matching BioSample IDs
matched_ids = pd.read_csv("matched_biosamples.txt", header=None, names=["biosample"], dtype=str)
matched_set = set(matched_ids["biosample"])

# Step 2:  Human_all_wide_2025-10-19.tsv.gz
human_file = "human_all_wide_2025-10-19.tsv.gz"
human_subset_file = "human_subset.tsv"

#smaller parts because the file is large
chunksize = 10000
first_chunk = True

for chunk in pd.read_csv(human_file, sep="\t", dtype=str, chunksize=chunksize):
    # Column 3 contains the biosample ID
    col_name = chunk.columns[2] 
    filtered = chunk[chunk[col_name].isin(matched_set)]
    
    filtered.to_csv(human_subset_file, sep="\t", index=False,
                    mode='w' if first_chunk else 'a', header=first_chunk)
    first_chunk = False

print(f"Human table subset saved to: {human_subset_file}")

# 3: Subset SRA_metadata_with_biosample_corrected.txt

sra_file = "SRA_metadata_with_biosample_corrected.txt"
sra_subset_file = "SRA_subset.csv"

sra = pd.read_csv(sra_file, sep=",", dtype=str)
sra_filtered = sra[sra['biosample'].isin(matched_set)]
sra_filtered.to_csv(sra_subset_file, index=False)

print(f"SRA metadata subset saved to: {sra_subset_file}")


print(f"Human subset rows: {len(pd.read_csv(human_subset_file, sep='\t')) - 1}")  # minus header
print(f"SRA subset rows: {len(pd.read_csv(sra_subset_file)) - 1}")
exit()

```

```
import pandas as pd

sra_subset_file = "SRA_subset.csv"
sra = pd.read_csv(sra_subset_file, dtype=str)

# Check number of rows vs unique BioSample IDs
total_rows = sra.shape[0]
unique_ids = sra['biosample'].nunique()

print(f"Total rows: {total_rows}")
print(f"Unique BioSample IDs: {unique_ids}")

if total_rows == unique_ids:
    print("All BioSample IDs are unique.")
else:
    print(f"There are {total_rows - unique_ids} duplicate BioSample ID rows.")

```
#Total rows: 24605
#Unique BioSample IDs: 20339


Merge subsetted tables:

```

import pandas as pd

human_file = "human_all_wide_2025-10-19.tsv.gz"
sra_file = "SRA_metadata_with_biosample_corrected.txt"
matched_file = "matched_biosamples.txt"

human_subset_file = "human_subset.tsv"
sra_subset_file = "SRA_subset.csv"
sra_unique_file = "SRA_subset_unique.csv"
merged_file = "merged_subset_unique.tsv"

chunksize = 10000  # adjust if you have lots of memory

# Load matched BioSample IDs
matched_ids = pd.read_csv(matched_file, header=None, names=["biosample"], dtype=str)
matched_set = set(matched_ids["biosample"])
print(f"Loaded {len(matched_set)} matched BioSample IDs.")

# Subset human table in chunks
print("Subsetting human table...")
first_chunk = True
for chunk in pd.read_csv(human_file, sep="\t", dtype=str, chunksize=chunksize):
    col_name = chunk.columns[2]  # column 3 = sample ID
    filtered = chunk[chunk[col_name].isin(matched_set)]
    
    filtered.to_csv(human_subset_file, sep="\t", index=False,
                    mode='w' if first_chunk else 'a', header=first_chunk)
    first_chunk = False

human_rows = sum(1 for _ in open(human_subset_file)) - 1
print(f"Human subset saved to '{human_subset_file}' ({human_rows} rows).")

# Subset and deduplicate SRA metadata
print("Subsetting SRA metadata...")
sra = pd.read_csv(sra_file, sep=",", dtype=str)
sra_filtered = sra[sra['biosample'].isin(matched_set)]

print(f"SRA subset: {sra_filtered.shape[0]} rows before deduplication.")
sra_unique = sra_filtered.drop_duplicates(subset=['biosample'])
print(f"SRA subset: {sra_unique.shape[0]} unique BioSamples after deduplication.")

sra_filtered.to_csv(sra_subset_file, index=False)
sra_unique.to_csv(sra_unique_file, index=False)
print(f"Saved full SRA subset to '{sra_subset_file}' and unique subset to '{sra_unique_file}'.")

# Merge human table and unique SRA subset
print("Merging human and SRA subsets (unique BioSamples only)...")
human = pd.read_csv(human_subset_file, sep="\t", dtype=str)
merged = pd.merge(human, sra_unique, left_on=human.columns[2], right_on='biosample', how='inner')

merged.to_csv(merged_file, sep="\t", index=False)
print(f"Merged table saved to '{merged_file}' ({merged.shape[0]} rows, {merged.shape[1]} columns).")

print("Done!")

```

