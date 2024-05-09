---
title: 'Data Preprocessing'
teaching: 10
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- What data files should I expect from the sequencing core?
- Which data preprocessing steps are required to prepare the raw data files for further analysis?
- What software will we use for data preprocessing?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain how to use markdown with the new lesson template
- Demonstrate how to include pieces of code, figures, and nested challenge blocks

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

The sequencing core will deliver a set of files and directories. The complete 
set of files is listed in the table below.

| File Name	 | Description |
|------------|-------------|
| web_summary.html	| Run summary metrics and plots in HTML format |
| cloupe.cloupe	| Loupe Browser visualization and analysis file |
| spatial/	| Folder containing outputs that capture the spatiality of the data. |
| spatial/aligned_fiducials.jpg	| Aligned fiducials QC image |
| spatial/aligned_tissue_image.jpg	| Aligned CytAssist and Microscope QC image. Present only for CytAssist workflow |
| spatial/barcode_fluorescence_intensity.csv	| CSV file containing the mean and standard deviation of fluorescence intensity for each spot and each channel. Present for the fluorescence image input specified by --darkimage |
| spatial/cytassist_image.tiff	| Input CytAssist image in original resolution that can be used to re-run the pipeline. Present only for CytAssist workflow |
| spatial/detected_tissue_image.jpg	| Detected tissue QC image. |
| spatial/scalefactors_json.json	| Scale conversion factors for spot diameter and coordinates at various image resolutions |
| spatial/spatial_enrichment.csv	| Feature spatial autocorrelation analysis using Moran's I in CSV format |
| spatial/tissue_hires_image.png	| Downsampled full resolution image. The image dimensions depend on the input image and slide version |
| spatial/tissue_lowres_image.png	| Full resolution image downsampled to 600 pixels on the longest dimension |
| spatial/tissue_positions.csv	| CSV containing spot barcode; if the spot was called under (1) or out (0) of tissue, the array position, image pixel position x, and image pixel position y for the full resolution image |
| analysis/	| Folder containing secondary analysis data including graph-based clustering and K-means clustering (K = 2-10); differential gene expression between clusters; PCA, t-SNE, and UMAP dimensionality reduction. |
| metrics_summary.csv	| Run summary metrics in CSV format |
| probe_set.csv	| Copy of the input probe set reference CSV file. Present for Visium FFPE and CytAssist workflow |
| possorted_genome_bam.bam	| Indexed BAM file containing position-sorted reads aligned to the genome and transcriptome, annotated with barcode information |
| possorted_genome_bam.bam.bai	| Index for possorted_genome_bam.bam. In cases where the reference transcriptome is generated from a genome with very long chromosomes (>512 Mbp), Space Ranger v2.0+ generates a possorted_genome_bam.bam.csi index file instead. |
| filtered_feature_bc_matrix/	| Contains only tissue-associated barcodes in MEX format. Each element of the matrix is the number of UMIs associated with a feature (row) and a barcode (column). This file can be input into third-party packages and allows users to wrangle the barcode-feature matrix (e.g. to filter outlier spots, run dimensionality reduction, normalize gene expression). |
| **filtered_feature_bc_matrix.h5**	| **Same information as filtered_feature_bc_matrix/ but in HDF5 format.** |
| raw_feature_bc_matrices/	| Contains all detected barcodes in MEX format. Each element of the matrix is the number of UMIs associated with a feature (row) and a barcode (column). |
| **raw_feature_bc_matrix.h5**	| **Same information as raw_feature_bc_matrices/ in HDF5 format.** |
﻿| raw_probe_bc_matrix.h5	| Contains UMI counts of each probe for all detected barcodes in HDF5 format. Only produced when running pipelines for probe-based assays. |
| molecule_info.h5	| Contains per-molecule information for all molecules that contain a valid barcode, valid UMI, and were assigned with high confidence to a gene or protein barcode. This file is required for additional analysis spaceranger pipelines including aggr, targeted-compare and targeted-depth. |

Fortunately, you will not need to look at all of these files. We provide a
brief description for you in case you are curious or need to look at one
of the files for technical reasons.

