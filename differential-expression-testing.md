---
title: 'Differential Expression Testing'
teaching: 60
exercises: 20
---

:::::::::::::::::::::::::::::::::::::: questions 

- What is the purpose of differential expression testing in bioinformatics?
- Can Moran's I algorithm independently identify region-specific differential expressions that align with results obtained from expert annotations?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify differentially expressed genes across different layers using expert annotations.
- Utilize Moran's I algorithm to find spatially variable genes.
- Explore the correlation between genes identified through expert annotations and those detected by Moran's I algorithm.

::::::::::::::::::::::::::::::::::::::::::::::::



## Introduction to Differential Expression Testing

Differential expression testing is crucial in bioinformatics for identifying genes that show significant differences in expression across different samples or groups. 
This method helps find genes that are upregulated or downregulated in specific contexts, providing insights into biological functions and disease mechanisms.

## Moran's I Statistic

Moran's I is a measure used to assess spatial autocorrelation in data, indicating whether similar values are clustered, dispersed, or random. 
In bioinformatics, it's applied to detect genes whose expression patterns exhibit clear spatial structure, aiding in understanding spatially localized biological processes.

## Data Preparation


``` r
plot_filter_st <- filter_st[,!is.na(filter_st$layer_guess)]
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

``` r
plot_filter_st$Layers <- plot_filter_st$layer_guess
unique_clusters <- unique(plot_filter_st$Layers)
num_clusters <- length(unique_clusters)
palette <- carto_pal(num_clusters, "Safe")
names(palette) <- unique_clusters
```

## Differential Expression Analysis

### Differential Expression Using Expert's Annotation

Based on the exprerts brain layers annotation, as it is indicated here:


``` r
plot_filter_st <- filter_st[,!is.na(filter_st$layer_guess)]
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

``` r
plot_filter_st$Layers <- plot_filter_st$layer_guess

unique_clusters <- unique(plot_filter_st$Layers)
num_clusters <- length(unique_clusters)

palette <- carto_pal(num_clusters, "Safe")
names(palette) <- unique_clusters

p <- SpatialDimPlot(plot_filter_st, group.by = 'Layers', cols=palette) +
  theme(legend.position = "right")

print(p)
```

<img src="fig/differential-expression-testing-rendered-layers-1.png" style="display: block; margin: auto;" />

We identify genes that are upregulated in each brain region in comparison to other regions.


``` r
Idents(filter_st) <- "layer_guess"
brain2 <- FindAllMarkers(filter_st, assay = "SCT", only.pos = TRUE, min.pct = 0.25, logfc.threshold = 0.25)
```

``` output
Calculating cluster Layer3
```

``` output
For a (much!) faster implementation of the Wilcoxon Rank Sum Test,
(default method for FindMarkers) please install the presto package
--------------------------------------------
install.packages('devtools')
devtools::install_github('immunogenomics/presto')
--------------------------------------------
After installation of presto, Seurat will automatically use the more 
efficient implementation (no further action necessary).
This message will be shown once per session
```

``` output
Calculating cluster Layer1
```

``` output
Calculating cluster WM
```

``` output
Calculating cluster Layer5
```

``` output
Calculating cluster Layer6
```

``` output
Calculating cluster Layer2
```

``` output
Calculating cluster Layer4
```

### Spatial Differential Expression Using Moran's I

We identify the genes whose expression patterns exhibit clear spatial structure using Moran's I algorithm.


``` r
brain <- FindSpatiallyVariableFeatures(filter_st, assay = "SCT", features = VariableFeatures(filter_st)[1:1000], selection.method = "moransi")
```

``` output
Computing Moran's I
```

``` warning
Warning in dist(x = pos): NAs introduced by coercion
```

## Correlation of Differnetially Expressed Genes in each Brain Region and genes with highset Moran's I value.

### Heatmap of Differential Expression


``` r
# Unique clusters
unique_clusters <- unique(brain2$cluster)

# Create list of data frames filtered by cluster
cluster_data_frames <- lapply(setNames(unique_clusters, unique_clusters), function(cluster) {
  brain2 %>% filter(cluster == !!as.character(cluster))
})

# Get and sort 'MoransI_observed' values
wer_sorted <- brain@assays[["SCT"]]@meta.features %>%
  arrange(desc(MoransI_observed)) %>%
  slice_head(n = 100)

# Initialize p-value adjustment matrix
p_val_adj_matrix <- matrix(1, nrow = nrow(wer_sorted), ncol = length(cluster_data_frames), 
                           dimnames = list(rownames(wer_sorted), names(cluster_data_frames)))

# Fill the matrix with adjusted p-values
for (i in rownames(wer_sorted)) {
  for (j in seq_along(cluster_data_frames)) {
    df <- cluster_data_frames[[j]]
    if (i %in% df$gene) {
      p_val_adj_value <- df$p_val_adj[df$gene == i]
      p_val_adj_matrix[i, j] <- p_val_adj_value
    }
  }
}

# Generate and save heatmap
g <- pheatmap(p_val_adj_matrix, 
              cluster_rows = TRUE, 
              cluster_cols = TRUE, 
              display_numbers = FALSE, 
              color = colorRampPalette(c("navy", "white", "firebrick3"))(50), 
              main = "Heatmap of DE p-values of spatially DE genes ")
print(g)
```

<img src="fig/differential-expression-testing-rendered-heatmap-de-1.png" style="display: block; margin: auto;" />

The heatmap visualization reveals a key finding of our analysis: genes displaying the highest Moran's I values show distinct expression patterns that align with specific brain regions identified through expert annotations. 
This observation underscores the spatial correlation of gene expression, highlighting its potential relevance in understanding regional brain functions and pathologies.

:::::::::::::::::::::::::::::::::: keypoints

- Differential expression testing pinpoints genes with significant expression variations across samples, helping to decode biological and disease mechanisms. 
- Moran's I statistic is applied to reveal spatial autocorrelation in gene expression, critical for examining spatially dependent biological activities.
- Moran's I algorithm effectively identifies genes expressed in anatomically distinct regions, as validated from the correlation analysis with the DE genes from the annotated regions.

:::::::::::::::::::::::::::::::::::::::::::::


