How to create a complex cohort
================

# ISB-CGC Community Notebooks

    Title:   How to create a complex cohort
    Author:  Lauren Hagen
    Created: 2019-08-12
    Purpose: More complex overview of creating cohorts
    Notes:   This notebook was adapted from work by Shiela Reynolds, 'How to Create TCGA Cohorts part 3' https://github.com/isb-cgc/examples-Python/blob/master/notebooks/Creating%20TCGA%20cohorts%20--%20part%201.ipynb.

-----

# How to create a complex cohort

This notebook will construct a cohort for a single tumor type based on
data availability, while also taking into consideration annotations
about the patients or samples.

As you’ve seen already, in order to work with BigQuery, the first thing
we need to is load the libraries required for using BigQuery in R:

``` r
library(bigrquery)
library(tidyverse)
```

Just so that this doesn’t get buried in the code below, we are going to
specify our tumor-type of interest here. In TCGA each tumor-type is also
a separate *study* within the TCGA *project*. The studies are referred
to based on the 2-4 letter tumor-type abbreviation. A complete list of
all study abbreviations, with the full study name can be found on this
[page](https://gdc.cancer.gov/resources-tcga-users/tcga-code-tables/tcga-study-abbreviations).
For this particular exercise, we will look at the “Breast invasive
carcinoma” study, abbreviated BRCA:

``` r
studyName = "TCGA-BRCA"
```

More information the the BRCA study can be found
[here](https://portal.gdc.cancer.gov/projects/TCGA-BRCA). In this
notebook, we are going to wind up making use of all of the available
data types, so let’s have a look at the entire **`TCGA_hg38_data_v0`**
dataset:

``` r
project <- 'isb-cgc-02-0001' # Insert your project ID in the ''
theDataset <- 'TCGA_hg38_data_v0'
```

``` r
print("Tables:")
```

    ## [1] "Tables:"

``` r
tables<-list_tables("isb-cgc", theDataset)
tables
```

    ##  [1] "Copy_Number_Segment_Masked"     "Copy_Number_Segment_Masked_r14"
    ##  [3] "DNA_Methylation"                "DNA_Methylation_chr1"          
    ##  [5] "DNA_Methylation_chr10"          "DNA_Methylation_chr11"         
    ##  [7] "DNA_Methylation_chr12"          "DNA_Methylation_chr13"         
    ##  [9] "DNA_Methylation_chr14"          "DNA_Methylation_chr15"         
    ## [11] "DNA_Methylation_chr16"          "DNA_Methylation_chr17"         
    ## [13] "DNA_Methylation_chr18"          "DNA_Methylation_chr19"         
    ## [15] "DNA_Methylation_chr2"           "DNA_Methylation_chr20"         
    ## [17] "DNA_Methylation_chr21"          "DNA_Methylation_chr22"         
    ## [19] "DNA_Methylation_chr3"           "DNA_Methylation_chr4"          
    ## [21] "DNA_Methylation_chr5"           "DNA_Methylation_chr6"          
    ## [23] "DNA_Methylation_chr7"           "DNA_Methylation_chr8"          
    ## [25] "DNA_Methylation_chr9"           "DNA_Methylation_chrX"          
    ## [27] "DNA_Methylation_chrY"           "Protein_Expression"            
    ## [29] "RNAseq_Gene_Expression"         "Somatic_Mutation"              
    ## [31] "Somatic_Mutation_DR10"          "Somatic_Mutation_DR6"          
    ## [33] "Somatic_Mutation_DR7"           "miRNAseq_Expression"           
    ## [35] "miRNAseq_Isoform_Expression"    "tcga_metadata_data_hg38_r14"

In this next code cell, we define an SQL query called **`DNU_patients`**
which finds all patients in the Annotations table which have either been
‘redacted’ or had ‘unacceptable prior treatment’. It will display the
output of the query and then save the data to a tibble.

``` r
# Create Query
DNU_patients_query <- "SELECT
                  case_barcode,
                  category AS categoryName,
                  classification AS classificationName
                FROM
                  `isb-cgc.TCGA_bioclin_v0.Annotations`
                WHERE
                  ( entity_type='Patient'
                    AND (category='History of unacceptable prior treatment related to a prior/other malignancy'
                    OR classification='Redaction' ) )
                GROUP BY
                  case_barcode,
                  categoryName,
                  classificationName
                ORDER BY
                  case_barcode"
# Run the query
DNU_patients <- bq_project_query(project, DNU_patients_query, quiet = TRUE) 
# Create a dataframe with the results from the query
DNU_patients <- bq_table_download(DNU_patients, quiet = TRUE)
# Show the dataframe
summary(DNU_patients)
```

    ##  case_barcode       categoryName       classificationName
    ##  Length:212         Length:212         Length:212        
    ##  Class :character   Class :character   Class :character  
    ##  Mode  :character   Mode  :character   Mode  :character

Now we’re gong to use the query defined previously in a function that
builds a “clean” list of patients in the specified study, with available
molecular data, and without any disqualifying
annotations.

``` r
buildCleanBarcodeList <- function( studyName, bqDataset, barcodeType, DNUList ) {
  print("in buldCleanBarcodeList ... ")
  ULists = list() # List to hold lists of barcodes
  print("  --> looping over data tables: ")
  barcodeField <- barcodeType # set the barcodeField to the barcodeType
  for (t in bqDataset) {
    currTable <- str_c("`isb-cgc.TCGA_hg38_data_v0.",t,"`",sep="") # Set the BigQuery Table
    barcodeField <- barcodeType # Reset the barcodeField for each loop
    # Check if the table is a somatic table and the barcodeType is "sample_barcode" as 
    # the somatic tables use sample_barcode_tumor instead
    if(grepl("Somatic", t) && barcodeField == "sample_barcode") {
      barcodeField <- "sample_barcode_tumor"
    }
    get_barcode_list <- ""
    # Check if the table is a Methlation table as these tables doesn't have the project_short_name column
    if (grepl("Methylation", t)) { 
      # Create the query
      get_barcode_list <- str_c("WITH a AS ( SELECT ",barcodeField, 
                                " FROM ", currTable, " ), b AS ( SELECT ",
                                barcodeField, 
                                ", project_short_name FROM `isb-cgc.TCGA_hg38_data_v0.Copy_Number_Segment_Masked`) SELECT ", 
                                barcodeField, 
                                " FROM ( SELECT a.", barcodeField, " AS ", 
                                barcodeField, 
                                ", b.project_short_name AS project_short_name FROM a JOIN b ON a.", 
                                barcodeField," = b.",barcodeField,
                                ") WHERE project_short_name='", studyName,
                                "' GROUP BY ", barcodeField, 
                                " ORDER BY ", barcodeField, sep = "")
    }
    else {
      get_barcode_list <- str_c("SELECT ", barcodeField, 
                                    " FROM ", currTable,
                                    " WHERE project_short_name='", studyName,
                                    "' GROUP BY ",barcodeField,
                                    " ORDER BY ",barcodeField,sep="")
    }
    # Run the query
    barcode_list <- bq_project_query(project, get_barcode_list, quiet = TRUE)
    # Create a dataframe with the results from the query
    barcode_list <- bq_table_download(barcode_list, quiet = TRUE)
    print(str_c("      ", t, "  --> ", nrow(barcode_list), " unique barcodes.", sep = ""))
    barcodes <- as.list(barcode_list) # make the barcodes a list
    ULists[[t]] <- barcodes # put the list of barcodes into ULists
  }
  print(str_c("  --> we have ", length(ULists), " lists to merge"))
  masterList <- list() # Create a list for the master list of barcodes
  for (a in 1:length(ULists)) {
    for (b in 1:length(ULists[[a]][[1]])) {
      if (!(ULists[[a]][[1]][[b]] %in% masterList)) {
        masterList <- c(masterList, ULists[[a]][[1]][[b]])
      }
    }
  }
  print(str_c("  --> which results in a single list with ", length(masterList), " barcodes"))
  print("  --> removing DNU barcodes")
  cleanList <- list()
  for ( aBarcode in 1:length(masterList) ) {
    if (!(masterList[[aBarcode]] %in% DNUList[[barcodeField]])) {
      cleanList <- c(cleanList, masterList[[aBarcode]])
    }
    else {
      print(str_c("      excluding this barcode ", masterList[[aBarcode]], sep = ""))
    }
  }
  print(str_c("  --> returning a clean list with ", length(cleanList), " barcodes", sep=""))
  return(cleanList)
}
```

``` r
barcodeType <- "case_barcode"
cleanPatientList <- buildCleanBarcodeList(studyName,tables,barcodeType,DNU_patients)
```

    ## [1] "in buldCleanBarcodeList ... "
    ## [1] "  --> looping over data tables: "
    ## [1] "      Copy_Number_Segment_Masked  --> 1096 unique barcodes."
    ## [1] "      Copy_Number_Segment_Masked_r14  --> 1098 unique barcodes."
    ## [1] "      DNA_Methylation  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr1  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr10  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr11  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr12  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr13  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr14  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr15  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr16  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr17  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr18  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr19  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr2  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr20  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr21  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr22  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr3  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr4  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr5  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr6  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr7  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr8  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chr9  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chrX  --> 1095 unique barcodes."
    ## [1] "      DNA_Methylation_chrY  --> 1095 unique barcodes."
    ## [1] "      Protein_Expression  --> 887 unique barcodes."
    ## [1] "      RNAseq_Gene_Expression  --> 1092 unique barcodes."
    ## [1] "      Somatic_Mutation  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR10  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR6  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR7  --> 986 unique barcodes."
    ## [1] "      miRNAseq_Expression  --> 1079 unique barcodes."
    ## [1] "      miRNAseq_Isoform_Expression  --> 1079 unique barcodes."
    ## [1] "      tcga_metadata_data_hg38_r14  --> 1098 unique barcodes."
    ## [1] "  --> we have 36 lists to merge"
    ## [1] "  --> which results in a single list with 1098 barcodes"
    ## [1] "  --> removing DNU barcodes"
    ## [1] "      excluding this barcode TCGA-5L-AAT1"
    ## [1] "      excluding this barcode TCGA-A8-A084"
    ## [1] "      excluding this barcode TCGA-A8-A08F"
    ## [1] "      excluding this barcode TCGA-A8-A08S"
    ## [1] "      excluding this barcode TCGA-A8-A09E"
    ## [1] "      excluding this barcode TCGA-A8-A09K"
    ## [1] "      excluding this barcode TCGA-AR-A2LL"
    ## [1] "      excluding this barcode TCGA-AR-A2LR"
    ## [1] "      excluding this barcode TCGA-BH-A0B6"
    ## [1] "      excluding this barcode TCGA-BH-A1F5"
    ## [1] "      excluding this barcode TCGA-D8-A146"
    ## [1] "  --> returning a clean list with 1087 barcodes"

Now we are going to repeat the same process, but at the sample barcode
level. Most patients will have provided two samples, a “primary tumor”
sample, and a “normal blood” sample, but in some cases additional or
different types of samples may have been provided, and sample-level
annotations may exist that should result in samples being excluded from
most downstream
analyses.

``` r
# there are many different types of annotations that are at the "sample" level
# in the Annotations table, and most of them seem like they should be "disqualifying"
# annotations, so for now we will just return all sample barcodes with sample-level
# annotations
DNUsamples_query <- "SELECT sample_barcode
                      FROM `isb-cgc.TCGA_bioclin_v0.Annotations`
                      WHERE (entity_type='Sample')
                      GROUP BY sample_barcode
                      ORDER BY sample_barcode"
# Run the query
DNUsamples <- bq_project_query(project, DNUsamples_query, quiet = TRUE) 
# Create a dataframe with the results from the query
DNUsamples <- bq_table_download(DNUsamples, quiet = TRUE)
# Show the dataframe
summary(DNUsamples)
```

    ##  sample_barcode    
    ##  Length:113        
    ##  Class :character  
    ##  Mode  :character

And now we can re-use the previously defined function get a clean list
of sample-level barcodes:

``` r
barcodeType <- "sample_barcode"
cleanSampleList <- buildCleanBarcodeList (studyName, tables, barcodeType, DNUsamples)
```

    ## [1] "in buldCleanBarcodeList ... "
    ## [1] "  --> looping over data tables: "
    ## [1] "      Copy_Number_Segment_Masked  --> 2219 unique barcodes."
    ## [1] "      Copy_Number_Segment_Masked_r14  --> 2224 unique barcodes."
    ## [1] "      DNA_Methylation  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr1  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr10  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr11  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr12  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr13  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr14  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr15  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr16  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr17  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr18  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr19  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr2  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr20  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr21  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr22  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr3  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr4  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr5  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr6  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr7  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr8  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chr9  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chrX  --> 1205 unique barcodes."
    ## [1] "      DNA_Methylation_chrY  --> 1205 unique barcodes."
    ## [1] "      Protein_Expression  --> 937 unique barcodes."
    ## [1] "      RNAseq_Gene_Expression  --> 1217 unique barcodes."
    ## [1] "      Somatic_Mutation  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR10  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR6  --> 986 unique barcodes."
    ## [1] "      Somatic_Mutation_DR7  --> 986 unique barcodes."
    ## [1] "      miRNAseq_Expression  --> 1202 unique barcodes."
    ## [1] "      miRNAseq_Isoform_Expression  --> 1202 unique barcodes."
    ## [1] "      tcga_metadata_data_hg38_r14  --> 3298 unique barcodes."
    ## [1] "  --> we have 36 lists to merge"
    ## [1] "  --> which results in a single list with 3339 barcodes"
    ## [1] "  --> removing DNU barcodes"
    ## [1] "      excluding this barcode TCGA-B6-A1KC-01A"
    ## [1] "  --> returning a clean list with 3338 barcodes"

Now we’re going to double-check first that we keep only sample-level
barcodes that correspond to patients in the “clean” list of patients,
and then we’ll also filter the list of patients in case there are
patients with no samples remaining in the “clean” list of samples.

``` r
finalSampleList <- list()

for (aSample in 1:length(cleanSampleList)) {
  aPatient <- str_sub(cleanSampleList[[aSample]], 1, 12)
  if (!aPatient %in% cleanPatientList) {
    print(str_c("     excluding this sample in the final pass: ", cleanSampleList[[aSample]], sep = ""))
  }
  else {
    finalSampleList <- c(finalSampleList, cleanSampleList[[aSample]])
  }
}
```

    ## [1] "     excluding this sample in the final pass: TCGA-5L-AAT1-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-5L-AAT1-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A084-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A084-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08F-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08F-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08S-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08S-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09E-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09E-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09K-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09K-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LL-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LL-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LR-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LR-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A0B6-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A0B6-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A1F5-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A1F5-11A"
    ## [1] "     excluding this sample in the final pass: TCGA-D8-A146-01A"
    ## [1] "     excluding this sample in the final pass: TCGA-D8-A146-10A"
    ## [1] "     excluding this sample in the final pass: TCGA-5L-AAT1-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A084-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08F-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A08S-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09E-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-A8-A09K-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LL-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-AR-A2LR-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A0B6-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-BH-A1F5-01Z"
    ## [1] "     excluding this sample in the final pass: TCGA-D8-A146-01Z"

``` r
print(str_c(" Length of final sample list:", length(finalSampleList), sep = ""))
```

    ## [1] " Length of final sample list:3305"

``` r
finalPatientList = list()
for (aSample in 1:length(finalSampleList)) {
  aPatient <- str_sub(finalSampleList[[aSample]], 1, 12)
  if (!(aPatient %in% finalPatientList)) {
    finalPatientList <- c(finalPatientList, aPatient)
  }
}
print(str_c(" Lenth of final patient list: ", length(finalPatientList), sep = ""))
```

    ## [1] " Lenth of final patient list: 1087"

``` r
for (aPatient in 1:length(cleanPatientList)) {
  if (!finalPatientList[[aPatient]] %in% finalPatientList) {
    print(str_c("     --> patient removed in final pass: ", cleanPatientList[[aPatient]], sep=""))
  }
}
```

We’re also interested in knowing what *types* of samples we have. The
codes for the major types of samples are: - **01** : primary solid tumor
- **02** : recurrent solid tumor - **03** : primary blood derived cancer
- **06** : metastatic - **10** : blood derived normal - **11** : solid
tissue normal

and a complete list of all sample type codes and their definitions can
be found
[here](https://gdc.cancer.gov/resources-tcga-users/tcga-code-tables/sample-type-codes).

``` r
finalSample <- tibble(sampleBarcodes = finalSampleList)

finalSample %>%
  mutate(type = str_match(sampleBarcodes, "-\\d{2}\\D")) %>%
  mutate(type = str_sub(type, 2, 3)) %>%
  group_by(type) %>%
  summarize(count=n())
```

    ## # A tibble: 4 x 2
    ##   type  count
    ##   <chr> <int>
    ## 1 01     2145
    ## 2 06        8
    ## 3 10      991
    ## 4 11      161

Now we are going to create a tibble with all of the sample barcodes and
the associated patient (participant) barcodes so that we can write this
to a BigQuery “cohort” table.

``` r
# Create the tibble
final <- finalSample %>%
  mutate(patientBarcodes = str_sub(sampleBarcodes, 1, 12))

# Check the patientBarcodes
final %>%
  group_by(patientBarcodes) %>%
  summarise(count=n()) %>%
  arrange(desc(count))
```

    ## # A tibble: 1,087 x 2
    ##    patientBarcodes count
    ##    <chr>           <int>
    ##  1 TCGA-A7-A0DB        5
    ##  2 TCGA-A7-A0DC        5
    ##  3 TCGA-A7-A13E        5
    ##  4 TCGA-E2-A15K        5
    ##  5 TCGA-A7-A0CE        4
    ##  6 TCGA-A7-A0D9        4
    ##  7 TCGA-A7-A13D        4
    ##  8 TCGA-A7-A13F        4
    ##  9 TCGA-A7-A13G        4
    ## 10 TCGA-A7-A26E        4
    ## # ... with 1,077 more rows

``` r
# Check the number of unique patientBarcodes  
length(unique(final$patientBarcodes))
```

    ## [1] 1087

As a next step you may want to consider is to put the data into your
GCP. An example of how to move files in and out of GCP and BigQuery can
be found
[here](https://github.com/isb-cgc/Community-Notebooks/tree/master/Notebooks)
along with other tutorial notebooks.
