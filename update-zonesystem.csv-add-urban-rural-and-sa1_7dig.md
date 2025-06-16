# Update MITO/SILO zonesystem.csv wit urban/rural classification and 7
digit SA1 code


This code is used to update the JIBE Melbourne input zonal data file
[zonesystem.csv](https://gitlab.lrz.de/ga78fel/melbourne/-/blob/5304cf021d3e1bda4969b55be5f14e3c9d89312b/input/zoneSystem.csv)
as uploaded to GitLab on 14 February 2025 with additional linkage
attributes.

``` r
library(tidyverse)
## Warning: package 'tidyverse' was built under R version 4.4.2
## Warning: package 'purrr' was built under R version 4.4.2
## Warning: package 'stringr' was built under R version 4.4.2
## Warning: package 'lubridate' was built under R version 4.4.2
## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
## ✔ dplyr     1.1.4     ✔ readr     2.1.5
## ✔ forcats   1.0.0     ✔ stringr   1.5.1
## ✔ ggplot2   3.5.1     ✔ tibble    3.2.1
## ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
## ✔ purrr     1.0.2     
## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
## ✖ dplyr::filter() masks stats::filter()
## ✖ dplyr::lag()    masks stats::lag()
## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
library(httr)
## Warning: package 'httr' was built under R version 4.4.2
library(utils)
library(readxl)
## Warning: package 'readxl' was built under R version 4.4.2
library(sf)
## Warning: package 'sf' was built under R version 4.4.3
## Linking to GEOS 3.13.0, GDAL 3.10.1, PROJ 9.5.1; sf_use_s2() is TRUE
library(data.table)
## Warning: package 'data.table' was built under R version 4.4.2
## 
## Attaching package: 'data.table'
## 
## The following objects are masked from 'package:lubridate':
## 
##     hour, isoweek, mday, minute, month, quarter, second, wday, week,
##     yday, year
## 
## The following objects are masked from 'package:dplyr':
## 
##     between, first, last
## 
## The following object is masked from 'package:purrr':
## 
##     transpose
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

## Add urban/rural classification

Urban and rural classification will be added using Australian Bureau of
Statistics [Sections of
State](https://www.abs.gov.au/ausstats/abs@.nsf/Lookup/by%20Subject/1270.0.55.004~July%202016~Main%20Features~Section%20of%20State%20(SOS)%20and%20Section%20of%20State%20Range%20(SOSR)~4)
classifications from the Australian Statistical Geography Standard
(Volume 4) for 2016. This data is available in zipped
[CSV](https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip&1270.0.55.004&Data%20Cubes&EE5F4698A91AD2F8CA2581B1000E09B0&0&July%202016&09.10.2017&Latest)
format for SA1 areas.

The previously extracted zone file contains NA values for the fields
`SA1_7DIG` and `urbanType`. Also, the abbreviations for SA1_7DIG and
SA1_MAIN are non-standard, and may relate to shapefile representations
of these identifiers. More properly, they would contain the year code,
for example, ‘SA1_MAIN16’ and ‘SA1_7DIG16’. This is because these
identifiers are year specific, and could be inadvertently result in
mal-linkage if mixed with codes from another year. To avoid this error,
and facilitate transparent linkage, the names will be updated.

``` r
zonesystem.csv <- zonesystem.csv %>%
  rename(
    SA1_MAIN16 = SA1_MAIN,
    SA1_7DIG16 = SA1_7DIG
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
##   Date: 2025-06-16 07:04
##   Status: 200
##   Content-Type: application/x-zip
##   Size: 575 kB
## <ON DISK>  D:\projects\jibe\Melbourne-zone-system\sos_data.zip

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
# Perform a left join to merge sections_of_state with zonesystem.csv on SA1_MAIN16
zonesystem_updated <- zonesystem.csv %>%
  left_join(sections_of_state, by = c("SA1_MAIN16" = "SA1_MAINCODE_2016"))

# Update the SA1_7DIG16 values in zonesystem.csv using the linked codes
zonesystem_updated <- zonesystem_updated %>%
  mutate(SA1_7DIG16 = ifelse(is.na(SA1_7DIG16), SA1_7DIGITCODE_2016, SA1_7DIG16)) %>%
  select(-SA1_7DIGITCODE_2016)

# Set urbanType based on SOS_NAME_2016
zonesystem_updated <- zonesystem_updated %>%
  mutate(urbanType = case_when(
    SOS_NAME_2016 %in% c("Major Urban", "Other Urban") ~ "urban",
    TRUE ~ "rural"
  ))

# Display the first few rows of the updated data
head(zonesystem_updated)
##   SA1_7DIG16  SA1_MAIN16 CHR EYA EE EDU       FIN       FR       PHC      RSPF
## 1    2110501 20601110501   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 2    2110502 20601110502   0   0  0   0 0.0000000 1.766788 0.0000000 0.5815015
## 3    2110503 20601110503   0   0  0   0 0.0000000 0.000000 0.4285339 0.0000000
## 4    2110504 20601110504   0   0  0   0 0.1240565 0.000000 0.0000000 0.0000000
## 5    2110505 20601110505   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 6    2110506 20601110506   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
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
##   STE_CODE_2016 STE_NAME_2016 AREA_ALBERS_SQKM
## 1             2      Victoria           0.0410
## 2             2      Victoria           0.1237
## 3             2      Victoria           0.0622
## 4             2      Victoria           0.0597
## 5             2      Victoria           0.0685
## 6             2      Victoria           0.0799
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

## Add SA2 identifier for linkage purposes

Now, we also want to add on an SA2 identifier for linkage purposes.

``` r
# Add SA2 linkage code based on the first 9 digits of SA1_MAIN16
zonesystem_updated <- zonesystem_updated %>%
  mutate(SA2_MAIN16 = substr(SA1_MAIN16, 1, 9))
```

## Add SEIFA Index of Relative Socio-economic Disadvantage deciles

We also want to link the Socio-economic Indicators for Areas (SEIFA)
Index of Relative Socio-economic Disadvantage deciles to the
zonesystem.csv file.

``` r
url <- "https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&2033055001%20-%20sa1%20indexes.xls&2033.0.55.001&Data%20Cubes&40A0EFDE970A1511CA25825D000F8E8D&0&2016&27.03.2018&Latest"
destfile_seifa <- "abs_seifa_data_2016.xls"
GET(url, write_disk(destfile_seifa, overwrite = TRUE))
## Response [https://www.ausstats.abs.gov.au/ausstats/subscriber.nsf/0/40A0EFDE970A1511CA25825D000F8E8D/$File/2033055001%20-%20sa1%20indexes.xls]
##   Date: 2025-06-16 07:04
##   Status: 200
##   Content-Type: application/vnd.ms-excel
##   Size: 54.1 MB
## <ON DISK>  D:\projects\jibe\Melbourne-zone-system\abs_seifa_data_2016.xls
# Read the SEIFA data from the Excel file
seifa_data <- read_excel(destfile_seifa, sheet = "Table 1", skip = 5, n_max = 55140)
## Warning: Expecting numeric in I12100 / R12100C9: got '-'
## Warning: Expecting numeric in J12100 / R12100C10: got '-'
## Warning: Expecting numeric in I17202 / R17202C9: got '-'
## Warning: Expecting numeric in J17202 / R17202C10: got '-'
## Warning: Expecting numeric in I22695 / R22695C9: got '-'
## Warning: Expecting numeric in J22695 / R22695C10: got '-'
## Warning: Expecting numeric in I23371 / R23371C9: got '-'
## Warning: Expecting numeric in J23371 / R23371C10: got '-'
## Warning: Expecting numeric in I25962 / R25962C9: got '-'
## Warning: Expecting numeric in J25962 / R25962C10: got '-'
## Warning: Expecting numeric in I28407 / R28407C9: got '-'
## Warning: Expecting numeric in J28407 / R28407C10: got '-'
## Warning: Expecting numeric in I29955 / R29955C9: got '-'
## Warning: Expecting numeric in J29955 / R29955C10: got '-'
## Warning: Expecting numeric in I54086 / R54086C9: got '-'
## Warning: Expecting numeric in J54086 / R54086C10: got '-'
## Warning: Expecting numeric in I54087 / R54087C9: got '-'
## Warning: Expecting numeric in J54087 / R54087C10: got '-'
## New names:
## • `` -> `...1`
## • `` -> `...2`
## • `Score` -> `Score...3`
## • `Decile` -> `Decile...4`
## • `Score` -> `Score...5`
## • `Decile` -> `Decile...6`
## • `Score` -> `Score...7`
## • `Decile` -> `Decile...8`
## • `Score` -> `Score...9`
## • `Decile` -> `Decile...10`
## • `` -> `...11`
# Display the first and last rows of the SEIFA data and manually check read in
head(seifa_data)
## # A tibble: 6 × 11
##      ...1    ...2 Score...3 Decile...4 Score...5 Decile...6 Score...7 Decile...8
##     <dbl>   <dbl> <chr>     <chr>      <chr>     <chr>      <chr>     <chr>     
## 1 1100701 1.01e10 991       4          972       4          1001      5         
## 2 1100702 1.01e10 1044      7          1044      7          1078      8         
## 3 1100703 1.01e10 980       4          962       4          951       3         
## 4 1100704 1.01e10 984       4          970       4          986       5         
## 5 1100705 1.01e10 944       3          936       3          965       4         
## 6 1100706 1.01e10 972       4          943       3          1001      5         
## # ℹ 3 more variables: Score...9 <dbl>, Decile...10 <dbl>, ...11 <dbl>
tail(seifa_data)
## # A tibble: 6 × 11
##      ...1    ...2 Score...3 Decile...4 Score...5 Decile...6 Score...7 Decile...8
##     <dbl>   <dbl> <chr>     <chr>      <chr>     <chr>      <chr>     <chr>     
## 1 9100302 9.01e10 611       1          709       1          712       1         
## 2 9100401 9.01e10 1042      7          993       5          1022      6         
## 3 9100402 9.01e10 1016      5          982       4          1003      5         
## 4 9100403 9.01e10 989       4          952       3          989       5         
## 5 9100404 9.01e10 943       3          927       3          934       3         
## 6 9100407 9.01e10 1057      7          1020      6          1039      7         
## # ℹ 3 more variables: Score...9 <dbl>, Decile...10 <dbl>, ...11 <dbl>
```

The first 4 columns are : SA1_7IG16, SA1_MAIN16, SEIFA_IRSD_2016, and
SEIFA_IRSD_DECILE_2016. We will use these to link the SEIFA deciles to
the zonesystem.csv file by SA1_MAIN16.

``` r
# Rename columns for clarity
colnames <- seifa_data %>% names()
seifa_data <- seifa_data %>%
  rename(
    SA1_7DIG16 = !!colnames[1],
    SA1_MAIN16 = !!colnames[2],
    SEIFA_IRSD_2016 = !!colnames[3],
    SEIFA_IRSD_DECILE_2016 = !!colnames[4]
  )
# Perform a left join to merge SEIFA data with zonesystem_updated on SA1_MAIN16
zonesystem_updated <- zonesystem_updated %>%
  left_join(seifa_data[,c('SA1_MAIN16','SEIFA_IRSD_DECILE_2016')], by = "SA1_MAIN16")

# Display the first few rows of the updated data with SEIFA
head(zonesystem_updated)
##   SA1_7DIG16  SA1_MAIN16 CHR EYA EE EDU       FIN       FR       PHC      RSPF
## 1    2110501 20601110501   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 2    2110502 20601110502   0   0  0   0 0.0000000 1.766788 0.0000000 0.5815015
## 3    2110503 20601110503   0   0  0   0 0.0000000 0.000000 0.4285339 0.0000000
## 4    2110504 20601110504   0   0  0   0 0.1240565 0.000000 0.0000000 0.0000000
## 5    2110505 20601110505   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 6    2110506 20601110506   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
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
##   STE_CODE_2016 STE_NAME_2016 AREA_ALBERS_SQKM SA2_MAIN16
## 1             2      Victoria           0.0410  206011105
## 2             2      Victoria           0.1237  206011105
## 3             2      Victoria           0.0622  206011105
## 4             2      Victoria           0.0597  206011105
## 5             2      Victoria           0.0685  206011105
## 6             2      Victoria           0.0799  206011105
##   SEIFA_IRSD_DECILE_2016
## 1                      7
## 2                      5
## 3                      4
## 4                      6
## 5                      8
## 6                      5
```

## Add population weighted centroid coordinates

To add population weighted centroid coordinates, we will retrieve Mesh
Block boundaries with SA1 linkage codes and join these with Mesh Block
person counts. The latter will the average of Mesh Block centroids
weighted by population data by SA1 will then be evaluated and the
resulting centroid linked on SA1.

``` r
# Download the SA1 Mesh Block boundaries for Victoria
sa1_mesh_url <- "https://www.abs.gov.au/AUSSTATS/subscriber.nsf/log?openagent&1270055001_mb_2016_vic_shape.zip&1270.0.55.001&Data%20Cubes&04F12B9E465AE765CA257FED0013B20F&0&July%202016&12.07.2016&Latest"
destfile_mesh <- "abs_sa1_mesh_block_boundaries_vic_2016.shp.zip"
GET(sa1_mesh_url, write_disk(destfile_mesh, overwrite = TRUE))
## Response [https://www.ausstats.abs.gov.au/ausstats/subscriber.nsf/0/04F12B9E465AE765CA257FED0013B20F/$File/1270055001_mb_2016_vic_shape.zip]
##   Date: 2025-06-16 07:04
##   Status: 200
##   Content-Type: application/x-zip
##   Size: 40.5 MB
## <ON DISK>  D:\projects\jibe\Melbourne-zone-system\abs_sa1_mesh_block_boundaries_vic_2016.shp.zip

mesh_blocks <- sf::read_sf(destfile_mesh, layer = "MB_2016_VIC")

person_counts_url <- "https://www.abs.gov.au/AUSSTATS/subscriber.nsf/log?openagent&2016%20census%20mesh%20block%20counts.csv&2074.0&Data%20Cubes&1DED88080198D6C6CA2581520083D113&0&2016&04.07.2017&Latest"
destfile_counts <- "abs_sa1_mesh_block_person_counts_2016.csv"
GET(person_counts_url, write_disk(destfile_counts, overwrite = TRUE))
## Response [https://www.ausstats.abs.gov.au/ausstats/subscriber.nsf/0/1DED88080198D6C6CA2581520083D113/$File/2016%20census%20mesh%20block%20counts.csv]
##   Date: 2025-06-16 07:04
##   Status: 200
##   Content-Type: application/octet-stream
##   Size: 14 MB
## <ON DISK>  D:\projects\jibe\Melbourne-zone-system\abs_sa1_mesh_block_person_counts_2016.csv
person_counts <- read_csv(destfile_counts)
## Rows: 358127 Columns: 6
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (2): MB_CODE_2016, MB_CATEGORY_NAME_2016
## dbl (4): AREA_ALBERS_SQKM, Dwelling, Person, State
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
# Join the person counts with the mesh blocks
mesh_blocks <- mesh_blocks %>%
  left_join(person_counts, by = c("MB_CODE16" = "MB_CODE_2016"))

# Re-project from EPSG4283 to EPSG 28355 
mesh_blocks <- st_transform(mesh_blocks, crs = 28355)

mesh_blocks[, c("x", "y")] <- st_coordinates(st_centroid(mesh_blocks))
## Warning: st_centroid assumes attributes are constant over geometries

# Calculate population-weighted centroid for each SA1
# Using the method described by the ABS (applied for Mesh Blocks), where the centroid is calculated as the weighted average of the coordinates of the mesh blocks within each SA1
# https://www.abs.gov.au/ausstats/abs@.nsf/Previousproducts/3218.0Glossary12016?opendocument&tabname=Notes&prodno=3218.0&issue=2016&num=&view=

# Convert mesh_blocks to data.table
dt <- as.data.table(mesh_blocks)

sa1_centroids <- dt[, .(
  mesh_block_person_count_2016 = sum(Person),
  pwc_x_epsg_28355 = sum(x * Person) / sum(Person),
  pwc_y_epsg_28355 = sum(y * Person) / sum(Person)
), by = SA1_7DIG16]

sa1_centroids$SA1_7DIG16 <- as.double(sa1_centroids$SA1_7DIG16)

# If any coordinates are NA, set these using the mean of the Mesh Block coordinates for that SA1, using data.table methods
sa1_centroids[, pwc_x_epsg_28355 := ifelse(
  is.na(pwc_x_epsg_28355),
  mean(pwc_x_epsg_28355, na.rm = TRUE),
  pwc_x_epsg_28355
), by = SA1_7DIG16]

sa1_centroids[, pwc_y_epsg_28355 := ifelse(
  is.na(pwc_y_epsg_28355),
  mean(pwc_y_epsg_28355, na.rm = TRUE),
  pwc_y_epsg_28355
), by = SA1_7DIG16]

# Join the population-weighted centroids back to the zonesystem_updated
zonesystem_updated <- zonesystem_updated %>%
  left_join(sa1_centroids, by = "SA1_7DIG16")

zonesystem_updated %>% head()
##   SA1_7DIG16  SA1_MAIN16 CHR EYA EE EDU       FIN       FR       PHC      RSPF
## 1    2110501 20601110501   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 2    2110502 20601110502   0   0  0   0 0.0000000 1.766788 0.0000000 0.5815015
## 3    2110503 20601110503   0   0  0   0 0.0000000 0.000000 0.4285339 0.0000000
## 4    2110504 20601110504   0   0  0   0 0.1240565 0.000000 0.0000000 0.0000000
## 5    2110505 20601110505   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
## 6    2110506 20601110506   0   0  0   0 0.0000000 0.000000 0.0000000 0.0000000
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
##   STE_CODE_2016 STE_NAME_2016 AREA_ALBERS_SQKM SA2_MAIN16
## 1             2      Victoria           0.0410  206011105
## 2             2      Victoria           0.1237  206011105
## 3             2      Victoria           0.0622  206011105
## 4             2      Victoria           0.0597  206011105
## 5             2      Victoria           0.0685  206011105
## 6             2      Victoria           0.0799  206011105
##   SEIFA_IRSD_DECILE_2016 mesh_block_person_count_2016 pwc_x_epsg_28355
## 1                      7                          219         321348.3
## 2                      5                          632         320813.2
## 3                      4                          448         321107.4
## 4                      6                          353         321306.5
## 5                      8                          356         321039.9
## 6                      5                          445         320708.2
##   pwc_y_epsg_28355
## 1          5818989
## 2          5818610
## 3          5818556
## 4          5818280
## 5          5818304
## 6          5818351
```

## Export final data

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
write.csv(zonesystem_updated[c(zonesystem.csv.columns,"SA2_MAIN16","SEIFA_IRSD_DECILE_2016", "mesh_block_person_count_2016","pwc_x_epsg_28355", "pwc_y_epsg_28355")], output_file, row.names = FALSE)
```
