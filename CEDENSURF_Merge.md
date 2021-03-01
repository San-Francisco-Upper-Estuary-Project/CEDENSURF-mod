---
title: "CEDENSURF Merge - Official Short"
author: "Erika W"
date: "2/23/2021"
output:
  html_document:
    code_download: true
    keep_md: true
    code_folding: hide
    toc: true
    toc_float:
      toc_collapsed: true
    toc_depth: 3
    theme: lumen
---




```r
rm(list = ls())
library(data.table)
library(lubridate)
library(sf)
library(tidyverse)
```

# Intro

This markdown covers the process to identify duplication within the CEDEN and SURF datasets, and to merge those datasets together to make a quality-assured database for use in our analyses.

# Load Data

SURF Data was acquired at the DPR SURF database web page as CSVs via FTP download on 2/17/2021, and modified via the methods outlined in https://github.com/WWU-IETC-R-Collab/CEDENSURF-mod/blob/main/CEDENSURF.md prior to this work.

CEDEN Data was acquired from https://ceden.waterboards.ca.gov/AdvancedQueryTool on January 29 2020 for the Central Valley and San Francisco Bay regions, and spatially queried to the USFE project area. This original data set can be found within the IETC Tox Box at: Upper San Francisco Project\Data & Analyses\Original\CEDEN. The methods of prior modification are at: https://github.com/WWU-IETC-R-Collab/CEDEN-mod/blob/main/CEDEN_ModMaster.md

<br>

#### CEDEN


```r
# Load CEDEN Data
CEDENMod_Tox <- fread("https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/Output/CEDENMod_Toxicity.csv")

CEDENMod_WQ <- fread("https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/Output/CEDENMod_WQ.csv")
```
Two files - one with tox data, and one with wq data

Date range of CEDEN water data: from 2009-10-06 to 2019-09-26

Date range of CEDEN tox data: from 2009-10-06 to 2019-09-25

<br> 

#### SURF


```r
SURFMod_SED <- fread("https://github.com/WWU-IETC-R-Collab/CEDENSURF-mod/raw/main/Data/Output/SURFMod_SED.csv")

SURFMod_WQ <- fread("https://github.com/WWU-IETC-R-Collab/CEDENSURF-mod/raw/main/Data/Output/SURFMod_water.csv")
```
Two files - one with wq data, and one with sediment data

Date range of SURF water data: from NA to NA

Date range of SURF sediment data: from from NA to NA

<br>

#### Append with source


```r
CEDENMod_Tox$Source <- rep("CEDEN", times=nrow(CEDENMod_Tox))

CEDENMod_WQ$Source <- rep("CEDEN", times=nrow(CEDENMod_WQ))

SURFMod_SED$Source <- rep("SURF", times=nrow(SURFMod_SED))

SURFMod_WQ$Source <- rep("SURF", times=nrow(SURFMod_WQ))
```

<br>

<br>

# Data prep

Due to reported combined efforts to translate CEDEN data to SURF and vice versa, and issues with replicates being retained in each dataset, careful detection and elimination of duplicates should precede any analysis.

<br>

## CEDEN

There are 60429 records in the original WQ dataset

and 60531 in the original Tox dataset. 

<br>

### Data prep

#### Remove duplicates

Removing exact duplicates via duplicated() misses duplication of wq data due to multiple species assessments, different sources of data upload, etc. 

Instead, we used distinct() which allows us to assume that records in the same location on the same date,  measuring the same analyte via the same collection method, and obtaining the same result are duplicates. This method deletes almost 50% more records than duplicated() 


```r
# Remove duplicate entries, determined as those matching all 5 columns names within distinct().

# Since we are using the Tox database for the water parameters, not the associated organism survival, we remove duplicate WQ entries regardless of the organism assessed.

NoDup_Tox<- distinct(CEDENMod_Tox, Date, StationName, Analyte, CollectionMethod, Result, .keep_all= TRUE) 

# How many duplicate entries were identified and removed?

nrow(CEDENMod_Tox) - nrow(NoDup_Tox) # 23,518
```

```
## [1] 23518
```


