# Update MITO/SILO zonesystem.csv wit urban/rural classification and 7
digit SA1 code


This code is used to update the JIBE Melbourne input zonal data file
[zonesystem.csv](https://gitlab.lrz.de/ga78fel/melbourne/-/blob/5304cf021d3e1bda4969b55be5f14e3c9d89312b/input/zoneSystem.csv)
as uploaded to GitLab on 14 February 2025 with urban and rural
classification using Australian Bureau of Statistics [Sections of
State](https://www.abs.gov.au/ausstats/abs@.nsf/Lookup/by%20Subject/1270.0.55.004~July%202016~Main%20Features~Section%20of%20State%20(SOS)%20and%20Section%20of%20State%20Range%20(SOSR)~4)
classifications from the Australian Statistical Geography Standard
(Volume 4) for 2016. This data is available in zipped
[CSV](https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip&1270.0.55.004&Data%20Cubes&EE5F4698A91AD2F8CA2581B1000E09B0&0&July%202016&09.10.2017&Latest)
format for SA1 areas.

``` r
library(tidyverse)
## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
## ✔ dplyr     1.1.4     ✔ readr     2.1.5
## ✔ forcats   1.0.0     ✔ stringr   1.5.1
## ✔ ggplot2   3.5.1     ✔ tibble    3.2.1
## ✔ lubridate 1.9.3     ✔ tidyr     1.3.1
## ✔ purrr     1.0.2     
## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
## ✖ dplyr::filter() masks stats::filter()
## ✖ dplyr::lag()    masks stats::lag()
## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
library(httr)
library(utils)
```

First, we’ll load in the current Melbourne `zonesystem.csv` file
(current as of 18 March 2024):

``` r
file_path <- file.choose()
zonesystem.csv <- read.csv(file_path)
zonesystem.csv %>% head()
##   SA1_7DIG    SA1_MAIN CHR EYA EE EDU       FIN       FR       PHC      RSPF
## 1       NA 20601110501   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 2       NA 20601110502   0   0  0   0 0.0000000 1.766788 0.0000000 0.5815015
## 3       NA 20601110503   0   0  0   0 0.0000000 0.000000 0.4285339 0.0000000
## 4       NA 20601110504   0   0  0   0 0.1240565 0.000000 0.0000000 0.0000000
## 5       NA 20601110505   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 6       NA 20601110506   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
##         SER SCL work education  HH POS  HH_sqrt urbanType
## 1 0.0000000   0    0         0 105   0 10.24695        NA
## 2 0.7915657   0    4         0 364   0 19.07878        NA
## 3 0.0000000   0    0         0 193   0 13.89244        NA
## 4 0.2426518   0    0         0 180   0 13.41641        NA
## 5 0.0000000   0    1         0 172   0 13.11488        NA
## 6 0.5908710   0    0         0 231   0 15.19868        NA
```

This contains NA values for the fields `SA1_7DIG` and `urbanType`. Also,
the abbreviations for SA1_7DIG and SA1_MAIN are non-standard, and may
relate to shapefile representations of these identifiers. More properly,
they would contain the year code, for example, ‘SA1_MAINCODE_2016’ and
‘SA1_7DIGITCODE_2016’. This is because these identifiers are year
specific, and could be inadvertently result in mal-linkage if mixed with
codes from another year. To avoid this error, and facilitate transparent
linkage, the names will be updated.

``` r
zonesystem.csv <- zonesystem.csv %>%
  rename(
    SA1_MAINCODE_2016 = SA1_MAIN,
    SA1_7DIGITCODE_2016 = SA1_7DIG
  )
zonesystem.csv.columns <- zonesystem.csv %>% names()
```

``` r
# Define the URL and the destination file path
url <- "https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip&1270.0.55.004&Data%20Cubes&EE5F4698A91AD2F8CA2581B1000E09B0&0&July%202016&09.10.2017&Latest"
destfile <- "sos_data.zip"

# Download the file
GET(url, write_disk(destfile, overwrite = TRUE))
## Response [https://www.ausstats.abs.gov.au/ausstats/subscriber.nsf/0/EE5F4698A91AD2F8CA2581B1000E09B0/$File/1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip]
##   Date: 2025-03-19 00:16
##   Status: 200
##   Content-Type: application/x-zip
##   Size: 575 kB
## <ON DISK>  C:\Users\carlh\OneDrive - RMIT University\General - JIBE working group\melbourne\travel_demand_model_mito\sos_data.zip

# Unzip the file
unzip(destfile, exdir = "sos_data")

# List the files in the unzipped directory
unzipped_files <- list.files("sos_data", full.names = TRUE)

# Read the CSV file (assuming there's only one CSV file in the unzipped directory)
sections_of_state <- read_csv(unzipped_files[grepl("\\.csv$", unzipped_files)])
## Rows: 57523 Columns: 11
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (4): UCL_NAME_2016, SOSR_NAME_2016, SOS_NAME_2016, STE_NAME_2016
## dbl (7): SA1_MAINCODE_2016, SA1_7DIGITCODE_2016, UCL_CODE_2016, SOSR_CODE_20...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

# Display the first few rows of the data
head(sections_of_state)
## # A tibble: 6 × 11
##   SA1_MAINCODE_2016 SA1_7DIGITCODE_2016 UCL_CODE_2016 UCL_NAME_2016             
##               <dbl>               <dbl>         <dbl> <chr>                     
## 1       10102100701             1100701        131777 Remainder of State/Territ…
## 2       10102100702             1100702        131777 Remainder of State/Territ…
## 3       10102100703             1100703        115024 Braidwood                 
## 4       10102100704             1100704        115024 Braidwood                 
## 5       10102100705             1100705        115024 Braidwood                 
## 6       10102100706             1100706        131777 Remainder of State/Territ…
## # ℹ 7 more variables: SOSR_CODE_2016 <dbl>, SOSR_NAME_2016 <chr>,
## #   SOS_CODE_2016 <dbl>, SOS_NAME_2016 <chr>, STE_CODE_2016 <dbl>,
## #   STE_NAME_2016 <chr>, AREA_ALBERS_SQKM <dbl>
```

``` r
# Perform a left join to merge sections_of_state with zonesystem.csv on SA1_MAINCODE_2016
zonesystem_updated <- zonesystem.csv %>%
  left_join(sections_of_state, by = c("SA1_MAINCODE_2016" = "SA1_MAINCODE_2016"))

# Update the SA1_7DIGITCODE_2016 values in zonesystem.csv using the linked codes
zonesystem_updated <- zonesystem_updated %>%
  mutate(SA1_7DIGITCODE_2016 = ifelse(is.na(SA1_7DIGITCODE_2016.x), SA1_7DIGITCODE_2016.y, SA1_7DIGITCODE_2016.x)) %>%
  select(-SA1_7DIGITCODE_2016.x, -SA1_7DIGITCODE_2016.y)

# Set urbanType based on SOS_NAME_2016
zonesystem_updated <- zonesystem_updated %>%
  mutate(urbanType = case_when(
    SOS_NAME_2016 %in% c("Major Urban", "Other Urban") ~ "urban",
    TRUE ~ "rural"
  ))

# Display the first few rows of the updated data
head(zonesystem_updated)
##   SA1_MAINCODE_2016 CHR EYA EE EDU       FIN       FR       PHC      RSPF
## 1       20601110501   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 2       20601110502   0   0  0   0 0.0000000 1.766788 0.0000000 0.5815015
## 3       20601110503   0   0  0   0 0.0000000 0.000000 0.4285339 0.0000000
## 4       20601110504   0   0  0   0 0.1240565 0.000000 0.0000000 0.0000000
## 5       20601110505   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 6       20601110506   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
##         SER SCL work education  HH POS  HH_sqrt urbanType UCL_CODE_2016
## 1 0.0000000   0    0         0 105   0 10.24695     urban        201001
## 2 0.7915657   0    4         0 364   0 19.07878     urban        201001
## 3 0.0000000   0    0         0 193   0 13.89244     urban        201001
## 4 0.2426518   0    0         0 180   0 13.41641     urban        201001
## 5 0.0000000   0    1         0 172   0 13.11488     urban        201001
## 6 0.5908710   0    0         0 231   0 15.19868     urban        201001
##   UCL_NAME_2016 SOSR_CODE_2016    SOSR_NAME_2016 SOS_CODE_2016 SOS_NAME_2016
## 1     Melbourne            201 1 million or more            20   Major Urban
## 2     Melbourne            201 1 million or more            20   Major Urban
## 3     Melbourne            201 1 million or more            20   Major Urban
## 4     Melbourne            201 1 million or more            20   Major Urban
## 5     Melbourne            201 1 million or more            20   Major Urban
## 6     Melbourne            201 1 million or more            20   Major Urban
##   STE_CODE_2016 STE_NAME_2016 AREA_ALBERS_SQKM SA1_7DIGITCODE_2016
## 1             2      Victoria           0.0410             2110501
## 2             2      Victoria           0.1237             2110502
## 3             2      Victoria           0.0622             2110503
## 4             2      Victoria           0.0597             2110504
## 5             2      Victoria           0.0685             2110505
## 6             2      Victoria           0.0799             2110506
```

Let’s just summarise the classifications before we update
`zonesystem.csv` before outputting the updated file:

``` r
table(zonesystem_updated$SOS_NAME_2016) 
## 
## Bounded Locality      Major Urban      Other Urban    Rural Balance 
##               40             9562              413              274
table(zonesystem_updated$urbanType)  
## 
## rural urban 
##   314  9975
```

This makes sense for Greater Melbourne, that it is overwhelmingly urban.

Now lets update `zonesystem.csv` using the updated data for the original
set of columns, and then save it as a CSV in an output folder (so we
don’t get it mixed up with the input data, that we don’t wish to
overwrite for reasons of transparency). Then, we’ll manually upload this
to the Melbourne Gitlab data repository.

``` r
# save the updated zonesystem.csv to a new output folder
output_folder <- file.path(dirname(file_path), "output")
dir.create(output_folder, showWarnings = FALSE)
output_file <- file.path(output_folder, "zonesystem.csv")
write.csv(zonesystem_updated[zonesystem.csv.columns], output_file, row.names = FALSE)
```
