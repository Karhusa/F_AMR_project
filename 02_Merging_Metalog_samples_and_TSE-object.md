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

```


```
awk -F, 'NR==1 {for (i=1; i<=NF; i++) if ($i=="ena_ers_sample_id") col=i} NR>1 {print $col}' samd_ids.csv > samd_ids.txt
awk -F, 'NR==1 {for (i=1; i<=NF; i++) if ($i=="ena_ers_sample_id") col=i} NR>1 {print $col}' samea_ids.csv > samea_ids.txt
awk -F, 'NR==1 {for (i=1; i<=NF; i++) if ($i=="ena_ers_sample_id") col=i} NR>1 {print $col}' samn_ids.csv > samn_ids.txt
```

```
cut -d',' -f2 SRA_metadata_with_biosample_corrected.txt > biosample_ids.txt

grep -Fxf <(cut -d',' -f1 samd_ids.txt) biosample_ids.txt > matches_samd.txt
grep -Fxf <(cut -d',' -f1 samea_ids.txt) biosample_ids.txt > matches_samea.txt
grep -Fxf <(cut -d',' -f1 samn_ids.txt) biosample_ids.txt > matches_samn.txt

```

```
Python 3.12.0 (v3.12.0:0fb18b02c8, Oct  2 2023, 09:45:56) [Clang 13.0.0 (clang-1300.0.29.30)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import pandas as pd
>>> 

```