```r
# 499 exact duplicates in the newest download

# Remove duplicate rows of the dataframe using multiple variables

NoDup_WQ <- distinct(CEDENMod_WQ, Date, Analyte, StationName, CollectionMethod, Result, .keep_all= TRUE)

nrow(CEDENMod_WQ) - nrow(NoDup_WQ) # 1336
```

```
## [1] 1336
```
<br>

#### Remove irrelevant data

Since we are using the Tox database for the water parameters, not the associated organism survival, we can also remove records that assess organism status.


```r
# We can also remove records that assess organism status (since we aren't using this for the biotic parameters in our model).

NoDup_Tox <- NoDup_Tox %>% filter(Analyte != "Survival") %>%
  filter(Analyte != "Biomass (wt/orig indiv)") %>%
  filter(Analyte != "Young/female") %>%
  filter(Analyte != "Total Cell Count") %>%
  select(-OrganismName)
```
<br>

####

After CEDEN data prep, there are 29695 unique, useful records in the tox dataset, and 59093 unique records in the WQ dataset.

<br>

<br>

### Merge CEDEN df

After dealing with duplication WITHIN the CEDEN tox and wq datasets, there were only 9 duplicate records found following the merged data. (75 if Collection Method is not a requirement for establishing duplication)


```r
# Vector of column names to compare
WQ <- names(NoDup_WQ)
TOX <- names(NoDup_Tox)

#Add missing columns to CEDEN WQ
DIF<- setdiff(TOX, WQ) # gives items in T that are not in W
NoDup_WQ[, DIF] <- NA
```

