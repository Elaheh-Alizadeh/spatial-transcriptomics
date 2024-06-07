---
title: 'Deconvolution in Spatial Transcriptomics'
teaching: 60
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions 

- How does deconvolution enhance the analysis of spatial transcriptomics data?
- What are the steps to integrate single-cell RNA-seq data for deconvolution in spatial transcriptomics?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Perform deconvolution to quantify different cell types in spatial transcriptomics spots using single-cell RNA-seq data.
- Understand the process of integrating single-cell RNA-seq data with spatial transcriptomics data.

::::::::::::::::::::::::::::::::::::::::::::::::


``` output
[1] "featselclust_meta_data.rds"      "featselclust_SCT_counts.rds"    
[3] "featselclust_SCT_data.rds"       "featselclust_SCT_scale.data.rds"
[5] "featselclust_seurat_obj.rds"     "featselclust_Spatial_counts.rds"
[1] "featselclust_seurat_obj.rds"
[1] "featselclust_meta_data.rds"      "featselclust_SCT_counts.rds"    
[3] "featselclust_SCT_data.rds"       "featselclust_SCT_scale.data.rds"
[5] "featselclust_Spatial_counts.rds"
[1] "featselclust_meta_data.rds"
[1] "featselclust_SCT_counts.rds"     "featselclust_SCT_data.rds"      
[3] "featselclust_SCT_scale.data.rds" "featselclust_Spatial_counts.rds"
[[1]]
[1] "featselclust" "SCT"          "counts"      

[[2]]
[1] "featselclust" "SCT"          "data"        

[[3]]
[1] "featselclust" "SCT"          "scale.data"  

[[4]]
[1] "featselclust" "Spatial"      "counts"      

                            files   assay      layer
1     featselclust_SCT_counts.rds     SCT     counts
2       featselclust_SCT_data.rds     SCT       data
3 featselclust_SCT_scale.data.rds     SCT scale.data
4 featselclust_Spatial_counts.rds Spatial     counts
[1] 1
                        files assay  layer
1 featselclust_SCT_counts.rds   SCT counts
[1] "featselclust_SCT_counts.rds"
[1] 2
                      files assay layer
2 featselclust_SCT_data.rds   SCT  data
[1] "featselclust_SCT_data.rds"
[1] 3
                            files assay      layer
3 featselclust_SCT_scale.data.rds   SCT scale.data
[1] "featselclust_SCT_scale.data.rds"
[1] 4
                            files   assay  layer
4 featselclust_Spatial_counts.rds Spatial counts
[1] "featselclust_Spatial_counts.rds"
```

## Deconvolution in Spatial Transcriptomics

Spatial transcriptomics (ST) provides valuable insights into the spatial 
distribution of gene expression within tissue sections. However, each spatial 
spot in an ST experiment generally contains multiple cells, leading to mixed 
gene expression signals. Deconvolution is needed to resolve these mixed signals 
into individual cell type contributions, allowing for a more accurate
interpretation of the spatial gene expression data.

## Why Deconvolution is Needed

Mixed cell populations in each spot of an ST dataset can lead to complex, mixed 
gene expression profiles. Deconvolution helps quantify the different cell types 
within each spot, enabling more precise biological insights and downstream analyses
such as cell type-specific expression patterns and interactions.

## Deconvolution with RCTD

RCTD (Robust Cell Type Decomposition) 
[Cable, D. M., et al., Nature Biotechnology, 2021](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8606190)
is a popular deconvolution algorithm designed to handle the complexities of 
mixed cell populations in spatial transcriptomics data. The algorithm uses
single-cell RNA sequencing (scRNA-seq) data as a reference to deconvolute the 
spatial transcriptomics data, estimating the proportions of different cell types
in each spatial spot. The RCTD algorithm models the observed gene expression in 
each spatial spot as a mixture of the gene expression profiles of different cell
types and estimates the proportion of each cell type in each spot using a 
non-negative least squares (NNLS) approach. RCTD can operate in two modes: 
in "doublet" mode it fits at most two cell types per spot, in "full" mode it 
fits potentially all cell types in the reference per spot, and in "multi" mode 
it again fits more than two cell types per spot by extending the "doublet" 
approach. Here, we will use "full" mode.

## Loading Single Cell RNA-Seq Data