The two files that you will use are "raw_feature_bc_matrix.h5" and
"filtered_feature_bc_matrix.h5". These files have an "h5" suffix,
which means that they are HDF5 files.
[HDF5](https://www.hdfgroup.org/solutions/hdf5/) is a compressed file format
for storing complex high-dimensional data. "HDF5 stands for "Hierarchical Data 
Formats, version 5". There is an R package designed to read and write HDF5 files 
called [rhdf5](https://bioconductor.org/packages/release/bioc/html/rhdf5.html).
This was one of the packages which you installed during the lesson setup.
  
Briefly, HDF5 organizes data into directories within the compressed file. There
are three "files" within the HDF5 file:

| File Name     | Description |
|---------------|-------------|
| features.csv  | Contains the features (i.e. genes in this case) for each row in the data matrix.|
| barcodes.csv  | Contains the probe barcodes for each spot on the tissue block. |
| matrix.mtx    | Contains the counts for each gene in each spot. Features (e.g. genes) are in rows and barcodes (e.g. spots) are in columns. |

## Set up Environment

Go to the "File" menu and select "Open Project...". Open the "spatialRNA" project which you 
created in the workshop Setup. 

First, we will load in some utility functions to make our lives a bit easier. The 
[source](https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/source) function
reads an R file and runs the code in it. In this case, this will load several useful
functions.


```r
source("https://raw.githubusercontent.com/smcclatchy/spatial-transcriptomics/main/code/spatial_utils.R")
```

We will then load the libraries that we need for this lesson.


```r
suppressPackageStartupMessages(library(tidyverse))
suppressPackageStartupMessages(library(hdf5r))
suppressPackageStartupMessages(library(Seurat))
```

Note that the [here library](https://here.r-lib.org/) helps you to find your files by taking
care of the absolute path.

## Load Raw and Filtered Spatial Expression Data

We will use the [Load10X_Spatial](https://www.rdocumentation.org/packages/Seurat/versions/5.0.1/topics/Load10X_Spatial) 
function from the [Seurat](https://satijalab.org/seurat/) package to read in the 
spatial transcription data. This is the data which you downloaded in the setup section.

First, we will read in the raw data for sample 151508.


```r
raw_st <- Load10X_Spatial(data.dir = "./data/151508", 
                          filename = "151508_raw_feature_bc_matrix.h5",
                          filter.matrix = FALSE)
```

If you did not see any error messages, then the data loaded in and you should see an
"raw_st" object in your "Environment" tab on the right.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: instructor

If the data does not load in correctly, verify that the students used the 
mode = "wb" argument in download.file() during the Setup. We have found that
Windows users have to use this.

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::


Let's look at the "raw_st" object.


```r
raw_st
```

```{.output}
An object of class Seurat 
33538 features across 4992 samples within 1 assay 
Active assay: Spatial (33538 features, 0 variable features)
 1 layer present: counts
 1 image present: slice1
```


The output says that we have 33,538 "features" and 4,992 "samples" with one assay.
"Feature" is a generic term for anything that we measured. In this case, we 
measured gene expression, so each feature is a gene. Each "sample" is one
spot on the spatial slide. So this tissue sample has 33,538 genes assayed
across 4,992 spots.

An experiment may have more than one assay. For example, you may run both RNA
sequencing and chromatin accessibility in the same set of samples. In this case,
we have one assay, which contains the RNA-seq counts for each spot.

There is also a single image called 'slice1' attached to the Seurat object.

Next, we will load in the filtered data. Use the code above and look in
a file browser to identify the "filtered" file for sample 151508.

::::::::::::::::::::::::::::::::::::: challenge 

## Challenge 1: Read in filtered HDF5 file for sample 151508.

Open a file browser and navigate to "Desktop/spatialRNA/data/151508". Can you 
find an HDF5 file (with an ".h5" suffix) that haw the word "filtered" in it?
If so, read that file in and assign it to a variable called "filter_st".


:::::::::::::::::::::::: solution 

## Solution 1
 

```r
filter_st <- Load10X_Spatial(data.dir = "./data/151508", 
                             filename = "151508_filtered_feature_bc_matrix.h5")
```

::::::::::::::::::::::::::::::::::
:::::::::::::::::::::::::::::::::::::::::::::::::::

Once you have the filtered data loaded in, look at the object.


```r
filter_st
```

```{.output}
An object of class Seurat 
33538 features across 4384 samples within 1 assay 
Active assay: Spatial (33538 features, 0 variable features)
 1 layer present: counts
 1 image present: slice1
```


The raw and filtered data both have 33,538 genes. But the filtered data has fewer
spots. The raw data had 4,992 spots and the filtered data has 4,384 spots.

Look at the H & E slide below and notice the grey spots around the border. These
are used by the spatial transcriptomics software to "register" the image.

![H & E slide of sample 151508](fig/tissue_hires_image.png){alt='H & E slide of sample 151508'}

::::::::::::::::::::::::::::::::::::: challenge 

## Challenge 2: How does the computer know how to orient the image?.

Look carefully at the spots in the H & E image above. Are the spots symmetric?
Is there anything different about the spots that might help a computer to 
assign up/down and left/right to the image?

:::::::::::::::::::::::: solution 

## Solution 2
 
Look at the spots in each corner. In the upper-left, you will see the following
patterns:

![Slide Diagram](fig/tissue_diagram_with_registration_spots.png){alt='Diagram showing tissue and registration spots in corners'}

The patterns in each corner allow the spatial transcriptomics software to 
orient the slide.

:::::::::::::::::::::::::::::::::
:::::::::::::::::::::::::::::::::

## Add Spot Metadata

Next, we will read a file containing information about whether each spot is in 
the background or the tissue. This file does not contain column names, although
the next version of 
[SpaceRanger 2.0](https://www.10xgenomics.com/support/software/space-ranger/latest/analysis/outputs/spatial-outputs), 
which is used to process the data at the sequencing core, should add column 
names to this file.


```r
tissue_position <- read_csv("./data/151508/spatial/tissue_positions_list.csv",
                            col_names = FALSE, show_col_types = FALSE) %>% 
                     column_to_rownames('X1')
colnames(tissue_position) <- c("in_tissue", 
                               "array_row", 
                               "array_col", 
                               "pxl_row_in_fullres", 
                               "pxl_col_in_fullres")
```

It is important to note that the order of the spots differs between the Seurat 
object and the tissue position file. We need to reorder the tissue positions
to match the Seurat object. We can extract the spot barcodes using the
[Cells()](https://satijalab.org/seurat/reference/cells) function. This is 
named for the earlier versions of Seurat, which processed single cell transcript
data. In this case, we are getting **spot** IDs, even though the function is
called "Cells".


```r
tissue_position <- tissue_position[Cells(raw_st),]
stopifnot(rownames(tissue_position) == Cells(raw_st))
```

Now that we have aligned the barcodes between the Seurat object and the tissue
positions, we can add the tissue positions to the Seurat object's metadata.


```r
raw_st    <- AddMetaData(object = raw_st,    metadata = tissue_position)
filter_st <- AddMetaData(object = filter_st, metadata = tissue_position)
```

Next, we will plot the spot annotation, indicating spots that are in the tissue
in blue and background spots in red.


```r
SpatialPlot(raw_st, group.by = "in_tissue")
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-unnamed-chunk-10-1.png" alt="Histology slide with tissue and background spots labelled"  />
<p class="caption">Spots identified in Tissue and Background</p>
</div>

We expect most of the transcript counts to be in the tissue spots. The Seurat 
object metadata contains a count of the number of transcripts in each spot 
called "nCount_Spatial". Let's plot the counts in the tissue and background
spots.


```r
raw_st@meta.data %>%
  ggplot(aes(as.logical(in_tissue), nCount_Spatial)) +
    geom_boxplot() +
    labs(title = 'Transcript Counts in Tissue and Background',
         x     = 'In Tissue?',
         y     = 'Counts')
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-unnamed-chunk-11-1.png" alt="Boxplot showing lower trancript counts in background area of slide"  />
<p class="caption">Transcript Counts in Tissue and Background</p>
</div>

As expected, we see most of the counts in the tissue spots.

We can also plot the number of genes detected in each spot. Seurat calls genes
"features", so we will plot the "nFeature_Spatial" value. This is stored in the
metadata of the Seurat object.


```r
raw_st@meta.data %>%
  ggplot(aes(as.logical(in_tissue), nFeature_Spatial)) +
    geom_boxplot() +
    labs(title = 'Number of Genes in Tissue and Background',
         x     = 'In Tissue?',
         y     = 'Number of Genes')
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-unnamed-chunk-12-1.png" alt="Boxplot showing lower numbers of genes in background area of slide"  />
<p class="caption">Number of Genes in Tissue and Background</p>
</div>

::::::::::::::::::::::::::::::::::::: challenge 

## Challenge 3: Why might there be transcript counts outside of the tissue boundaries?

We expect transcript counts in the spots which overlap with the tissue section.
What reasons can you think of that might lead transcript counts to occur in the
background spots?

:::::::::::::::::::::::: solution 

## Solution 3
 
1. When the tissue section is lysed, some transcripts may leak out of the cells
and into the background region of the slide.
> DMG: Ask the core.

:::::::::::::::::::::::::::::::::
:::::::::::::::::::::::::::::::::

Up to this point, we have been working with the raw, unfiltered data to show
you how the spots are filtered. However, in most workflows, you will work
directly with the filtered file. From this point forward, we will work with the
filtered data object.

Let's plot the spots in the tissue in the filtered object to verify that it is
only using spots in the tissue.


```r
plot1 <- SpatialDimPlot(filter_st, alpha = c(0, 0)) + 
           NoLegend()
plot2 <- SpatialDimPlot(filter_st) + 
           NoLegend()
plot1 | plot2
```

<img src="fig/data-preprocessing-rendered-unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

## Plot Transcript and Feature Counts

Next, we want to look at the distribution of transcript counts and numbers of
genes in each spot across the tissue. This can be helpful in identifying 
technical issues with the sample processing.
We will use Seurat's 
[SpatialFeaturePlot](https://satijalab.org/seurat/reference/spatialplot) 
function to look at these values. We can color the spots based on the spot
metadata stored in the Seurat object. You can find these column names by looking 
at the "meta.data" slot of the Seurat object.


```r
head(filter_st@meta.data)
```

```{.output}
                      orig.ident nCount_Spatial nFeature_Spatial in_tissue
AAACAAGTATCTCCCA-1 SeuratProject           3627             1847         1
AAACAATCTACTAGCA-1 SeuratProject            956              635         1
AAACACCAATAACTGC-1 SeuratProject            990              716         1
AAACAGAGCGACTCCT-1 SeuratProject           1377              862         1
AAACAGCTTTCAGAAG-1 SeuratProject           3215             1720         1
AAACAGGGTCTATATT-1 SeuratProject           3035             1617         1
                   array_row array_col pxl_row_in_fullres pxl_col_in_fullres
AAACAAGTATCTCCCA-1        50       102               8586               9171
AAACAATCTACTAGCA-1         3        43               2944               5127
AAACACCAATAACTGC-1        59        19               9646               3454
AAACAGAGCGACTCCT-1        14        94               4273               8634
AAACAGCTTTCAGAAG-1        43         9               7728               2772
AAACAGGGTCTATATT-1        47        13               8208               3046
```

To plot the transcript counts, we will use the "nCount_Spatial" column in the
spot metadata.


```r
plot1 <- SpatialDimPlot(filter_st, alpha = c(0, 0)) + 
           NoLegend()
plot2 <- SpatialFeaturePlot(filter_st, features = "nCount_Spatial")
plot1 | plot2
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-unnamed-chunk-15-1.png" alt="Figure showing transcript counts in each spot with varying intensity across the tissue"  />
<p class="caption">Transcript Counts in each Spot</p>
</div>

In this case, we see a band of higher counts running from upper-left to 
lower-right. There are also bands of lower counts above and below this band.
The band in the upper-right corner may be due to the fissure in the tissue. It
is less clear why the expression is low in the lower-left corner.

We can also look at the number of genes detected in each spot using 
"nFeature_Spatial".


```r
plot1 <- SpatialDimPlot(filter_st, alpha = c(0, 0)) + 
           NoLegend()
plot2 <- SpatialFeaturePlot(filter_st, features = "nFeature_Spatial")
plot1 | plot2
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-unnamed-chunk-16-1.png" alt="Figure showing number of genes detected in each spot with varying intensity across the tissue"  />
<p class="caption">Number of Genes in each Spot</p>
</div>

It is difficult to lay out a broad set of rules that will work for all types of
tissues and samples. Some tissues may have homogeneous transcript counts
across the section, while others may show variation in transcript counts due
to tissue structure. For example, in cancer tissue sections, stromal cells tend
to have lower counts than tumor cells and this should be evident in a transcript
count plot. In the brain sample below, we might expect some variation in 
transcript counts in different regions of the brain.

## Conclusion


::::::::::::::::::::::::::::::::::::: keypoints 

- The sequencing core will provide you with an unfiltered and a filtered data
file.
- 

::::::::::::::::::::::::::::::::::::::::::::::::