```
## Warning in `[<-.data.table`(`*tmp*`, , DIF, value = NA): length(LHS)==0; no
## columns to delete or assign RHS to.
```

```r
#Add missing columns to CEDEN TOX
DIF<- setdiff(WQ, TOX) # gives items in W that are not in T
NoDup_Tox[, DIF] <- NA

# Finishing touches before merge; order columns to match - is this really necessary for merge? IDK

WQ <- sort(names(NoDup_WQ))
TOX <- sort(names(NoDup_Tox))

NoDup_Tox <- NoDup_Tox %>% select(all_of(TOX))
NoDup_WQ <- NoDup_WQ %>% select(all_of(WQ))

# Check once all columns have perfect matches?
# tibble(SURF = names(NoDup_Tox), CEDEN = names(NoDup_WQ))

# MERGE
CEDEN_ALL <- rbind(NoDup_WQ,NoDup_Tox)
```
<br>

### Further refine

#### Remove duplicates


```r
# Remove duplicate rows of the dataframe using multiple variables

CEDEN_ALL_DupChecked <- distinct(CEDEN_ALL, Date, Analyte, CollectionMethod, StationName, Result, .keep_all= TRUE)
```

<br>

#### Problematic duplicates 

Further assessment revealed problematic data duplication that was not caught when requiring Collection Methods to be equal. We corrected the majority of these errors by:

1. Removing records where Collection Method = "Not Recorded" (Of 305 samples labeled "not recorded" in the entire CEDEN dataset, 153 were duplicated data with non-zero results)

2. *(Add another bullet & code if we take action on Sediment Core vs Grab issues)*


```r
# While there were 307 entries marked "Not Recorded" in the entire combined dataset, 153 were identified as duplicated data. To err on the safe side, we removed all records for which the Collection Method was "Not Recorded"

CEDEN_ALL_DupChecked <- filter(CEDEN_ALL_DupChecked, CollectionMethod != "Not Recorded")
```

<br>

#### Fix nomenclature

**Split Analyte Column**

Because of formatting differences between the amount of data recorded under "Analyte" in CEDEN compared to the "Chemical_name" in SURF (which will be renamed Analyte), we opted to split the data in CEDEN's analyte column into two columns: 

Analyte (Chemical Name), and Analyte_Type (ie: total or particulate)

Using separate() to split the column and requiring the separation to be a comma and space ", " seems to have worked well, except for one name which appears to be empty

<br>

**Convert to lower-case**

SURF chemicals are all lowercase. We can make all letters in the CEDEN data lowercase using tolower() so that they will be compatible.


```r
# Split Analyte column

CEDEN_ALL_DupChecked <- CEDEN_ALL_DupChecked %>%
  separate(Analyte, into = c("Analyte", "Analyte_type"), sep = ", " , extra = "merge")
```

```
## Warning: Expected 2 pieces. Missing pieces filled with `NA` in 19377 rows [54,
## 55, 57, 58, 101, 103, 104, 106, 140, 144, 145, 146, 673, 678, 681, 682, 2657,
## 2658, 2660, 2664, ...].
```

```r
# Convert to lowercase

CEDEN_ALL_DupChecked$Analyte <- tolower(CEDEN_ALL_DupChecked$Analyte)

# Preview
head(sort(unique(CEDEN_ALL_DupChecked$Analyte))) # 908 unique Analytes total
```

```
## [1] ""                                     
## [2] "1,2-bis(2,4,6- tribromophenoxy)ethane"
## [3] "2-ethylhexyl-diphenyl phosphate"      
## [4] "2,4,6-tribromophenyl allyl ether"     
## [5] "acenaphthene"                         
## [6] "acenaphthenes"
```

```r
# Looks like requiring the separation from extra to analyte to contain a comma and a space allowed the full names to be retained. Without that, the separation led to an analyte "1" which should have been 1,2-bis(2,4,6- tribromophenoxy)ethane, etc.
```

This simplification of Analyte name would lead to more records being considered "duplication" using our current method, 90% of which containing a zero-result. Because these differences in analytes are not retained in the SURF dataset, it makes sense to condense them (remove duplicates) prior to merging the databases.

Removal of these simplified duplicates also eliminates the utility of retaining the "Analyte Type" column. For example, a reading of Analyte X with Type = Total Type = Suspended have the exact same result. Removing duplicates would keep the first record (Analyte = X, Type = Total) and remove the second (Analyte X, Type = S). In the dataframe, if you are trying to reason backwards to the meaning of that remaining meaning (Analyte X, Type = Total), you're missing the other half of the story (Type = Sus too). So, to avoid improper interpretation of this dataframe, Analyte Type should be removed. 


```r
CEDEN_ALL_DupChecked <- distinct(CEDEN_ALL_DupChecked, Date, Analyte, CollectionMethod, StationName, Result, .keep_all= TRUE) %>%
  select(-Analyte_type)
```

<br>

### CEDEN merge result

Using these QA/QC methods, 87794 unique records are available through the CEDEN datasets. 

<br>

<br>

## SURF data

There are 91021 records in the WQ dataset
and 35346 in the SED dataset. 

There were no exact duplicates in either the WQ or SED data from SURF. Far fewer duplicates were located using our flexible methods than in the CEDEN dataset.

<br>

### Data prep

#### Rename columns

We renamed columns with analogous data to match CEDEN column names.


```r
### SURF WATER

# Move units from embedded in Result column name to their own column
SURFMod_WQ$Unit <- "ppb"

# Rename columns with analogous data to match CEDEN column names.
SURFMod_WQ <- SURFMod_WQ %>% rename(Date = Sample_date,
          Analyte = Chemical_name, 
          Result = Concentration..ppb., 
          CollectionMethod = Sample_type, 
          StationCode = Site_code,
          StationName = Site_name,
          MDL = Method_detection_level..ppb.,
          LOQ = Level_of_quantification..ppb.)

### SURF SEDIMENT

# Move units from embedded in Result column name to their own column
SURFMod_SED$Unit <- "ppb"

# Rename columns with analogous data to match CEDEN column names.
SURFMod_SED <- SURFMod_SED %>% rename(Date = Sample_date,
          Analyte = Chemical_name, 
          Result = Concentration..ppb., 
          CollectionMethod = Sample_type, 
          StationCode = Site_code,
          StationName = Site_name,
          MDL = Method_detection_level..ppb.,
          LOQ = Level_of_quantification..ppb.)
```

#### Remove duplicates

We used distinct() to remove records in the same location on the same date, measuring the same analyte via the same collection method which had identical results.


```r
# Remove duplicate rows of the dataframe using multiple variables

# SURF Water
NoDup_WQ <- distinct(SURFMod_WQ, Date, Analyte, CollectionMethod, StationName, Result, .keep_all= TRUE)

# SURF Sediment
NoDup_SED <- distinct(SURFMod_SED, Date, Analyte, CollectionMethod, StationName, Result, .keep_all= TRUE)
```

This results in 75688 unique records in the WQ dataset
and 31266 unique records in the SED dataset, prior to merging.

<br>

### Merge SURF df


```r
# Vector of column names to compare
WQ <- names(NoDup_WQ)
SED <- names(NoDup_SED)

#Add missing columns to CEDEN WQ
DIF<- setdiff(SED, WQ) # gives items in S that are not in W
NoDup_WQ[, DIF] <- NA

#Add missing columns to CEDEN SED
DIF<- setdiff(WQ, SED) # gives items in W that are not in S
NoDup_SED[, DIF] <- NA
```

```
## Warning in `[<-.data.table`(`*tmp*`, , DIF, value = NA): length(LHS)==0; no
## columns to delete or assign RHS to.
```

```r
# Finishing touches before merge; order columns to match - is this really necessary for merge? IDK

WQ <- sort(names(NoDup_WQ))
SED <- sort(names(NoDup_SED))

NoDup_SED <- NoDup_SED %>% select(all_of(SED))
NoDup_WQ <- NoDup_WQ %>% select(all_of(WQ))

# Check once all columns have perfect matches?
# tibble(SURF = names(NoDup_SED), CEDEN = names(NoDup_WQ))

# MERGE
SURF_ALL <- rbind(NoDup_WQ,NoDup_SED)
```

<br>

### Further refine

#### Duplication between SURF sets

ZERO duplication found between the SED and WQ datasets, assuming duplicates would have to have the exact same Location, Date, Analyte, Collection Method, and Result.


```r
# Remove duplicate rows of the dataframe using multiple variables

SURF_ALL_DupChecked <- distinct(SURF_ALL, Date, Analyte, CollectionMethod, StationName, Result, .keep_all= TRUE)

nrow(SURF_ALL)-nrow(SURF_ALL_DupChecked)
```

```
## [1] 0
```

We further investigated instances of duplicated entries retained using these methods which differed only in their collection methods, specifically targeting records with identical results != 0.

All but one were corrected by removing records by Study_cd 305, which was an exact replicate of Study_cd 523 except missing collection methods.


```r
SURF_ALL_DupChecked <- filter(SURF_ALL_DupChecked, Study_cd != "305")
```

<br>

### SURF merge result

There are 106890 unique records available through SURF.


```r
# Create a subset of SURF data that excludes data sourced from CEDEN

SURF_ALL_NC <- filter(SURF_ALL_DupChecked, Data.source != "CEDEN")
```

That said, 28100 of these records are listed as having been sourced from CEDEN.

In theory only 78790 unique records will be contributed through the SURF dataset. Rather than filter these out ahead of the merge, I am retaining them and then using the identification of those records as a test to see whether there are other differentiating factors (such as persisting differences in naming) between the merged dataset that will inhibit our analyses

<br>

<br>

# Merge SURF and CEDEN

Now that each dataset has been independently inspected for duplication *within* the dataset, they can be merged and searched for duplication *between* the datasets.


```r
## Data Prep - Match Columns between DF prior to merge

C <- names(CEDEN_ALL_DupChecked)
S <- names(SURF_ALL_DupChecked)

DIF<- setdiff(S, C) # gives items in S that are not in C

#Add missing columns to CEDEN
CEDEN_ALL_DupChecked[, DIF] <- NA

#Add missing columns to SURF
DIF<- setdiff(C, S) # gives items in C that are not in S
SURF_ALL_DupChecked[, DIF] <- NA

# Re-order columns to align
C <- sort(names(CEDEN_ALL_DupChecked)) # 908
S <- sort(names(SURF_ALL_DupChecked)) # 327

SURF_ALL_DupChecked <- SURF_ALL_DupChecked %>% select(all_of(S))
CEDEN_ALL_DupChecked <- CEDEN_ALL_DupChecked %>% select(all_of(C))

# Check once all columns have perfect matches?
# tibble(SURF = names(SURF_ALL_DupChecked), CEDEN = names(CEDEN_ALL_DupChecked))

## MERGE ##

CEDENSURF <- rbind(SURF_ALL_DupChecked, CEDEN_ALL_DupChecked)
```

There are 194684 total records in the initial merge of CEDEN with SURF.

Due to initial barriers to removing duplicates between the datasets (see below), I will simply filter out data identified as being sourced from CEDEN within SURF to eliminate duplicates. This is not an ideal solution though, because there is a large amount of data identified as coming from CEDEN which is not present in our CEDEN WQ data (again, see below).


```r
CEDENSURFMod <- filter(CEDENSURF, Data.source != "CEDEN")

write_csv(CEDENSURFMod, "Data/Output/CEDENSURFMod.csv") # Note: coerces empty data fields to NA
```

<br>

## Next Steps

### 1. Investigate barriers to duplicate removal {.tabset}
*(area of active investigation)*

Because the station names differ between these databases, we used Lat and Long in lieu of StationName to detect duplicates.

This only works if the projection and rounding of latitude and longitude have been made consistent both within and between the datasets (see linked protocols for our CEDENMod and SURFMod data preparation).

It seems to only detect 11 duplicates, while there are 28100 labeled in SURF as having come from CEDEN. Some may be unique, and some may be additional duplicates that we are not catching. 


```r
# Remove duplicate rows of the dataframe using multiple variables

CEDENSURF_DupChecked <- distinct(CEDENSURF, Date, Analyte, CollectionMethod, Latitude, Longitude, Result, .keep_all= TRUE)

nrow(CEDENSURF)-nrow(CEDENSURF_DupChecked)
```

```
## [1] 11
```

**Causes of these records being retained include:**

A. No match actually exists in CEDEN. Record SHOULD be retained.

B. CEDEN and SURF have different naming protocols - both Station Name and Station Code differ for the same sites.

C. Latitude and Longitude may differ between the databases - this ought to be corrected now that Skyler has unified the projections used for all data.

#### Examples of A

**1. Glyphosate in SURF-CEDEN data, but not in CEDEN data.**

Comparing analytes measured at Grizzley Bay includes a number of records of glyphosate in SURF-CEDEN data, yet the CEDEN set reveals none including glyphosate.

An example record from SURF cited as coming from CEDEN:

```r
A <- CEDENSURF %>% filter(grepl('Grizzly', StationName)) %>%
  filter(grepl('Dolphin', StationName)) %>%
  filter(Source == "SURF") %>%
  filter(Data.source == "CEDEN")%>%
  filter(Analyte == "glyphosate")

A[1]
```

```
##                     Agency    Analyte          CollectionMethod County
## 1: Michael L. Johnson, LLC glyphosate Single whole water sample Solano
##    Data.source       Date Datum                geometry Latitude LocationCode
## 1:       CEDEN 2012-05-08  <NA> c(-122.03972, 38.11708) 38.11708         <NA>
##    Longitude LOQ MatrixName MDL ParentProject Program Project rb_number
## 1: -122.0397   5       <NA> 1.7          <NA>    <NA>    <NA>        NA
##    Record_id regional_board Result RL Source StationCode
## 1:   1763867           <NA>      0 NA   SURF       48_52
##                                                   StationName Study_cd
## 1: Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7      244
##                                                                                        Study_description
## 1: SuisunBayMonitoring _BACWA , Suisun Bay Monitoring Project , BACWA Suisun Bay Monitoring Project 2012
##                                                                                Study_weblink
## 1: http://www.swrcb.ca.gov/sanfranciscobay/water_issues/programs/SWAMP/SB_Workplan_11-12.pdf
##     Subregion Total organic carbon (%) Unit
## 1: Suisun Bay                       NA  ppb
```
All of the records in SURF measuring glyphosate at this station, cited as coming from CEDEN, are from the same agency: Michael L. Johnson, LLC

In contrast, none of the records from CEDEN at this site claim to measure glyphosate. We can preview all analytes observed in CEDEN's records at this site to confirm that these are not simply due to naming errors:

```r
# Locate similar location records in CEDEN and SURF using queries

# Record Analyte = "endosulfan sulfate", Agency = Applied Marine Sciences, Inc. California, Collection Method = Filtered water sample, Date = 1997-04-22, Station Name: "Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7", StationCode: "48_52"

# Locate similar location records in CEDEN and SURF using flexible queries

A <- CEDENSURF %>% filter(grepl('Grizzly', StationName)) %>%
  filter(grepl('Dolphin', StationName)) %>%
  filter(Source == "SURF") %>%
  filter(Data.source == "CEDEN")

CS <- sort(unique(A$Analyte)) # 27

B <- CEDENSURF %>% filter(grepl('Grizzly', StationName)) %>%
  filter(grepl('Dolphin', StationName)) %>%
  filter(Source == "CEDEN")

C <- sort(unique(B$Analyte)) # 34
C
```

```
##  [1] "ammonia as n"                 "arsenic"                     
##  [3] "atrazine"                     "azoxystrobin"                
##  [5] "cadmium"                      "chlorophyll a"               
##  [7] "chromium"                     "copper"                      
##  [9] "dichlorobenzenamine"          "dichlorophenyl-3-methyl urea"
## [11] "dissolved organic carbon"     "diuron"                      
## [13] "hexazinone"                   "imidacloprid"                
## [15] "lead"                         "manganese"                   
## [17] "mbas"                         "mercury"                     
## [19] "nickel"                       "nitrate as n"                
## [21] "nitrite as n"                 "nitrogen"                    
## [23] "orthophosphate as p"          "oxygen"                      
## [25] "ph"                           "phosphorus as p"             
## [27] "salinity"                     "secchi depth"                
## [29] "silicate as si"               "silver"                      
## [31] "simazine"                     "specificconductivity"        
## [33] "temperature"                  "zinc"
```

There are other analytes at this site recorded in SURF-CEDEN data but not in the data brought directly from CEDEN, too: Datum, LocationCode, MatrixName, ParentProject, Program, Project, rb_number, regional_board, RL


```r
DIF<- setdiff(CS, C) #Analytes in ceden-surf and not in ceden
DIF
```


**2. CEDEN-SURF data at Toe Drain Nr Babel Slough not in CEDEN** 

SURF Name: "Toe Drain Nr Babel Slough Nr Freeport Ca"
SURF Code: "57_58"

Could not ID this location in CEDEN data, using grepl() to allow approximate naming. None with "Babel" and T in StationName


```r
# From SURF, identified as coming from CEDEN
CS <- CEDENSURF %>% filter(grepl('Toe', StationName)) %>%
  filter(grepl('Babel', StationName)) %>%
  filter(Data.source == "CEDEN")%>%
  filter(Source == "SURF")

# Direct from CEDEN
C <- CEDENSURF %>%
  filter(grepl('Babel', StationName)) %>%
  filter(Source == "CEDEN")
```
There are 657 records at this site from SURF, all from the same agency, and identified as coming from CEDEN. 
Here is one example record:


```r
nrow(CS)
```

```
## [1] 657
```

```r
unique(CS$Agency)
```

```
## [1] "USGS California Water Science Center"
```

```r
CS[1]
```

```
##                                  Agency      Analyte      CollectionMethod
## 1: USGS California Water Science Center pyrimethanil Filtered water sample
##    County Data.source       Date Datum                   geometry Latitude
## 1:   Yolo       CEDEN 2016-07-28  <NA> c(-121.588225, 38.4747806) 38.47478
##    LocationCode Longitude    LOQ MatrixName  MDL ParentProject Program Project
## 1:         <NA> -121.5882 0.0041       <NA> -999          <NA>    <NA>    <NA>
##    rb_number Record_id regional_board Result RL Source StationCode
## 1:        NA   1805112           <NA>      0 NA   SURF       57_58
##                                 StationName Study_cd
## 1: Toe Drain Nr Babel Slough Nr Freeport Ca      747
##                                                                                                Study_description
## 1: SFCWA YoloBypassFoodWeb , State and Federal Contractors Water Agency , SFCWA YoloBypassFoodWeb PEST 2015-2016
##            Study_weblink   Subregion Total organic carbon (%) Unit
## 1: http://www.ceden.org/ North Delta                       NA  ppb
```


#### Examples of B

It appears that SURF has appended station names from CEDEN with extra info. For example:

SURF: "Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7"
SURF Code: "48_52"

CEDEN: "Grizzly Bay at Dolphin nr. Suisun Slough"
CEDEN Code: "207SNB0D7"

and 

SURF Name: "Sacramento River at Freeport (USGS-11447650)"
SURF Code: "34_5"

CEDEN Name: "Sacramento River at Freeport, CA"
CEDEN Code: 11447650


```r
rbind(A[1,], B[1,])
```

```
##                                  Agency Analyte      CollectionMethod County
## 1: USGS California Water Science Center linuron Filtered water sample Solano
## 2:                                 <NA>  oxygen          Field Method   <NA>
##    Data.source       Date Datum                       geometry Latitude
## 1:       CEDEN 2011-04-21  <NA>        c(-122.03972, 38.11708) 38.11708
## 2:        <NA> 2010-03-17 NAD83 c(-122.03971862793, 38.117081) 38.11708
##    LocationCode Longitude    LOQ  MatrixName  MDL
## 1:         <NA> -122.0397 0.0043        <NA> -999
## 2:    OpenWater -122.0397     NA samplewater   NA
##                         ParentProject                       Program
## 1:                               <NA>                          <NA>
## 2: Suisun Bay Monitoring Project RWB2 Suisun Bay Monitoring Project
##                                  Project rb_number Record_id    regional_board
## 1:                                  <NA>        NA   1805592              <NA>
## 2: RWB2 Suisun Bay Monitoring Study 2010         2        NA San Francisco Bay
##    Result RL Source StationCode
## 1:   0.00 NA   SURF       48_52
## 2:   9.16 NA  CEDEN   207SNB0D7
##                                                   StationName Study_cd
## 1: Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7      825
## 2:                   Grizzly Bay at Dolphin nr. Suisun Slough       NA
##                                                                                                 Study_description
## 1: SuisunBayMonitoring _SFCWA_USGS , Suisun Bay Monitoring Project , USGS Suisun Bay Monitoring Project 2011-2012
## 2:                                                                                                           <NA>
##            Study_weblink  Subregion Total organic carbon (%) Unit
## 1: http://www.ceden.org/ Suisun Bay                       NA  ppb
## 2:                  <NA> Suisun Bay                       NA mg/L
```

#### Example of C

Same station at Grizzly Bay; differences in coordinates listed in SURF (1998 and 2012), compared to CEDEN (2010)


```r
C <- CEDENSURF %>%
  filter(grepl('Grizzly', StationName)) %>%
  filter(grepl('Dolphin', StationName)) %>%
  filter(grepl('2012', Date)) %>%
  filter(Data.source == "CEDEN")

rbind(A[1,c(23, 25, 7:10,12)], C[1,c(23, 25, 7:10,12)], B[1,c(23, 25, 7:10,12)])
```

```
##    Source                                                StationName Datum
## 1:   SURF Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7  <NA>
## 2:   SURF Grizzly Bay at Dolphin nr. Suisun Slough. CEDEN: 207SNB0D7  <NA>
## 3:  CEDEN                   Grizzly Bay at Dolphin nr. Suisun Slough NAD83
##                          geometry Latitude LocationCode    LOQ
## 1:        c(-122.03972, 38.11708) 38.11708         <NA> 0.0043
## 2:        c(-122.03972, 38.11708) 38.11708         <NA> 0.0037
## 3: c(-122.03971862793, 38.117081) 38.11708    OpenWater     NA
```

### 2. Decide whether to retain composite samples

If we decide to remove them, determine how to identify.

In CEDEN, might be able to locate by CollectionMethod OR CollectionDeviceDescription

ie: "7 day auto sampler" and "AutoSampler" collection methods may indicate composite over time, or "depth-integrating" collection device description may indivate composite over depths.
