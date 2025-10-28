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
## 4. Make a new file from compressed human_all_wide_2025-10-19.tsv.gz

```
gzcat human_all_wide_2025-10-19.tsv.gz | head -n 1 > human_subset.tsv

```

## 4. Make a subset of matching ID's from human_all_wide_2025-10-19.tsv.gz and SRA_metadata_with_biosample_corrected.txt

```
import pandas as pd

# Load list of matching BioSample IDs
matched_ids = pd.read_csv("matched_biosamples.txt", header=None, names=["biosample"], dtype=str)
matched_set = set(matched_ids["biosample"])

# Process the large human file in chunks
human_file = "human_all_wide_2025-10-19.tsv.gz"
human_subset_file = "human_subset.tsv"

chunksize = 10000
first_chunk = True

for chunk in pd.read_csv(human_file, sep="\t", dtype=str, chunksize=chunksize):
    col_name = chunk.columns[2]   # the 3rd column holds the biosample ID
    filtered = chunk[chunk[col_name].isin(matched_set)]
    
    filtered.to_csv(
        human_subset_file,
        sep="\t",
        index=False,
        mode='w' if first_chunk else 'a',
        header=first_chunk
    )
    first_chunk = False

print(f" Human table subset saved to: {human_subset_file}")

# Subset the SRA metadata file
sra_file = "SRA_metadata_with_biosample_corrected.txt"
sra_subset_file = "SRA_subset.csv"

sra = pd.read_csv(sra_file, sep=",", dtype=str)
sra_filtered = sra[sra['biosample'].isin(matched_set)]
sra_filtered.to_csv(sra_subset_file, index=False)

print(f" SRA metadata subset saved to: {sra_subset_file}")

# Check for sample id duplicates in both files

# Read the human subset
human_df = pd.read_csv(human_subset_file, sep="\t", dtype=str)
human_id_col = human_df.columns[2]  # 3rd column = biosample ID

# Count total and unique IDs
total_human_rows = len(human_df)
unique_human_ids = human_df[human_id_col].nunique()
duplicate_human_rows = total_human_rows - unique_human_ids

print(f"Human subset total rows: {total_human_rows}")
print(f"Human subset unique BioSample IDs: {unique_human_ids}")
print(f"Human subset duplicate rows: {duplicate_human_rows}")

# Do the same for SRA
total_sra_rows = len(sra_filtered)
unique_sra_ids = sra_filtered["biosample"].nunique()
duplicate_sra_rows = total_sra_rows - unique_sra_ids

print(f"SRA subset total rows: {total_sra_rows}")
print(f"SRA subset unique BioSample IDs: {unique_sra_ids}")
print(f"SRA subset duplicate rows: {duplicate_sra_rows}")

print("\n Duplicate check complete.")

```

## 6. Merge subsetted tables:

```
import pandas as pd

# Load SRA metadata subset (keep all columns)
sra = pd.read_csv("SRA_subset.csv", dtype=str)

# Load human data subset (already filtered)
human = pd.read_csv("human_subset.tsv", sep="\t", dtype=str)

# The shared ID column:
sra_id_col = "biosample"
human_id_col = human.columns[2]  # Third column holds the biosample ID

print(f"Human sample ID column: {human_id_col}") #check the right column

# Merge (many-to-many merge to keep duplicates)
merged = pd.merge(
    human,
    sra,
    left_on=human_id_col,
    right_on=sra_id_col,
    how="inner",        # keep only matched IDs
    validate="many_to_many"  # allow duplicates in both
)

# Save the merged file
merged_file = "merged_full_with_duplicates.tsv"
merged.to_csv(merged_file, sep="\t", index=False)

print(f"Merged file saved to: {merged_file}")
print(f"Total rows in merged file: {len(merged)}")
print(f"Total columns in merged file: {len(merged.columns)}")