First, we load the single-cell RNA-seq data that will serve as a reference for
deconvolution. This data is essential for mapping the gene expression profiles 
from the ST data to specific cell types.


``` r
# Load single-cell RNA-seq data
sc.counts     <- readRDS("data/scRNA-seq/sc_counts.rds")
```

``` error
Error in readRDS("data/scRNA-seq/sc_counts.rds"): unknown input format
```

``` r
# Load cell type annotations
sc.metadata   <- read.delim("data/scRNA-seq/sc_cell_types.tsv")
sc.cell.types <- setNames(factor(sc.metadata$Value), sc.metadata$Name)
shared.cells  <- intersect(colnames(sc.counts), names(sc.cell.types))
```

``` error
Error in eval(expr, envir, enclos): object 'sc.counts' not found
```

``` r
sc.cell.types <- sc.cell.types[shared.cells]
```

``` error
Error in eval(expr, envir, enclos): object 'shared.cells' not found
```

``` r
sc.counts     <- sc.counts[, shared.cells]
```

``` error
Error in eval(expr, envir, enclos): object 'sc.counts' not found
```

## Deconvolution with RCTD

RCTD (Robust Cell Type Decomposition) is a powerful tool that uses scRNA-seq 
reference data to deconvolute the spatial transcriptomics data, thereby 
quantifying the cell types present in each spatial spot.

First, we will create the reference object encapsulating the scRNA-seq data.


``` r
reference <- Reference(sc.counts, sc.cell.types)
```

``` error
Error in eval(expr, envir, enclos): object 'sc.counts' not found
```


Let's write a wrapper function that performs RCTD deconvolution. This will 
facilitate running RCTD on other samples within this dataset.


``` r
run.rctd <- function(reference, st.obj) {
  
  # Get raw ST counts
  st.counts <- GetAssayData(st.obj, assay="Spatial", layer="counts")
  
  # Get the spot coordinates
  st.coords <- st.obj[[]][, c("array_col", "array_row")]
  colnames(st.coords) <- c("x","y")
    
  # Create the RCTD 'puck', representing the ST data
  puck <- SpatialRNA(st.coords, st.counts)

  myRCTD <- create.RCTD(puck, reference, max_cores = 1, keep_reference = TRUE)

  # Run deconvolution -- note that we are using 'full' mode to devolve a spot into 
  # (potentially) all available cell types.
  myRCTD <- suppressWarnings(run.RCTD(myRCTD, doublet_mode = 'full'))
  
  myRCTD
}
```
## Running Deconvolution on Brain Samples

We apply the RCTD wrapper to our spatial transcriptomics data to deconvolute the 
spots and quantify the cell types. This may take ~10 minutes. If you prefer, 
you can load the precomputed results directly.


``` r
# Change this variable to TRUE to load precomputed results, or FALSE to compute
# the results here.
load.precomputed.results <- TRUE

rds.file <- paste0("data/rctd-sample-1.rds")

if(!load.precomputed.results || !file.exists(rds.file)) {

  result_1 <- run.rctd(reference, filter_st)
  # The RCTD file is large. To save space, we will remove the reference counts.
  result_1 <- remove.RCTD.reference.counts(result_1)
  saveRDS(result_1, rds.file)

} else {

  result_1 <- readRDS(rds.file)

}
```

``` error
Error in eval(expr, envir, enclos): object 'reference' not found
```

## Interpreting Deconvolution Results

The deconvolution process outputs the proportion of different cell types in each
spatial spot. Let's write a utility function to extract these proportions from 
the RCTD output. This function is also defined in code/spatial_utils.R.


``` r
format.rctd.output_ <- function(rctd, normalize = FALSE) {

  barcodes <- colnames(rctd@spatialRNA@counts)
  weights  <- rctd@results$weights

  if(normalize) {

    weights <- normalize_weights(weights)

  }

  df   <- as.data.frame(weights)
  df$x <- rctd@spatialRNA@coords$x
  df$y <- rctd@spatialRNA@coords$y
  df
}
```

And now let's see the predicted proportions in our sample:


``` r
props <- format.rctd.output_(result_1, normalize = FALSE)
```

``` error
Error in eval(expr, envir, enclos): object 'result_1' not found
```

``` r
head(props)
```

