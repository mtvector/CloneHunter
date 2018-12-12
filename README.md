# R Package - CloneHunter
Hacked to change the sequences of barcode recognition and allow for variable length barcodes.

This is a wrapped package of the above workflow with additional checks on the Celltag Library sequences. This package have a dependency on R version (R >= 3.5.0). This can be used as an alternative approach for this pipeline.

(You might need to install devtools to be able to install from Github first)
```r
install.packages("devtools")
```
Install the package from GitHub.
```r
library("devtools")
devtools::install_github("mtvector/CloneHunter")
```
Load the package
```r
library("CloneHunter")
```

## Assessment of CellTag Library Complexity via Sequencing
In the first section, we would like to evaluate the CellTag library complexity using sequencing. Following is an example using the sequencing data we generated in lab for pooled CellTag library V2. 
### 1. Read in the fastq sequencing data and extract the CellTags
#### Note: This function is still under construction for extension to accept BAM file as input file. Currently it only works with FASTQ files.
```r
# Read in data file that come with the package
fpath <- system.file("extdata", "V2-1_S2_L001_R1_001.fastq", package = "CloneHunter")
output.path <- "./celltag_extracted_v2-1_r1.txt"
# Extract the CellTags
extracted.cell.tags <- CellTagExtraction(fastq.bam.input = fpath, celltag.version = "v2", extraction.output.filename = output.path, save.fullTag = FALSE, save.onlyTag = FALSE)
```
The extracted CellTag - `extracted.cell.tags` variable - contains a list of two vectors as following.

|First Vector-`extracted.cell.tags[[1]]`|Second Vector-`extracted.cell.tags[[1]]` |
|:-------------------------------------:|:---------------------------------------:|
|Full CellTag with conserved regions    |8nt CellTag region                       |

### 2. Count the CellTags and sort based on occurrence of each CellTag
```r
# Count the occurrence of each CellTag
cell.tag.count <- as.data.frame(table(extracted.cell.tags[[2]]), stringsAsFactors = F)
# Sort the CellTags in descending order of occurrence
cell.tag.count.sort <- cell.tag.count[order(-cell.tag.count$Freq), ]
colnames(cell.tag.count.sort) <- c("CellTag", "Count")
```

### 3. Generation of whitelist for this CellTag library
Here are are generating the whitelist for this CellTag library - CellTag V2. This will remove the CellTags with an occurrence number below the threshold. The threshold (using 90th percentile as an example) is determined: floor[(90th quantile)/10]. The percentile can be changed while calling the function. Occurrence scatter plots are saved under the `output.dir`, which could be further used to determine the percentile for each different CellTag library.
```r
whitelisted.cell.tag <- CellTagWhitelistFiltering(count.sorted.table = cell.tag.count.sort, percentile = 0.9, output.dir="./", output.file = "my_favourite_v1.csv", save.output = TRUE)
```
The generated whitelist for each library can be used to filter and clean the single-cell CellTag UMI matrices.

## Clone Calling
In this section, we are presenting an alternative approach that utilizes this package that we established to carry out clone calling with single-cell CellTag UMI count matrices. In this pipeline below, we are using a subset dataset generated from the full data (Full data can be found here: https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99915). Briefly, in our lab, we reprogram mouse embryonic fibroblasts (MEFs) to induced endoderm progenitors (iEPs). This dataset is a single-cell dataset that contains cells collected from different time points during the process. This subset is a part of the first replicate of the data. It contains cells collected at Day 28 with three different CellTag libraries - V1, V2 & V3. 

### 1. Read in the single-cell CellTag UMI count matrix
These matrices can be obtained from the first part of this tutorial. In this pipeline below, the matrices were saved as .Rds files. It will need to be changed when saving the matrices into different data types.
```r
# Read the RDS file
dt.mtx.path <- system.file("extdata", "hf1.d28.prefiltered.Rds", package = "CloneHunter")
sc.celltag <- readRDS(dt.mtx.path)
# Change the rownames
rnm <- sc.celltag$Cell.BC
sc.celltag <- sc.celltag[,-1]
rownames(sc.celltag) <- rnm
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
metric.filter.sc.data.2 <- MetricBasedFiltering(whitelisted.celltag.data = whitelist.sc.data.v2, cutoff = 2, comparison = "greater")
```
### 7. Last check of metric plots
```r
metric.p3 <- MetricPlots(celltag.data = metric.filter.sc.data.2)
print(paste0("Mean CellTags Per Cell: ", metric.p3[[1]]))
print(paste0("Mean CellTag Frequency across Cells: ", metric.p3[[2]]))
```
If it looks good, proceed to the following steps to carry out clone calling via Jaccard analysis and hierarchical clustering.
### 8. Jaccard Analysis
This calculates pairwise Jaccard similarities among cells using the filtered CellTag UMI count matrix. This will generate a Jaccard similarity matrix and plot a correlation heatmap with cells ordered by hierarchical clustering. The matrix and plot will be saved in the current working directory.
```r
jac.mtx <- JaccardAnalysis(whitelisted.celltag.data = metric.filter.sc.data.2)
```
### 9. Clone Calling
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

