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

```
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

```

Lets try another approach, because the prefixes ERR and SRR can be mixed and only the series of numbers behind prefix matters. df_col in TSE-object also includes prefix DRR which need to taken into account.

So lets filter out the numbers behind prefixes and compare those to TSE-object.

```
df_filtered <- df_col$acc[grepl("^(SRR|ERR|DRR)", df_col$acc)]

err_nums <- sub("^ERR", "", err_ids$ena_ers_sample_id)
srr_nums <- sub("^SRR", "", srr_ids$ena_ers_sample_id)
df_nums  <- sub("^[A-Z]+", "", df_filtered)

err_df <- data.frame(type = "ERR", id = err_ids$ena_ers_sample_id, num = err_nums)
srr_df <- data.frame(type = "SRR", id = srr_ids$ena_ers_sample_id, num = srr_nums)
df_ref <- data.frame(type = "df_col", id = df_filtered, num = df_nums)

combined <- rbind(err_df, srr_df)
matches <- merge(combined, df_ref, by = "num", suffixes = c("_query", "_df"))

matches

```
Unfortunately no matches were found. This was also visualised and checked by hand.

All of the searches above returned empty matches. This was also looked through with with hands on (looked through the files). So there were no matches. Now we need to look more closely the sample ids. Katariina was able to get a new SRA file (TSE-objest) with sample id numbers (how and where)

Lets compare the sample id numbers to the new SRA file

```
{r}

SRA_metadata_with_biosample <- read.csv("~/F_AMR_project/Gradu_AMR/SRA_metadata_with_biosample.txt")
View(SRA_metadata_with_biosample)

matches_samd <- samd_ids$ena_ers_sample_id[samd_ids$ena_ers_sample_id %in% SRA_metadata_with_biosample$biosample]
length(matches_samd)
# --> 27

matches_samea <- samea_ids$ena_ers_sample_id[samea_ids$ena_ers_sample_id %in% SRA_metadata_with_biosample$biosample]
length(matches_samea)
# --> 1617

matches_samn <- samn_ids$ena_ers_sample_id[samn_ids$ena_ers_sample_id %in% SRA_metadata_with_biosample$biosample]
length(matches_samea)
# --> 1976

# total: 3620
```

There were matches.

Make a new column "Metalog" to SRA-table and add a "1" to the column if there was a match between the SRA table and metalog sample ids (all_matches). Otherwise add 0.


```
all_matches <- c(matches_samd, matches_samea, matches_samn)
length(all_matches)
#3620

SRA_metadata_with_biosample$Metalog <- ifelse(SRA_metadata_with_biosample$biosample %in% all_matches, 1, 0)
sum(SRA_metadata_with_biosample$Metalog == 1, na.rm = TRUE)
#5391
```
--> numbers are not equal (all_matches length 3620 and Metalog column values returned 5391) and we need to understand why.

Lets see if there are duplicates in the biosample column or in the all_matches values.

```
library(dplyr)

# SRA_metadata_with_biosample$biosample[SRA_metadata_with_biosample$Metalog == 1] #returns a vector of biosample IDs that had a match in all_matches list (number 1).
# unique removes all the duplicate values and length gives the length of the list.

length(unique(SRA_metadata_with_biosample$biosample[SRA_metadata_with_biosample$Metalog == 1]))
# 3620 --> same value as in the all_matches list, so there are duplicates

length(unique(all_matches))
# 3620 --> all are unique values

```
There were duplicates in the SRA biosample column, lets see why.

```
#keep biosample numbers with Metalog value 1, count the biomaple values and keep only duplicates (or count > 1)
matched_rows <- SRA_metadata_with_biosample[SRA_metadata_with_biosample$Metalog == 1, ]
biosample_counts <- table(matched_rows$biosample)
duplicates <- names(biosample_counts[biosample_counts > 1])

length(duplicates)
# 381

head(duplicates)
# "SAMEA2466887" "SAMEA2466888" "SAMEA2466890" "SAMEA2466891" "SAMEA2466892" "SAMEA2466898"
```
Lets find out why there are duplicate biosample values

```
SRA_metadata_with_biosample %>% filter(biosample == "SAMEA2466887")

#        acc    biosample geo_loc_name_country_calc geo_loc_name_country_continent_calc platform          instrument
# 1 ERR480588 SAMEA2466887                                                               ILLUMINA Illumina HiSeq 2000
# 2 ERR479091 SAMEA2466887                                                               ILLUMINA Illumina HiSeq 2000
#  bioproject avgspotlen mbases collection_date_sam Metalog
# 1  PRJEB6070         63    327                           1
# 2  PRJEB6070        145   1452                           1

```
--> biosample number is the same for different acc numbers (most likely two diffenent samples from the same patient)

Collect only files with metalog value 1

```
SRA_metadata_with_biosample_matched_1 <- SRA_metadata_with_biosample %>% filter(Metalog == 1)

View(SRA_metadata_with_biosample_matched_1)
```