``` error
Error in h(simpleError(msg, call)): error in evaluating the argument 'x' in selecting a method for function 'head': object 'props' not found
```

Notice that the proportions don't sum exactly to one.


``` r
head(rowSums(select(props, -c(x,y))))
```

``` error
Error in h(simpleError(msg, call)): error in evaluating the argument 'x' in selecting a method for function 'head': error in evaluating the argument 'x' in selecting a method for function 'rowSums': object 'props' not found
```

Let's classify the spot according to the layer type with highest proportion


``` r
props$classification <- apply(select(props, -c(x,y)), 1, function(row) names(row)[which.max(row)])
```

``` error
Error in eval(expr, envir, enclos): object 'props' not found
```

Let's add the deconvolution results to our Seurat object.


``` r
filter_st <- AddMetaData(object = filter_st, metadata =  select(props, -c(x,y)))
```

``` error
Error in eval(expr, envir, enclos): object 'props' not found
```

We can now visualize the predicted layer classifications and compare them alongside
the ground truth annotations that we saw previously.


``` r
g1 <- SpatialDimPlotColorSafe(filter_st[, !is.na(filter_st[[]]$classification)], "classification")
```

``` error
Error in `[.Seurat`(filter_st, , !is.na(filter_st[[]]$classification)): Incorrect number of logical values provided to subset cells
```

``` r
g2 <- SpatialDimPlotColorSafe(filter_st[, !is.na(filter_st[[]]$layer_guess)], "layer_guess")
```

``` warning
Warning: Not validating Centroids objects
Not validating Centroids objects
```

``` warning
Warning: Not validating FOV objects
Not validating FOV objects
Not validating FOV objects
Not validating FOV objects
Not validating FOV objects
Not validating FOV objects
```

``` warning
Warning: Not validating Seurat objects
```

``` output
Scale for fill is already present.
Adding another scale for fill, which will replace the existing scale.
```

``` r
g1 + g2
```

``` error
Error in eval(expr, envir, enclos): object 'g1' not found
```

To be more quantitative, we can compute a confusion matrix comparing the predicted and observed
layers.


``` r
df            <- as.data.frame(table(filter_st[[]]$layer_guess, filter_st[[]]$classification))
```

``` error
Error in table(filter_st[[]]$layer_guess, filter_st[[]]$classification): all arguments must have the same length
```

``` r
colnames(df)  <- c("Annotation", "Prediction", "Freq")
```

``` error
Error in `colnames<-`(`*tmp*`, value = c("Annotation", "Prediction", "Freq": attempt to set 'colnames' on an object with less than two dimensions
```

``` r
df$Annotation <- factor(df$Annotation)
```

``` error
Error in df$Annotation: object of type 'closure' is not subsettable
```

``` r
df$Prediction <- factor(df$Prediction)
```

``` error
Error in df$Prediction: object of type 'closure' is not subsettable
```

``` r
g <- ggplot(data = df, aes(x = Annotation, y = Prediction, fill = Freq)) + geom_tile()
```

``` error
Error in `ggplot()`:
! `data` cannot be a function.
ℹ Have you misspelled the `data` argument in `ggplot()`
```

``` r
g <- g + theme(text = element_text(size = 20))
```

``` error
Error in eval(expr, envir, enclos): object 'g' not found
```

``` r
g
```

``` error
Error in eval(expr, envir, enclos): object 'g' not found
```

Note that there is a fairly strong correlation between the predicted and observed layers,
particularly for the pairs Oligodendrocytes and WM (White Matter), L4 and Layer 4, and L2-3 and Layer 3.

## Summary

Incorporating deconvolution into the spatial transcriptomics workflow enhances the ability to quantify the different cell
types in each spatial spot, providing a richer and more detailed understanding of the tissue architecture. By following
these steps, researchers can ensure that their spatial transcriptomics data is accurately deconvoluted, leading to more
robust and insightful analyses.

:::::::::::::::::::::::::::::::::: keypoints

- Deconvolution enhances spatial transcriptomics by quantifying the different cell types within spatial spots.
- Integrating single-cell RNA-seq data with spatial transcriptomics data is essential for accurate deconvolution.
- The RCTD method is effective for quantifying the proportion of different cell types in spatial transcriptomics data.

:::::::::::::::::::::::::::::::::::::::::::::


