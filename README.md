Sample code for extracting AEA RCT data
================
Lars Vilhuber
2022-01-29

## Preliminaries

I used both the stable release on
[CRAN](https://cran.r-project.org/package=dataverse), and installed the
latest development version from
[GitHub](https://github.com/iqss/dataverse-client-r/) (both as of
2022-01-29).

``` r
# Install from CRAN
install.packages("dataverse")

# Install from GitHub
# install.packages("remotes")
remotes::install_github("iqss/dataverse-client-r")
```

See the documentation on how to record the `DATAVERSE_KEY`. You can set
the API Token (or key) **when logged in** at
<https://dataverse.harvard.edu/dataverseuser.xhtml?selectTab=apiTokenTab>.

``` r
Sys.setenv("DATAVERSE_SERVER" = "dataverse.harvard.edu")
```

I am using the Dataverse at **dataverse.harvard.edu**. A key is included
in the `.Renviron`, in the prescribed format:

``` r
DATAVERSE_KEY="examplekey12345"
```

## Finding the data

``` r
library("dataverse")
library("tidyverse")
# we need the short name for the AEA RCT dataverse
rct.repo.name <- "aearegistry"
```

Let’s first get all the records:

``` r
rct.dv <- get_dataverse(rct.repo.name)
rct.dv.contents <- dataverse_contents(rct.dv)
```

The resulting data is a list of 23 elements:

    ## Dataset (3656537): https://doi.org/10.7910/DVN/DFMLIU
    ## Publisher: Harvard Dataverse
    ## publicationDate: 2020-01-17
    ## 
    ## Dataset (3676911): https://doi.org/10.7910/DVN/THFDBX
    ## Publisher: Harvard Dataverse
    ## publicationDate: 2020-02-03
    ## 
    ## Dataset (3742449): https://doi.org/10.7910/DVN/ZW3XWF
    ## Publisher: Harvard Dataverse
    ## publicationDate: 2020-03-02

We will get them all, read in the list of data files, and keep those
that have “trials.csv” file (and for some earlier ones, a variation on
that name):

``` r
rct.files <- tibble()
for (i in rct.dv.contents ) {
  tmp <- as_tibble(get_dataset(i)$files) %>%
    filter(originalFileFormat == "text/csv") %>%
    filter(str_detect(originalFileName,"trials") | str_detect(originalFileName,"Registry_Trials"))
  # for some reason, more recent deposits do not have the DOI in the files content
  if ( tmp$pidURL == "" ) { tmp$pidURL <- as.character(i$persistentUrl)}
  rct.files <- bind_rows(rct.files,tmp)
}
```

with variable names

``` r
names(rct.files)
```

    ##  [1] "label"               "restricted"          "version"            
    ##  [4] "datasetVersionId"    "id"                  "persistentId"       
    ##  [7] "pidURL"              "filename"            "contentType"        
    ## [10] "filesize"            "storageIdentifier"   "originalFileFormat" 
    ## [13] "originalFormatLabel" "originalFileSize"    "originalFileName"   
    ## [16] "UNF"                 "rootDataFileId"      "md5"                
    ## [19] "checksum"            "creationDate"        "description"

and contents

``` r
head(rct.files)
```

    ## # A tibble: 6 × 21
    ##   label  restricted version datasetVersionId     id persistentId pidURL filename
    ##   <chr>  <lgl>        <int>            <int>  <int> <chr>        <chr>  <chr>   
    ## 1 aea_t… FALSE            1           181001 3.67e6 doi:10.7910… https… aea_tri…
    ## 2 trial… FALSE            2           181000 3.68e6 doi:10.7910… https… trials.…
    ## 3 trial… FALSE            2           187899 3.74e6 doi:10.7910… https… trials.…
    ## 4 trial… FALSE            2           192477 3.79e6 doi:10.7910… https… trials …
    ## 5 trial… FALSE            3           194250 3.83e6 doi:10.7910… https… trials …
    ## 6 trial… FALSE            3           196398 3.86e6 doi:10.7910… https… trials …
    ## # … with 13 more variables: contentType <chr>, filesize <int>,
    ## #   storageIdentifier <chr>, originalFileFormat <chr>,
    ## #   originalFormatLabel <chr>, originalFileSize <int>, originalFileName <chr>,
    ## #   UNF <chr>, rootDataFileId <int>, md5 <chr>, checksum <df[,2]>,
    ## #   creationDate <chr>, description <chr>

This is to ensure we don’t keep any codebooks or auxiliary files.

If we simply want to keep the latest, let’s do so:

``` r
rct.files %>% 
  filter(creationDate == max(creationDate)) %>%
  select(label,id,persistentId,pidURL,filename,originalFileName,creationDate) -> rct.latest
```

We should remember to keep track of when, since “latest” changes over
time:

-   AEA Registry Data used here as of 2022-01-11
-   DOI <https://doi.org/10.7910/DVN/6LSFVC>.

We should also cite it appropriately (not done here).

We could also use the `creationDate` or the name of the deposit to
filter on a particular version (identified by a unique DOI).

## Loading the data

We can now simply load the file by DOI or by `id`:

``` r
rct.data.raw <- get_dataframe_by_id(fileid=rct.latest$id[1],.f = readr::read_csv,original = TRUE)
```

    ## Rows: 5417 Columns: 50── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (39): Title, Url, Last update date, Published at, RCT_ID, DOI Number, P...
    ## lgl   (4): Data collection completion, Attrition correlated, Public data, Pr...
    ## date  (7): First registered on, Start date, End date, Intervention start dat...
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
head(rct.data.raw %>% select(RCT_ID,Title,`Last update date`))
```

    ## # A tibble: 6 × 3
    ##   RCT_ID          Title                                         `Last update d…`
    ##   <chr>           <chr>                                         <chr>           
    ## 1 AEARCTR-0000005 Voter Pessimism and Electoral Accountability… May 02, 2017    
    ## 2 AEARCTR-0000006 Community Based Strategies to Reduce Materna… September 07, 2…
    ## 3 AEARCTR-0000008 An Evaluation of Continuous Comprehensive Ev… June 15, 2013   
    ## 4 AEARCTR-0000009 Enhancing Local Public Service Delivery: Exp… October 04, 2013
    ## 5 AEARCTR-0000010 Free DFS - Intervention to fight anemia and … May 15, 2013    
    ## 6 AEARCTR-0000011 Pricing Experiment - Intervention to fight a… May 15, 2013
