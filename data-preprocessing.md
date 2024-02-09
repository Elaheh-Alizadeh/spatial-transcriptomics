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
#source("https://raw.githubusercontent.com/smcclatchy/spatial-transcriptomics/main/code/spatial_utils.R")
```

We will then load the libraries that we need for this lesson.

```r
#library(here)
#library(tidyverse)
#library(Seurat)
```

Note that the [here](https://here.r-lib.org/) library helps you to find your files by taking
care of the absolute path.

## Load Raw and Filtered Spatial Expression Data

We will use the [Load10X_Spatial](https://www.rdocumentation.org/packages/Seurat/versions/5.0.1/topics/Load10X_Spatial) 
function from the [Seurat](https://satijalab.org/seurat/) package to read in the 
spatial transcription data. This is the data which you downloaded in the setup section.

First, we will read in the raw data for sample 151508.

```r
raw_st = Load10X_Spatial(data.dir = "data/151508", filename = "151508_raw_feature_bc_matrix.h5")
```

If you did not see any error messages, then the data loaded in and you should see an
"raw_st" object in your "Environment" tab on the right.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: instructor

If the data does not load in correctly, verify that the students used the 
mode = "wb" argument in download.file() duing the Setup. We have found that
Windows users have to use this.

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::


Let's look at the "raw_st" object.

```r
raw_st
```

```output
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

Next, we will load in the filtered data. Use the code above and look in
a file browser to identify the "filtered" file for sample 151508.

::::::::::::::::::::::::::::::::::::: challenge 

## Challenge 1: Read in filtered HDF5 file for sample 151508.

Open a file browser and navigate to "Desktop/spatialRNA/data/151508". Can you 
find an HDF5 file (with an ".h5" suffix) that haw the word "filtered" in it?
If so, read that file in and assign it to a variable called "filter_st".


:::::::::::::::::::::::: solution 

## Solution
 
```r
filter_st = Load10X_Spatial(data.dir = "data/151508", filename = "151508_filtered_feature_bc_matrix.h5")
```

:::::::::::::::::::::::::::::::::
:::::::::::::::::::::::::::::::::

Once you have the filtered data loaded in, look at the object.

```r
filter_st
```

```output
An object of class Seurat 
33538 features across 4384 samples within 1 assay 
Active assay: Spatial (33538 features, 0 variable features)
 1 layer present: counts
 1 image present: slice1
```

The raw and filtered data both have 33,538 genes. But the filtered data has fewer
spots. The raw data had 4,992 spots and the filtered data has 4,384 spots.

`![H & E slide of sample 151508](episodes/fig/tissue_lowres_image.png){alt='H & E slide of sample 151508'}`

#########################
### DMG: STOPPED HERE ###
#########################








## Figures

You can include figures generated from R Markdown:


```r
pie(
  c(Sky = 78, "Sunny side of pyramid" = 17, "Shady side of pyramid" = 5), 
  init.angle = 315, 
  col = c("deepskyblue", "yellow", "yellow3"), 
  border = FALSE
)
```

<div class="figure" style="text-align: center">
<img src="fig/data-preprocessing-rendered-pyramid-1.png" alt="pie chart illusion of a pyramid"  />
<p class="caption">Sun arise each and every morning</p>
</div>
Or you can use pandoc markdown for static figures with the following syntax:

`![optional caption that appears below the figure](figure url){alt='alt text for
accessibility purposes'}`

![You belong in The Carpentries!](https://raw.githubusercontent.com/carpentries/logo/master/Badge_Carpentries.svg){alt='Blue Carpentries hex person logo with no text.'}

## Math

One of our episodes contains $\LaTeX$ equations when describing how to create
dynamic reports with {knitr}, so we now use mathjax to describe this:

`$\alpha = \dfrac{1}{(1 - \beta)^2}$` becomes: $\alpha = \dfrac{1}{(1 - \beta)^2}$

Cool, right?

::::::::::::::::::::::::::::::::::::: keypoints 

- Use `.md` files for episodes when you want static content
- Use `.Rmd` files for episodes when you need to generate output
- Run `sandpaper::check_lesson()` to identify any issues with your lesson
- Run `sandpaper::build_lesson()` to preview your lesson locally

::::::::::::::::::::::::::::::::::::::::::::::::
