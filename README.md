# R Package - CellTagR
This is a wrapped R package of the workflow (https://github.com/morris-lab/CellTagWorkflow) with additional assessment of the complexity of the Celltag Library sequences. This package have a dependency on R version (R >= 3.5.0). This can be used as an alternative approach for this pipeline. For details regarding development and usage of CellTag, please refer to the following papaer - (<<< PAPER SOURCE >>>)

(You might need to install devtools to be able to install from Github first)
```r
install.packages("devtools")
```
Install the package from GitHub.
```r
library("devtools")
devtools::install_github("morris-lab/CellTagR")
```
Load the package
```r
library("CellTagR")
```

## Assessment of CellTag Library Complexity via Sequencing
In the first section, we would like to evaluate the CellTag library complexity using sequencing. Following is an example using the sequencing data we generated in lab for pooled CellTag library V2. 
### 1. Read in the fastq sequencing data and extract the CellTags
```r
# Read in data file that come with the package
fpath <- system.file("extdata", "V2-1_R1.zip", package = "CloneHunter")
extract.dir <- "."
# Extract the dataset
unzip(fpath, overwrite = FALSE, exdir = ".")
full.fpath <- paste0(extract.dir, "/", "V2-1_S2_L001_R1_001.fastq")
# Set up the CellTag Object
test.obj <- CellTagObject(object.name = "v2.whitelist.test", fastq.bam.directory = full.fpath)
# Extract the CellTags
test.obj <- CellTagExtraction(celltag.obj = test.obj, celltag.version = "v2")
```
The extracted CellTag will be stored as attribute (fastq.full.celltag & fastq.only.celltag) in the result object with the following format. 

<<<<<<<<<<<<<<<<<<< ADD FORMAT HERE >>>>>>>>>>>>>>>>>>

### 2. Count the CellTags and sort based on occurrence of each CellTag
```r
# Count and Sort the CellTags in descending order of occurrence
test.obj <- AddCellTagFreqSort(test.obj)
# Check the stats
test.obj@celltag.freq.stats
```

### 3. Generation of whitelist for this CellTag library
Here are are generating the whitelist for this CellTag library - CellTag V2. This will remove the CellTags with an occurrence number below the threshold. The threshold (using 90th percentile as an example) is determined: floor[(90th quantile)/10]. The percentile can be changed while calling the function. A plot of CellTag reads will be plotted and it can be used to further choose the percentile. If the output directory is offered, whitelist files will be stored in the provided directory. Otherwise, whitelist files will be saved under the same directory as the fastq files with name as <CellTag Version Number>_whitelist.csv (Example: v2_whitelist.csv). 

```r
# Generate the whitelist
test.obj <- CellTagWhitelistFiltering(celltag.obj = test.obj, percentile = 0.9, output.dir = NULL)
```
The generated whitelist for each library can be used to filter and clean the single-cell CellTag UMI matrices.

## Single-Cell CellTag Extraction and Quantification
In this section, we are presenting an alternative approach that utilizes this package to carry out CellTag extraction, quantification, and generation of UMI count matrices. This can be also accomplished via the workflow supplied - https://github.com/morris-lab/CellTagWorkflow. 
#### Note: Using the package could be slow for the extraction part. For reference, it took approximately an hour to extract from a 40Gb BAM file using a maximum of 8Gb of memory.

### 1. Download the BAM file 
Here we would follow the same step as in https://github.com/morris-lab/CellTagWorkflow to download the a BAM file from the Sequence Read Archive (SRA) server. Again, this file is quite large. Hence, it might take a while to download. The file can be downloaded using wget in terminal as well as in R.
```r
# bash
wget https://sra-download.ncbi.nlm.nih.gov/traces/sra65/SRZ/007347/SRR7347033/hf1.d15.possorted_genome_bam.bam
```
OR
```r
download.file("https://sra-download.ncbi.nlm.nih.gov/traces/sra65/SRZ/007347/SRR7347033/hf1.d15.possorted_genome_bam.bam", "./hf1.d15.bam")
```

### 2. Extract the CellTags from BAM file
In this step, we will extract the CellTag information from the BAM file, which contains information including cell barcodes, CellTag and Unique Molecular Identifiers (UMI). The result generated from this extraction will be a data table containing the following information. The result will then be saved into the slot "bam.parse.rslt" in the object in the following format.

|Cell Barcode|Unique Molecular Identifier|CellTag Motif|
|:----------:|:-:|:---------:|
|Cell.BC|UMI|Cell.Tag|
```r
# Set up the CellTag Object
bam.test.obj <- CellTagObject(object.name = "bam.cell.tag.obj", fastq.bam.directory = "./hf1.d15.bam")
# Extract the CellTag information
bam.test.obj <- CellTagExtraction(bam.test.obj, celltag.version = "v1")
# Check the bam file result
head(bam.test.obj@bam.parse.rslt[["v1"]])
```

### 3. Quantify the CellTag UMI Counts and Generate UMI Count Matrices
In this step, we will quantify the CellTag UMI counts and generate the UMI count matrices. This function will take in two inputs, including the barcode tsv file generated by 10X and celltag object processed from Step 2. The barcode tsv file can be either filtered or raw. **However, note that using the raw barcodes file could result in a requirement of large memory for using this function**. If filtered barcodes files are used, **only cell barcodes that appear in the filtered barcode file** will be preserved. The result will also be saved as a *dgCMatrix* in a slot - "raw.count" - under the object. At the same time, a initial celltag statistics will be saved as another slot under the object. The matrix will be in the format as following.

||CellTag Motif 1|CellTag Motif 2|\<all tags detected\>|CellTag Motif N|
|:----------:|:-:|:---------:|:--:|:--:|
|Cell.BC|Motif 1|Motif 2|\<all tags detected\>|Motif N|

```r
# Generate the sparse count matrix
bam.test.obj <- CellTagMatrixCount(celltag.obj = bam.test.obj, barcodes.file = "./barcodes.tsv")
# Check the dimension of the raw count matrix
dim(bam.test.obj@raw.count)
```

The generated CellTag UMI count matrices can then be used in the following steps for clone identification.

## Single-cell CellTag UMI Count Matrix Processing
In this section, we are presenting an alternative approach that utilizes this package that we established to carry out clone calling with single-cell CellTag UMI count matrices. In this pipeline below, we are using a subset dataset generated from the full data (Full data can be found here: https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99915). Briefly, in our lab, we reprogram mouse embryonic fibroblasts (MEFs) to induced endoderm progenitors (iEPs). This dataset is a single-cell dataset that contains cells collected from different time points during the process. This subset is a part of the first replicate of the data. It contains cells collected at Day 15 with three different CellTag libraries - V1, V2 & V3. 

### 1. Read in the single-cell CellTag UMI count matrix
This object is what we have generated from the above steps using bam files. As mentioned before, bam files take a long time to process. Hence, in this repository, we include a sample object saved as .Rds file from the previous steps, in which raw count matrix is included in the slot - "raw.count"
```r
# Read the RDS file and get the object
dt.mtx.path <- system.file("extdata", "hf1.d15.demo.Rds", package = "CellTagR")
bam.test.obj <- readRDS(dt.mtx.path)
```

### (RECOMMENDED) Optional Step: CellTag Error Correction
***NOTE:*** If CellTag error correction was **NOT** planned to be performed, skip this step and move to the *Step 2 - binarization*, in which the raw matrix will be used. Otherwise, before binarization and additional filtering, we will carry out the following error correction step via Starcode, from which collapsed matrix will be used further for binarization.

In this step, we will identify CellTags with similar sequences and collapse similar CellTags to the centroid CellTag. For more information and installation, please refer to starcode software - https://github.com/gui11aume/starcode. Briefly, starcode clusters DNA sequences based on the Levenshtein distances between each pair of sequences, from which we collapse similar CellTag sequences to correct for potential errors occurred during single-cell RNA-sequencing process. Default maximum distance from starcode was used to cluster the CellTags.

### I. Prepare for the data to be collapsed
First, we will prepare the data to the format that could be accepted by starcode. This function accepts two inputs including the CellTag object with raw count matrix generated and a path to where to save the output text file. The output will be a text file with each line containing one sequence to collapse with others. In this function, we concatenate the CellTag with cell barcode and use the combined sequences as input to execute Starcode. The file to be used for Starcode will be stored under the provided directory.
```r
# Generating the collapsing file
bam.test.obj <- CellTagDataForCollapsing(celltag.obj = bam.test.obj, output.file = "~/Desktop/collapsing.txt")
```

### II. Run Starcode to cluster the CellTag
Following the instruction for Starcode, we will run the following command to generate the result from starcode.
```r
./starcode -s --print-clusters ~/Desktop/collapsing.txt > ~/Desktop/collapsing_result.txt
```

### III. Extract information from Starcode result and collapse similar CellTags
With the collapsed results, we will regenerate the CellTag x Cell Barcode matrix. The collpased matrix will be stored in a slot - "collapsed.count" - in the CellTag object. This function takes two inputs including the CellTag Object to modify and the path to th result file from collapsing.
```r
# Recount and generate collapsed matrix
bam.test.obj <- CellTagDataPostCollapsing(celltag.obj = bam.test.obj, collapsed.rslt.file = "~/Desktop/collapsing_rslt.txt")
# Check the dimension of this collapsed count.
head(bam.test.obj@collapsed.count)
```

### 2. Binarize the single-cell CellTag UMI count matrix
Here we would like to binarize the count matrix to contain 0 or 1, where 0 indicates no such CellTag found in a single cell and 1 suggests the existence of such CellTag. The suggested cutoff that marks existence or absence is at least 2 counts per CellTag per Cell. For details, please refer to the paper - https://www.nature.com/articles/s41586-018-0744-4
```r
# Calling binarization
binary.sc.celltag <- SingleCellDataBinarization(sc.celltag, 2)
```

### 3. Metric plots to facilitate for additional filtering
We then generate scatter plots for the number of total celltag counts in each cell and the number each tag across all cells. These plots could help us further in filtering and cleaning the data.
```r
metric.p <- MetricPlots(celltag.data = binary.sc.celltag)
print(paste0("Mean CellTags Per Cell: ", metric.p[[1]]))
print(paste0("Mean CellTag Frequency across Cells: ", metric.p[[2]]))
```

### 4. Apply the whitelisted CellTags generated from assessment
##### Note: This filters the single-cell data based on the whitelist of CellTags one by one. By mean of that, if three CellTag libraries were used, the following commands need to be executed for 3 times and result matrices can be further joined (Example provided).

Based on the whitelist generated earlier, we filter the UMI count matrix to contain only whitelisted CelTags.
```r
whitelist.sc.data.v2 <- SingleCellDataWhitelist(binary.sc.celltag, whitels.cell.tag.file = "./my_favourite_v2_1.csv")
```
For all three CellTags,
```r
##########
# Only run if this sample has been tagged with more than 1 CellTags libraries
##########
## NOT RUN
whitelist.sc.data.v1 <- SingleCellDataWhitelist(binary.sc.celltag, whitels.cell.tag.file = "./my_favourite_v1.csv")
whitelist.sc.data.v2 <- SingleCellDataWhitelist(binary.sc.celltag, whitels.cell.tag.file = "./my_favourite_v2_1.csv")
whitelist.sc.data.v3 <- SingleCellDataWhitelist(binary.sc.celltag, whitels.cell.tag.file = "./my_favourite_v3.csv")
```
For each version of CellTag library, it should be processed through the following steps one by one to call clones for different pooled CellTag library.

### 5. Check metric plots after whitelist filtering
Recheck the metric as similar as Step 3
```r
metric.p2 <- MetricPlots(celltag.data = whitelist.sc.data.v2)
print(paste0("Mean CellTags Per Cell: ", metric.p2[[1]]))
print(paste0("Mean CellTag Frequency across Cells: ", metric.p2[[2]]))
```

### 6. Additional filtering
#### Filter out cells with more than 20 CellTags
```r
metric.filter.sc.data <- MetricBasedFiltering(whitelisted.celltag.data = whitelist.sc.data.v2, cutoff = 20, comparison = "less")
```
#### Filter out cells with less than 2 CellTags
```r
metric.filter.sc.data.2 <- MetricBasedFiltering(whitelisted.celltag.data = metric.filter.sc.data, cutoff = 2, comparison = "greater")
```
### 7. Last check of metric plots
```r
metric.p3 <- MetricPlots(celltag.data = metric.filter.sc.data.2)
print(paste0("Mean CellTags Per Cell: ", metric.p3[[1]]))
print(paste0("Mean CellTag Frequency across Cells: ", metric.p3[[2]]))
```
If it looks good, proceed to the following steps to call the clones.

## Clone Calling
### 1. Jaccard Analysis
This calculates pairwise Jaccard similarities among cells using the filtered CellTag UMI count matrix. This will generate a Jaccard similarity matrix and plot a correlation heatmap with cells ordered by hierarchical clustering. The matrix and plot will be saved in the current working directory.
```r
jac.mtx <- JaccardAnalysis(whitelisted.celltag.data = metric.filter.sc.data.2)
```
### 2. Clone Calling
Based on the Jaccard similarity matrix, we can call clones of cells. A clone will be selected if the correlations inside of the clones passes the cutoff given (here, 0.7 is used. It can be changed based on the heatmap/correlation matrix generated above). Using this part, a list containing the clonal identities of all cells and the count information for each clone. The tables will be saved in the given directory and filename.

##### Clonal Identity Table `result[[1]]`

|clone.id|cell.barcode|
|:-------:|:------:|
|Clonal ID|Cell BC |

##### Count Table `result[[2]]`
|Clone.ID|Frequency|
|:------:|:-------:|
|Clonal ID|The cell number in the clone|

```r
Clone.result <- CloneCalling(Jaccard.Matrix = jac.mtx, output.dir = "./", output.filename = "clone_calling_result.csv", correlation.cutoff = 0.7)
```

## Optional: CellTag Error Correction
In this step, we will identify CellTags with similar sequences and collapse similar CellTags to the centroid CellTag. For more information, please refer to starcode software - https://github.com/gui11aume/starcode. Briefly, starcode clusters DNA sequences based on the Levenshtein distances between each pair of sequences, from which we collapse similar CellTag sequences to correct for potential errors occurred during single-cell RNA-sequencing process. Default maximum distance from starcode was used to cluster the CellTags.

### 1. Prepare for the data to be collapsed
First, we will prepare the data to the format that could be accepted by starcode. This function accepts three inputs including the unfiltered single-cell data, the single-cell full UMI count matrix and the output csv file to save to. The output will be a data frame containing the CellTag information with their corresponding cell barcode and UMI counts. In this function, we concatenate the CellTag with cell barcode and use the combined sequences as input to execute Starcode. The file to be used for Starcode will be stored under the same directory as the output file and with the name provided and the suffix of "collapse.txt".
```r
# Expecting matrix with each column = a cell, each row = a celltag
sc.celltag.t <- t(sc.celltag)
colnames(sc.celltag.t) <- rownames(sc.celltag)
# Generating the collapsing files
collapse.df <- CellTagDataForCollapsing(sc.cell.tag, sc.celltag.t, "./my_favoriate.csv")
```

### 2. Run Starcode to cluster the CellTag
Following the instruction for Starcode, we will run the following command to generate the result from starcode.
```r
./starcode -s --print-clusters ./my_favoriate_collapse.txt > ./collapsing_result.txt
```

### 3. Extract information from Starcode result and collapse similar CellTags
With the collapsed results, we will regenerate the CellTag x Cell Barcode matrix. The output will be a matrix that contain the combined counts and collapsed CellTags. Also, the output will be saved under the output file directory given.
```r
collapsed.mtx <- CellTagDataPostCollapsing(sc.cell.tag, "./collapsing_result.txt", "./my_favoriate.csv", "./collapsed_matrix.RDS")
```
##### You can now use the collaped matrix to continue the single-cell data processing section (Step 2 - binarization and on) with the collapsed matrix.
