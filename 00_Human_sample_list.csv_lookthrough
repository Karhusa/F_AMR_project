### 1. Download and check TSE-object

Download already existing TSE-object to your local computer and open it with R and check what kind of sample or ACC identifies data includes

```{r}
library(tibble)

TSE_filtered <- readRDS("~/Downloads/TSE_filtered.rds")
df_col <- as_tibble(colData(TSE_filtered))
View(df_col)

```

We are interested in column number1 named "acc", it includes Accession numbers of samples.

### 2. Download sample list from Metalog and check it for acc-numbers 

Download the list from [Metalog- human samples](https://metalog.embl.de/explore/human) and open with unix (too large for R)

File includes three columns: study code, ena_ers_sample_id and sample alias. We are interested in column 2. Save unique prefixes from column 2 to a textfile to see what it icludes

```
bash

# Save unique values from colum 1 to a textfile
cut -d',' -f1 human_sample_list.csv | sort -u > studies.txt

# Save unique prefixes from column 2 to a textfile
tail -n +2 human_sample_list.csv | cut -d',' -f2 | sed 's/[0-9]*//g' | sort -u > unique_sample_names.tx
#ERR
#ERS
#SAMD
#SAMEA
#SAMN
#SRR
#SRS
#SRX

# make different tables with different sample number prefixes (smaller and easier to run)
awk -F',' 'NR==1 || $2 ~ /^ERR/ {print}' human_sample_list.csv > err_ids.csv
awk -F',' 'NR==1 || $2 ~ /^ERS/ {print}' human_sample_list.csv > ers_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SAMD/ {print}' human_sample_list.csv > samd_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SAMEA/ {print}' human_sample_list.csv > samea_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SAMN/ {print}' human_sample_list.csv > samn_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SRR/ {print}' human_sample_list.csv > srr_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SRS/ {print}' human_sample_list.csv > srs_ids.csv
awk -F',' 'NR==1 || $2 ~ /^SRX/{print}' human_sample_list.csv > srx_ids.csv
````
download to R

````
{r}

library(readr)

err_ids <- read_csv("Gradu_AMR/err_ids.csv")
ers_ids <- read_csv("Gradu_AMR/ers_ids.csv")
samd_ids <- read_csv("Gradu_AMR/samd_ids.csv")
samea_ids <- read_csv("Gradu_AMR/samea_ids.csv")
samn_ids <- read_csv("Gradu_AMR/samn_ids.csv")
srr_ids <- read_csv("Gradu_AMR/srr_ids.csv")
srs_ids <- read_csv("Gradu_AMR/srs_ids.csv")
srx_ids <- read_csv("Gradu_AMR/srx_ids.csv")


matches_err <- err_ids$ena_ers_sample_id[err_ids$ena_ers_sample_id %in% df_col$acc]
matches_ers <- ers_ids$ena_ers_sample_id[ers_ids$ena_ers_sample_id %in% df_col$acc]
matches_samd <- samd_ids$ena_ers_sample_id[samd_ids$ena_ers_sample_id %in% df_col$acc]
matches_samea <- samea_ids$ena_ers_sample_id[samea_ids$ena_ers_sample_id %in% df_col$acc]
matches_samn <- samn_ids$ena_ers_sample_id[samn_ids$ena_ers_sample_id %in% df_col$acc]
matches_srr <- srr_ids$ena_ers_sample_id[srr_ids$ena_ers_sample_id %in% df_col$acc]
matches_srs<- srs_ids$ena_ers_sample_id[srs_ids$ena_ers_sample_id %in% df_col$acc]
matches_srx <- srx_ids$ena_ers_sample_id[srx_ids$ena_ers_sample_id %in% df_col$acc]

````

All of the searches returned empty matches. This was also looked through with with hands on (looked through the files)
