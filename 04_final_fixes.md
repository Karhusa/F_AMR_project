
# Final fixes

## Download the file

```{r}
port numpy as np
import pandas as pd
import re

df = pd.read_csv("kesken2.tsv", sep="\t")

```

Size 
```
print(df.shape)
(24605, 37)



['acc', 'age_category', 'age_range', 'age_years', 'avgspotlen', 'bioproject', 'biosample', 'collection_date_sam', 'environmental_package', 'geo_loc_name_country_calc', 'geo_loc_name_country_continent_calc', 'instrument', 'location_resolution', 'mbases', 'platform', 'raw_metadata_Carbapenemase_bla_Gene', 'raw_metadata_Colitis', 'raw_metadata_Drug_antivirus', 'raw_metadata_IBD', 'raw_metadata_MaternalAntimicrobials', 'raw_metadata_TotalAntimicrobialsDays', 'raw_metadata_multi_drug_resistant_organism_infection', 'sex', 'spire_sample_name', 'antibiotics_list', 'antibiotic_1', 'antibiotic_2', 'antibiotic_3', 'antibiotic_4', 'antibiotic_5', 'antibiotic_6', 'antibiotic_7', 'antibiotic_8', 'antibiotic_9', 'name_of_antibiotic', 'Antibiotics_used', 'UTI_history']



```
