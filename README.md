# Master's thesis: AMR project female

Saana Karhula 29.10.2025 - 22.11.2025

## Table of contents

1.  **Main**
2.  **Look through TSE-object and compare it to METALOG csv file**
    1.  Main
    2.  Quality control


## 2. Look through TSE-object and compare it to METALOG csv file

We need to find matching ACC-numbers from TSE-object and Meatlog file

### 2.1. Main

Download already existing TSE-object to your local computer and open it with R and check what kind of sample or ACC identifies data includes. We are interested in column number1 named "acc", it includes Accession numbers of samples.

Download the csv file from [Metalog - human samples](https://metalog.embl.de/explore/human) and open with unix (too large for R). File includes three columns: study code, ena_ers_sample_id and sample alias. We are interested in values in column 2. Save unique prefixes from column 2 to a textfile to see what it icludes.

Make new lists based on the unique sample prefixes (SRR, ERR, SAMN and so on). Go through the lists and see if those have any matches to the TSE object

[Codes for 2.1.](https://github.com/Karhusa/F_AMR_project/blob/main/00_Human_sample_list.csv_lookthrough)

### 2.2. QC

3.10.2025 All of the searches returned empty matches. This was also looked through with with hands on (looked through the files).

Now next we need to find ACC numbers for sample ids (f.ex. SAMN), because TSE-object includes only ACC-numbers.




