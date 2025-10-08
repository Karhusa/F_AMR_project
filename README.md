# Master's thesis: AMR project female

Saana Karhula 29.10.2025 - 22.11.2025

## Table of contents

1.  **Main**
2.  **Look through TSE-object and compare it to METALOG csv file**
    1.  Main
    2.  Quality control


## 2. Look through TSE-object and compare it to METALOG csv file

The objective of this step is to identify matching accession numbers (ACC) between the TSE object and the METALOG dataset.

### 2.1. Main

**Prepare the TSE Object**

Download the existing TSE object to your local machine. Open it in R and inspect the data structure to determine what types of sample or accession identifiers it contains.
The primary column of interest is column 1 ("acc"), which includes the accession numbers corresponding to the samples.

**Obtain and Inspect the METALOG File**

Download the CSV file from Metalog – Human Samples. Since the file is too large to efficiently open in R, examine it using Unix-based tools.
The file includes three columns:
1. Study code
2. ena_ers_sample_id
3. Sample alias

**Extract and Analyze Identifiers**

Focus on column 2 (ena_ers_sample_id), which contains sample identifiers. Save the unique prefixes from this column (e.g., SRR, ERR, SAMN) to a text file to review the types of identifiers included. Create separate lists for each prefix type and compare these lists to the accession numbers in the TSE object.

**Results**

Initially, no matches were found between the accession numbers in the TSE object and those in the METALOG file.

A new table, SRA_metadata_with_biosample_numbers, was then obtained, which included both SRA and BioSample identifiers. Using this updated table, matching entries were successfully identified within the TSE object.


[Codes for 2.1.](https://github.com/Karhusa/F_AMR_project/blob/main/00_Human_sample_list.csv_lookthrough.md)

### 2.2. QC

3.10.2025 All of the searches returned empty matches. This was also looked through with with hands on (looked through the files).
8.10.2025 Matches found for the sampleid numbers

Now next we need to find ACC numbers for sample ids (f.ex. SAMN), because TSE-object includes only ACC-numbers.




