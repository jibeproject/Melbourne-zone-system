# Update MITO/SILO zonesystem.csv with additional linkage attributes


This code is used to update the JIBE Melbourne input zonal data file
[zonesystem.csv](https://gitlab.lrz.de/ga78fel/melbourne/-/blob/5304cf021d3e1bda4969b55be5f14e3c9d89312b/input/zoneSystem.csv)
as uploaded to GitLab on 14 February 2025 with additional linkage
attributes.

# Summary of Methods

## Overview

This document describes the methods used to enhance the JIBE Melbourne
zonesystem.csv file with additional linkage attributes from Australian
Bureau of Statistics (ABS) data sources. The original zone system file
contained basic zonal identifiers but lacked comprehensive linkage codes
and socio-economic attributes necessary for detailed transport modeling
analysis.

## Data Sources

The following ABS data sources were integrated:

1.  **Sections of State (SOS) Data (2016)**: Urban/rural classification
    data from the Australian Statistical Geography Standard
    - Source: ABS 1270.0.55.004 SA1 Urban Centres and Localities/Section
      of State/Section of State Range 2016
    - Purpose: Classify zones as urban or rural
2.  **SEIFA Index Data (2016)**: Socio-Economic Indexes for Areas
    - Source: ABS 2033.0.55.001 SA1 Index of Relative Socio-economic
      Disadvantage
    - Purpose: Add socio-economic disadvantage deciles for demographic
      analysis
3.  **Mesh Block Boundaries and Population Data (2016)**: Geometric and
    demographic data
    - Source: ABS 1270.0.55.001 Mesh Block Digital Boundaries and ABS
      Census Mesh Block Counts
    - Purpose: Calculate population-weighted centroids for each
      Statistical Area 1 (SA1)
4.  **Dwelling Vacancy Data (2016)**: Population and Housing Census data
    - Source: ABS 2016 Census Population and Housing ‘2016 Place of
      Enumeration Profile for Statistical Area 1 (SA1)’ datapack for
      Victoria SA1
    - Purpose: Calculate rate of vacant dwellings

## Methods Applied

### 1. Standardization of Linkage Codes

- Renamed `SA1_MAIN` and `SA1_7DIG` to `SA1_MAIN16` and `SA1_7DIG16`
  respectively
- This ensures year-specific identification and prevents inadvertent
  linkage errors with other census years
- Created `SA2_MAIN16` identifier from the first 9 digits of
  `SA1_MAIN16` for higher-level geographic linkage

### 2. Urban/Rural Classification

- Linked Sections of State data using `SA1_MAIN16` as the join key
- Classified zones as:
  - “urban”: Major Urban and Other Urban areas
  - “rural”: All other areas including migratory and no usual address
    categories
- Updated missing `SA1_7DIG16` values from the linked SOS data where
  available

### 3. Socio-Economic Disadvantage Integration

- Integrated SEIFA Index of Relative Socio-economic Disadvantage deciles
- Linked using `SA1_MAIN16` to provide standardized socio-economic
  rankings (1-10 scale)
- Decile 1 represents most disadvantaged areas, decile 10 represents
  least disadvantaged

### 4. Population-Weighted Centroid Calculation

Applied the Australian Bureau of Statistics method for calculating
population-weighted centroids:

**Method**: For each SA1, the latitude and longitude coordinates of the
geographic centroid of each constituent Mesh Block were multiplied by
the population of each Mesh Block to obtain weighted coordinates. These
were summed and divided by the total population of the SA1 to obtain a
single population-weighted coordinate.

**Implementation**: - Downloaded Mesh Block boundaries for Victoria
(2016) and re-projected to EPSG:28355 (GDA94 MGA Zone 55) - Calculated
geometric centroids for each Mesh Block - Joined with Census population
counts by Mesh Block - Computed population-weighted mean coordinates
grouped by SA1 - For SA1s with zero population, used unweighted
geometric mean of constituent Mesh Blocks

### 5. Dwelling Vacancy Rate Calculation

- Calculated percentage vacant dwellings as:
  `100 × (Unoccupied_private_dwgs_Dwgs / Total_Dwellings)`
- Implemented error checking for division by zero and missing data
- Linked to zone system using `SA1_7DIG16` ↔ `SA1_7DIGITCODE_2016`

## Technical Implementation

- Used R with tidyverse, sf (spatial), and data.table packages for
  efficient data processing
- Employed vectorized operations and data.table syntax for performance
  optimization
- Implemented comprehensive error checking and validation throughout the
  process
- All data downloads are conditional to avoid redundant downloads

## Quality Assurance

- Verified coordinate completeness after population-weighted centroid
  calculation
- Checked for non-numeric results in vacancy rate calculations
- Displayed warnings and diagnostic information for problematic cases
- Maintained original column structure while adding new attributes

## Output

The enhanced zone system file includes all original attributes plus: -
`SA2_MAIN16`: SA2 linkage identifier - `SEIFA_IRSD_DECILE_2016`:
Socio-economic disadvantage decile (1-10) -
`mesh_block_person_count_2016`: Total population count from constituent
Mesh Blocks - `pwc_x_epsg_28355`, `pwc_y_epsg_28355`:
Population-weighted centroid coordinates in GDA94 MGA Zone 55 -
`vacancy_rate`: Percentage of unoccupied private dwellings

This enhanced dataset provides comprehensive demographic,
socio-economic, and spatial attributes suitable for detailed transport
demand modeling and analysis in the JIBE Melbourne model system.

``` r
# Ensure required packages are installed and loaded. This will install any missing CRAN packages.
required_pkgs <- c("tidyverse", "httr", "readxl", "sf", "data.table")

# Identify packages not currently installed
installed <- rownames(installed.packages())
missing_pkgs <- setdiff(required_pkgs, installed)
if (length(missing_pkgs) > 0) {
  # Ensure a CRAN mirror is set (non-interactive sessions can fail without this)
  repos <- getOption("repos")
  if (is.null(repos) || is.na(repos["CRAN"]) || repos["CRAN"] == "@CRAN@" || repos["CRAN"] == "") {
    options(repos = c(CRAN = "https://cloud.r-project.org"))
  }

  message("Installing missing packages: ", paste(missing_pkgs, collapse = ", "))
  tryCatch(
    install.packages(missing_pkgs, dependencies = TRUE),
    error = function(e) {
      stop("Failed to install packages: ", paste(missing_pkgs, collapse = ", "), "\\n",
           "Install manually or set options(repos = ...) and re-run.\\nError: ", e$message)
    }
  )

  # Re-check installed packages after attempted install
  installed <- rownames(installed.packages())
  still_missing <- setdiff(missing_pkgs, installed)
  if (length(still_missing) > 0) {
    stop("The following packages could not be installed automatically: ",
         paste(still_missing, collapse = ", "), ".\\n",
         "If you are on Windows, installing 'sf' may require OS-level dependencies (GDAL/PROJ).",
         " Install them first or install packages manually and re-run the script.")
  }
}
## Installing missing packages: sf
## Installing package into 'C:/Users/E33390/AppData/Local/R/win-library/4.5'
## (as 'lib' is unspecified)
## also installing the dependencies 'geometries', 'rapidjsonr', 'sfheaders', 'systemfonts', 'logger', 'geojsonsf', 'leaflet.providers', 'svglite', 'plyr', 'goftest', 'tensor', 'stringdist', 'spacesXYZ', 'jsonify', 'slippymath', 'XML', 'leafem', 'leaflet', 'leafpop', 'satellite', 'servr', 'plogr', 'spatstat.data', 'spatstat.univar', 'spatstat.explore', 'spatstat.model', 'deldir', 'polyclip', 'spatstat.sparse', 'abind', 'cols4all', 'leafgl', 'leaflegend', 'leafsync', 'maptiles', 'tmaptools', 's2', 'units', 'nanoarrow', 'covr', 'lwgeom', 'maps', 'mapview', 'microbenchmark', 'odbc', 'pbapply', 'pool', 'raster', 'RPostgres', 'RPostgreSQL', 'RSQLite', 'sp', 'spatstat', 'spatstat.geom', 'spatstat.random', 'spatstat.linnet', 'spatstat.utils', 'stars', 'terra', 'tmap', 'wk'
## package 'geometries' successfully unpacked and MD5 sums checked
## package 'rapidjsonr' successfully unpacked and MD5 sums checked
## package 'sfheaders' successfully unpacked and MD5 sums checked
## package 'systemfonts' successfully unpacked and MD5 sums checked
## package 'logger' successfully unpacked and MD5 sums checked
## package 'geojsonsf' successfully unpacked and MD5 sums checked
## package 'leaflet.providers' successfully unpacked and MD5 sums checked
## package 'svglite' successfully unpacked and MD5 sums checked
## package 'plyr' successfully unpacked and MD5 sums checked
## package 'goftest' successfully unpacked and MD5 sums checked
## package 'tensor' successfully unpacked and MD5 sums checked
## package 'stringdist' successfully unpacked and MD5 sums checked
## package 'spacesXYZ' successfully unpacked and MD5 sums checked
## package 'jsonify' successfully unpacked and MD5 sums checked
## package 'slippymath' successfully unpacked and MD5 sums checked
## package 'XML' successfully unpacked and MD5 sums checked
## package 'leafem' successfully unpacked and MD5 sums checked
## package 'leaflet' successfully unpacked and MD5 sums checked
## package 'leafpop' successfully unpacked and MD5 sums checked
## package 'satellite' successfully unpacked and MD5 sums checked
## package 'servr' successfully unpacked and MD5 sums checked
## package 'plogr' successfully unpacked and MD5 sums checked
## package 'spatstat.data' successfully unpacked and MD5 sums checked
## package 'spatstat.univar' successfully unpacked and MD5 sums checked
## package 'spatstat.explore' successfully unpacked and MD5 sums checked
## package 'spatstat.model' successfully unpacked and MD5 sums checked
## package 'deldir' successfully unpacked and MD5 sums checked
## package 'polyclip' successfully unpacked and MD5 sums checked
## package 'spatstat.sparse' successfully unpacked and MD5 sums checked
## package 'abind' successfully unpacked and MD5 sums checked
## package 'cols4all' successfully unpacked and MD5 sums checked
## package 'leafgl' successfully unpacked and MD5 sums checked
## package 'leaflegend' successfully unpacked and MD5 sums checked
## package 'leafsync' successfully unpacked and MD5 sums checked
## package 'maptiles' successfully unpacked and MD5 sums checked
## package 'tmaptools' successfully unpacked and MD5 sums checked
## package 's2' successfully unpacked and MD5 sums checked
## package 'units' successfully unpacked and MD5 sums checked
## package 'nanoarrow' successfully unpacked and MD5 sums checked
## package 'covr' successfully unpacked and MD5 sums checked
## package 'lwgeom' successfully unpacked and MD5 sums checked
## package 'maps' successfully unpacked and MD5 sums checked
## package 'mapview' successfully unpacked and MD5 sums checked
## package 'microbenchmark' successfully unpacked and MD5 sums checked
## package 'odbc' successfully unpacked and MD5 sums checked
## package 'pbapply' successfully unpacked and MD5 sums checked
## package 'pool' successfully unpacked and MD5 sums checked
## package 'raster' successfully unpacked and MD5 sums checked
## package 'RPostgres' successfully unpacked and MD5 sums checked
## package 'RPostgreSQL' successfully unpacked and MD5 sums checked
## package 'RSQLite' successfully unpacked and MD5 sums checked
## package 'sp' successfully unpacked and MD5 sums checked
## package 'spatstat' successfully unpacked and MD5 sums checked
## package 'spatstat.geom' successfully unpacked and MD5 sums checked
## package 'spatstat.random' successfully unpacked and MD5 sums checked
## package 'spatstat.linnet' successfully unpacked and MD5 sums checked
## package 'spatstat.utils' successfully unpacked and MD5 sums checked
## package 'stars' successfully unpacked and MD5 sums checked
## package 'terra' successfully unpacked and MD5 sums checked
## package 'tmap' successfully unpacked and MD5 sums checked
## package 'wk' successfully unpacked and MD5 sums checked
## package 'sf' successfully unpacked and MD5 sums checked
## 
## The downloaded binary packages are in
##  C:\Users\E33390\AppData\Local\Temp\Rtmp2lHplL\downloaded_packages

# Load base 'utils' explicitly (it is part of base R) and then load the required packages
library(utils)
invisible(lapply(required_pkgs, function(pkg) {
  suppressPackageStartupMessages(library(pkg, character.only = TRUE))
}))
## Warning: package 'sf' was built under R version 4.5.2
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
destfile <- "sos_data.zip"
url <- "https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip&1270.0.55.004&Data%20Cubes&EE5F4698A91AD2F8CA2581B1000E09B0&0&July%202016&09.10.2017&Latest"

# Download the file, if it hasn't been downloaded already
if (!file.exists(destfile)) {
  message("Downloading Sections of State data...")
  GET(url, write_disk(destfile, overwrite = TRUE))
}

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
destfile_seifa <- "abs_seifa_data_2016.xls"
url <- "https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&2033055001%20-%20sa1%20indexes.xls&2033.0.55.001&Data%20Cubes&40A0EFDE970A1511CA25825D000F8E8D&0&2016&27.03.2018&Latest"

# Download data if it hasn't been downloaded already
if (!file.exists(destfile_seifa)) {
  message("Downloading SEIFA data...")
  GET(url, write_disk(destfile_seifa, overwrite = TRUE))
}

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

# Join additional SEIFA linkage codes using SA1_7DIG16 and "2016 Statistical Area Level 2 (SA2) Name" (renamed as SA2_NAME_2016)
seifa_linkage <- read_excel(destfile_seifa, sheet = "Table 7", skip = 4, n_max = 57523)
zonesystem_updated <- zonesystem_updated %>%
  left_join(seifa_linkage[, c("2016 Statistical Area Level 1  (SA1) 7-Digit Code", "2016 Statistical Area Level 2 (SA2) Name")], 
            by = c("SA1_7DIG16" = "2016 Statistical Area Level 1  (SA1) 7-Digit Code")) %>%
  rename(SA2_NAME_2016 = `2016 Statistical Area Level 2 (SA2) Name`)

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
##   SEIFA_IRSD_DECILE_2016 SA2_NAME_2016
## 1                      7     Brunswick
## 2                      5     Brunswick
## 3                      4     Brunswick
## 4                      6     Brunswick
## 5                      8     Brunswick
## 6                      5     Brunswick
```

## Add population weighted centroid coordinates

To add population weighted centroid coordinates, we will retrieve Mesh
Block boundaries with SA1 linkage codes and join these with Mesh Block
person counts. The latter will the average of Mesh Block centroids
weighted by population data by SA1 will then be evaluated and the
resulting centroid linked on SA1.

``` r
destfile_mesh <- "abs_sa1_mesh_block_boundaries_vic_2016.shp.zip"
sa1_mesh_url <- "https://www.abs.gov.au/AUSSTATS/subscriber.nsf/log?openagent&1270055001_mb_2016_vic_shape.zip&1270.0.55.001&Data%20Cubes&04F12B9E465AE765CA257FED0013B20F&0&July%202016&12.07.2016&Latest"

# Download data if it hasn't been downloaded already
if (!file.exists(destfile_mesh)) {
  message("Downloading SA1 Mesh Block boundaries...")
  GET(sa1_mesh_url, write_disk(destfile_mesh, overwrite = TRUE))
}

mesh_blocks <- sf::read_sf(destfile_mesh, layer = "MB_2016_VIC")

person_counts_url <- "https://www.abs.gov.au/AUSSTATS/subscriber.nsf/log?openagent&2016%20census%20mesh%20block%20counts.csv&2074.0&Data%20Cubes&1DED88080198D6C6CA2581520083D113&0&2016&04.07.2017&Latest"
destfile_counts <- "abs_sa1_mesh_block_person_counts_2016.csv"

# Download data if it hasn't been downloaded already
if (!file.exists(destfile_counts)) {
  message("Downloading Mesh Block person counts...")
  GET(person_counts_url, write_disk(destfile_counts, overwrite = TRUE))
}

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

# If any coordinates are NA, set these using the mean of the Mesh Block coordinates for each SA1
sa1_centroids[is.na(pwc_x_epsg_28355), pwc_x_epsg_28355 := dt[SA1_7DIG16 == .BY$SA1_7DIG16, mean(x, na.rm = TRUE)], by = SA1_7DIG16]
sa1_centroids[is.na(pwc_y_epsg_28355), pwc_y_epsg_28355 := dt[SA1_7DIG16 == .BY$SA1_7DIG16, mean(y, na.rm = TRUE)], by = SA1_7DIG16]

# Join the population-weighted centroids back to the zonesystem_updated
zonesystem_updated <- zonesystem_updated %>%
  left_join(sa1_centroids, by = "SA1_7DIG16")

# Verify that there are no NA coordinates for SA1s in the final dataset
na_count <- zonesystem_updated %>%
  filter(is.na(pwc_x_epsg_28355) | is.na(pwc_y_epsg_28355)) %>%
  nrow()
if (na_count > 0) {
  print(paste("There are", na_count, "SA1s with NA coordinates in the final dataset."))
  # Show which SA1s have missing coordinates
  missing_coords <- zonesystem_updated %>%
    filter(is.na(pwc_x_epsg_28355) | is.na(pwc_y_epsg_28355)) %>%
    select(SA1_7DIG16, SA1_MAIN16, pwc_x_epsg_28355, pwc_y_epsg_28355)
  print(missing_coords)
}

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
##   SEIFA_IRSD_DECILE_2016 SA2_NAME_2016 mesh_block_person_count_2016
## 1                      7     Brunswick                          219
## 2                      5     Brunswick                          632
## 3                      4     Brunswick                          448
## 4                      6     Brunswick                          353
## 5                      8     Brunswick                          356
## 6                      5     Brunswick                          445
##   pwc_x_epsg_28355 pwc_y_epsg_28355
## 1         321348.3          5818989
## 2         320813.2          5818610
## 3         321107.4          5818556
## 4         321306.5          5818280
## 5         321039.9          5818304
## 6         320708.2          5818351
```

## Add percentage vacant dwellings

We’ll add the percentage of vacant dwellings using data from the 2016
Census Population and Housing data for SA1 areas in Victoria.

``` r
destfile_dwellings <- "2016_PEP_SA1_for_VIC_short-header.zip"
dwellings_url <- "https://www.abs.gov.au/census/find-census-data/datapacks/download/2016_PEP_SA1_for_VIC_short-header.zip"

# Download data if it hasn't been downloaded already
if (!file.exists(destfile_dwellings)) {
  message("Downloading dwelling data...")
  GET(dwellings_url, write_disk(destfile_dwellings, overwrite = TRUE))
}

# Unzip the file
unzip(destfile_dwellings, exdir = "dwelling_data")

# Read the dwelling data CSV file
dwelling_data <- read_csv("dwelling_data/2016 Census PEP Statistical Area 1 for VIC/2016Census_P31_VIC_SA1.csv")
## Rows: 14072 Columns: 37
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr  (1): Unoccupied_private_dwgs_Psns
## dbl (36): SA1_7DIGITCODE_2016, Separate_house_Dwellings, Separate_house_Pers...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

# Calculate percentage vacant dwellings
dwelling_data <- dwelling_data %>%
  mutate(
    vacancy_rate = ifelse(Total_Dwellings == 0, 0, Unoccupied_private_dwgs_Dwgs / Total_Dwellings)
  )

# Check for non-numeric results and display warnings
non_numeric_count <- dwelling_data %>%
  filter(is.na(vacancy_rate) | is.infinite(vacancy_rate)) %>%
  nrow()

if (non_numeric_count > 0) {
  warning(paste("There are", non_numeric_count, "SA1s with non-numeric vacancy_rate values (NA or Inf)."))
  
  # Show problematic cases
  problematic_cases <- dwelling_data %>%
    filter(is.na(vacancy_rate) | is.infinite(vacancy_rate)) %>%
    select(SA1_7DIGITCODE_2016, Total_Dwellings, Unoccupied_private_dwgs_Dwgs, vacancy_rate)
  print(problematic_cases)
}

# Join dwelling data with zonesystem_updated
zonesystem_updated <- zonesystem_updated %>%
  left_join(dwelling_data[, c("SA1_7DIGITCODE_2016", "vacancy_rate")], 
            by = c("SA1_7DIG16" = "SA1_7DIGITCODE_2016"))

# Display summary of the new field
summary(zonesystem_updated$vacancy_rate)
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
## 0.00000 0.04571 0.06977 0.07860 0.09688 0.88978
```

## Summary of the updated zonesystem.csv

Let’s summarise the updated `zonesystem.csv` file to see the new fields
that have been added:

``` r
summary(zonesystem_updated)
##    SA1_7DIG16        SA1_MAIN16             CHR                EYA         
##  Min.   :2110501   Min.   :2.060e+10   Min.   : 0.00000   Min.   :0.00000  
##  1st Qu.:2118610   1st Qu.:2.080e+10   1st Qu.: 0.00000   1st Qu.:0.00000  
##  Median :2127614   Median :2.110e+10   Median : 0.00000   Median :0.00000  
##  Mean   :2127474   Mean   :2.102e+10   Mean   : 0.03594   Mean   :0.08425  
##  3rd Qu.:2135128   3rd Qu.:2.121e+10   3rd Qu.: 0.00000   3rd Qu.:0.00000  
##  Max.   :2146828   Max.   :2.140e+10   Max.   :13.12053   Max.   :5.50649  
##        EE               EDU               FIN                 FR         
##  Min.   : 0.0000   Min.   :0.00000   Min.   : 0.00000   Min.   : 0.0000  
##  1st Qu.: 0.0000   1st Qu.:0.00000   1st Qu.: 0.00000   1st Qu.: 0.0000  
##  Median : 0.0000   Median :0.00000   Median : 0.00000   Median : 0.0000  
##  Mean   : 0.1966   Mean   :0.03715   Mean   : 0.05198   Mean   : 0.1368  
##  3rd Qu.: 0.0000   3rd Qu.:0.00000   3rd Qu.: 0.00000   3rd Qu.: 0.0000  
##  Max.   :63.4058   Max.   :4.94803   Max.   :19.58055   Max.   :53.1626  
##       PHC               RSPF               SER                SCL         
##  Min.   :0.00000   Min.   : 0.00000   Min.   :  0.0000   Min.   :0.00000  
##  1st Qu.:0.00000   1st Qu.: 0.00000   1st Qu.:  0.0000   1st Qu.:0.00000  
##  Median :0.00000   Median : 0.00000   Median :  0.0000   Median :0.00000  
##  Mean   :0.02961   Mean   : 0.04616   Mean   :  0.3699   Mean   :0.02772  
##  3rd Qu.:0.00000   3rd Qu.: 0.00000   3rd Qu.:  0.0000   3rd Qu.:0.00000  
##  Max.   :8.20925   Max.   :10.86753   Max.   :195.4371   Max.   :7.26985  
##       work           education              HH              POS           
##  Min.   :  0.000   Min.   :  0.0000   Min.   :   0.0   Min.   :  0.00000  
##  1st Qu.:  0.000   1st Qu.:  0.0000   1st Qu.: 123.0   1st Qu.:  0.00000  
##  Median :  0.000   Median :  0.0000   Median : 164.0   Median :  0.00863  
##  Mean   :  1.944   Mean   :  0.8178   Mean   : 178.6   Mean   :  1.75673  
##  3rd Qu.:  1.000   3rd Qu.:  0.0000   3rd Qu.: 214.0   3rd Qu.:  0.37550  
##  Max.   :410.000   Max.   :171.0000   Max.   :3278.0   Max.   :696.66954  
##     HH_sqrt       urbanType         UCL_CODE_2016    UCL_NAME_2016     
##  Min.   : 0.00   Length:10289       Min.   :201001   Length:10289      
##  1st Qu.:11.09   Class :character   1st Qu.:201001   Class :character  
##  Median :12.81   Mode  :character   Median :201001   Mode  :character  
##  Mean   :12.87                      Mean   :202383                     
##  3rd Qu.:14.63                      3rd Qu.:201001                     
##  Max.   :57.25                      Max.   :231777                     
##  SOSR_CODE_2016  SOSR_NAME_2016     SOS_CODE_2016   SOS_NAME_2016     
##  Min.   :201.0   Length:10289       Min.   :20.00   Length:10289      
##  1st Qu.:201.0   Class :character   1st Qu.:20.00   Class :character  
##  Median :201.0   Mode  :character   Median :20.00   Mode  :character  
##  Mean   :202.4                      Mean   :20.13                     
##  3rd Qu.:201.0                      3rd Qu.:20.00                     
##  Max.   :231.0                      Max.   :23.00                     
##  STE_CODE_2016 STE_NAME_2016      AREA_ALBERS_SQKM    SA2_MAIN16       
##  Min.   :2     Length:10289       Min.   :  0.0023   Length:10289      
##  1st Qu.:2     Class :character   1st Qu.:  0.1091   Class :character  
##  Median :2     Mode  :character   Median :  0.1537   Mode  :character  
##  Mean   :2                        Mean   :  0.9712                     
##  3rd Qu.:2                        3rd Qu.:  0.2384                     
##  Max.   :2                        Max.   :275.6103                     
##  SEIFA_IRSD_DECILE_2016 SA2_NAME_2016      mesh_block_person_count_2016
##  Length:10289           Length:10289       Min.   :   0.0              
##  Class :character       Class :character   1st Qu.: 332.0              
##  Mode  :character       Mode  :character   Median : 418.0              
##                                            Mean   : 435.8              
##                                            3rd Qu.: 519.0              
##                                            Max.   :4354.0              
##  pwc_x_epsg_28355 pwc_y_epsg_28355   vacancy_rate    
##  Min.   :270549   Min.   :5739815   Min.   :0.00000  
##  1st Qu.:318143   1st Qu.:5801139   1st Qu.:0.04571  
##  Median :329156   Median :5811083   Median :0.06977  
##  Mean   :328590   Mean   :5808989   Mean   :0.07860  
##  3rd Qu.:339876   3rd Qu.:5820157   3rd Qu.:0.09688  
##  Max.   :395506   Max.   :5876189   Max.   :0.88978
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
write.csv(zonesystem_updated[c(zonesystem.csv.columns,"SA2_MAIN16","SA2_NAME_2016","SEIFA_IRSD_DECILE_2016", "mesh_block_person_count_2016","pwc_x_epsg_28355", "pwc_y_epsg_28355", "vacancy_rate")], output_file, row.names = FALSE)
```
