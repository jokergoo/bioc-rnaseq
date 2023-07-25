---
source: Rmd
title: Differential expression analysis
teaching: XX
exercises: XX
---


```{.warning}
Warning: replacing previous import 'S4Arrays::makeNindexFromArrayViewport' by
'DelayedArray::makeNindexFromArrayViewport' when loading 'SummarizedExperiment'
```

::::::::::::::::::::::::::::::::::::::: objectives

- Explain the steps involved in a differential expression analysis.
- Explain how to perform these steps in R, using DESeq2.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- What are the steps performed in a typical differential expression analysis? 
- How does one interpret the output of DESeq2?


::::::::::::::::::::::::::::::::::::::::::::::::::




## Differential expression inference

A major goal of RNA-seq data analysis is the quantification and statistical inference of systematic changes between experimental groups or conditions (e.g., treatment vs. control, timepoints, tissues). This is typically performed by identifying genes with differential expression pattern using between- and within-condition variability and thus requires biological replicates (multiple sample of the same condition).
Multiple software packages exist to perform differential expression analysis. Comparative studies have shown some concordance of differentially expressed (DE) genes, but also variability between tools with no tool consistently outperforming all others (see [Soneson and Delorenzi, 2013](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-14-91)). 
In the following we will explain and conduct differential expression analysis using the [DESeq2](https://bioconductor.org/packages/release/bioc/html/DESeq2.html) 
software package. The [edgeR](https://bioconductor.org/packages/release/bioc/html/edgeR.html) package implements similar methods following the same main assumptions about count data. Both packages show a general good and stable performance with comparable results.  

## The DESeqDataSet

To run `DESeq2` we need to represent our count data as object of the `DESeqDataSet` class.
The `DESeqDataSet` is an extension of the `SummarizedExperiment` class (see section [Importing and annotating quantified data into R](../episodes/03-import-annotate.Rmd) ) that stores a *design formula* in addition to the count assay(s) and feature (here gene) and sample metadata.
The *design formula* expresses the variables which will be used in modeling. These are typically the variable of interest (group variable) and other variables you want to account for (e.g., batch effect variables). Objects of the `DESeqDataSet` class can be build from [count matrices](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#countmat), [SummarizedExperiment objects](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#se), [transcript abundance files](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#tximport) or [htseq count files](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#htseq).

### Load libraries


```r
suppressPackageStartupMessages({
    library(SummarizedExperiment)
    library(DESeq2)
    library(ggplot2)
    library(ExploreModelMatrix)
    library(cowplot)
    library(ComplexHeatmap)
})
```

### Load data


```r
se <- readRDS("data/GSE96870_se.rds")
```

### Create DESeqDataSet


```r
dds <- DESeq2::DESeqDataSet(se[, se$tissue == "Cerebellum"],
                            design = ~ sex + time)
```

```{.warning}
Warning in DESeq2::DESeqDataSet(se[, se$tissue == "Cerebellum"], design = ~sex
+ : some variables in design formula are characters, converting to factors
```


::::::::::::::::::::::::::::::::::::: instructor
The function to generate a `DESeqDataSet` needs to be adapted depending on the input type, e.g, 


```r
#From SummarizedExperiment object
ddsSE <- DESeqDataSet(se, design = ~ cell + dex)
```

```{.error}
Error in DESeqDataSet(se, design = ~cell + dex): all variables in design formula must be columns in colData
```

```r
#From count matrix
dds <- DESeqDataSetFromMatrix(countData = assays(se)$counts,
                              colData = colData(se),
                              design = ~ condition)
```

```{.error}
Error in DESeqDataSet(se, design = design, ignoreRank): all variables in design formula must be columns in colData
```

:::::::::::::::::::::::::::::::::::::::::::::::::


## Normalization

 `DESeq2` and `edgeR` make the following assumptions:

- most genes are not differentially expressed
- the probability of a read mapping to a specific gene is the same for all samples within the same group

As shown in the [previous section](../episodes/04-exploratory-qc.Rmd) on exploratory data analysis the total counts of a sample (even from the same condition) depends on the library size (total number of reads sequenced). To compare the variability of counts from a specific gene between and within groups we first need to account for library sizes and compositional effects. 
Recall the `estimateSizeFactors()` function from the previous section:


```r
dds <- estimateSizeFactors(dds)
```

::::::::::::::::::::::::::::::::::::: instructor

*DESeq2* uses the __"Relative Log Expression” (RLE)__ method to calculate sample-wise *size factors* tĥat account for read depth and library composition.
*edgeR* uses the __“Trimmed Mean of M-Values” (TMM)__ method to account for library size differences and compositional effects. *edgeR*'s *normalization factors* and *DESeq2*'s *size factors* yield similar results, but are not equivalent theoretical parameters.

:::::::::::::::::::::::::::::::::::::::::::::::::


## Statistical modeling

`DESeq2` and `edgeR` model RNA-seq counts as __negative binomial__ distribution to account for a limited number of replicates per group, a mean-variance dependency (see [exploratory data analysis](../episodes/04-exploratory-qc.Rmd)) and a skewed count distribution. 

### Dispersion

The within-group variance of the counts for a gene following a negative binomial distribution with mean $\mu$ can be modeled as:

$var = \mu + \theta \mu^2$

$\theta$ represents the gene-specific __dispersion__, a measure of variability or spread in the data. As a second step, we need to estimate gene-wise dispersions to get the expected within-group variance and test for group differences. Good dispersion estimates are challenging with a few sample per group only. Thus, information from genes with similar expression pattern are "borrowed". Gene-wise dispersion estimates are *shrinked* towards center values of the observed distribution of dispersions. With `DESeq2` we can get dispersion estimates using the `estimateDispersions()` function.
We can visualize the effect of *shrinkage* using `plotDispEsts()`:


```r
dds <- estimateDispersions(dds)
```

```{.output}
gene-wise dispersion estimates
```

```{.output}
mean-dispersion relationship
```

```{.output}
final dispersion estimates
```

```r
plotDispEsts(dds)
```

<img src="fig/05-differential-expression-rendered-unnamed-chunk-8-1.png" style="display: block; margin: auto;" />

### Testing

Finally, we can use the `nbinomWaldTest()`function of `DESeq2` to fit a *generalized linear model (GLM)* and compute *log2 fold changes* (synonymous with "GLM coefficients", "beta coefficients" or "effect size"). Log2 fold changes are scaled by their standard error and compared to a standard Normal distribution to calculate *Wald test* p-values. 



```r
dds <- nbinomWaldTest(dds)
```


::::::::::::::::::::::::::::::::::::: callout

Standard differential expression analysis as performed above is wrapped into a single function, `DESeq()`. Running the first code chunk is equivalent to running the second one:


```r
dds <- DESeq(dds)
```


```r
dds <- estimateSizeFactors(dds)
dds <- estimateDispersions(dds)
dds <- nbinomWaldTest(dds)
```

:::::::::::::::::::::::::::::::::::::::::::::::::



## Explore results for specific contrasts

The `results()` function can be used to extract gene-wise statistics, such as log2 fold changes and (adjusted) p-values. By default these statistics will be based on the __last variable__ (reference variable) of the design formular. For comparisons `DESeq2` defines so called contrasts, which refer to the factor levels of the reference variable. By default the first factor level in alphabetical order will be used as *reference level* and the *last level* will be used for comparison. You can explicitly specify the contrast by using the `contrast` or `name` argument of the `results()` function. Names of all available contrasts can be accessed using `resultsNames()`.


::::::::::::::::::::::::::::::::::::: challenge 

How can the terms __reference level__ and __last level__ be interpreted? What will be the default __contrast__, __reference level__ and __last level__ when running `results(dds)` for the above example?

*Hint: Check the design formular used to build the object.*

:::::::::::::::::::::::: solution 

The __reference level__ can be interpreted as your experiments __"control group"__ and the __last level__ as __treatment or intervention group__. 

In the above example the last variable of the design formular is `time`. 
The __reference level__ (first in alphabetical order) is `Day0` and the __last level__ is `Day8` 


```r
levels(dds$time)
```

```{.output}
[1] "Day0" "Day4" "Day8"
```

No worries, if you had difficulties to identify the default contrast the output of the `results()` function explicitly states the contrast it is referring to (see below)!

:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

To explore the output of the `results()` function we can use the `summary()` function and order results by significance (p-value).



```r
## Day 8 vs Day 0
resTime <- results(dds, contrast = c("time", "Day8", "Day0"))
summary(resTime)
```

```{.output}

out of 32652 with nonzero total read count
adjusted p-value < 0.1
LFC > 0 (up)       : 4472, 14%
LFC < 0 (down)     : 4276, 13%
outliers [1]       : 10, 0.031%
low counts [2]     : 8732, 27%
(mean count < 1)
[1] see 'cooksCutoff' argument of ?results
[2] see 'independentFiltering' argument of ?results
```

```r
head(resTime[order(resTime$pvalue), ])
```

```{.output}
log2 fold change (MLE): time Day8 vs Day0 
Wald test p-value: time Day8 vs Day0 
DataFrame with 6 rows and 6 columns
               baseMean log2FoldChange     lfcSE      stat      pvalue
              <numeric>      <numeric> <numeric> <numeric>   <numeric>
Asl             701.343        1.11733 0.0592541   18.8565 2.59885e-79
Apod          18765.146        1.44698 0.0805186   17.9708 3.30147e-72
Cyp2d22        2550.480        0.91020 0.0554756   16.4072 1.69794e-60
Klk6            546.503       -1.67190 0.1058989  -15.7877 3.78228e-56
Fcrls           184.235       -1.94701 0.1279847  -15.2128 2.90708e-52
A330076C08Rik   107.250       -1.74995 0.1154279  -15.1606 6.45112e-52
                     padj
                <numeric>
Asl           6.21386e-75
Apod          3.94690e-68
Cyp2d22       1.35326e-56
Klk6          2.26086e-52
Fcrls         1.39017e-48
A330076C08Rik 2.57077e-48
```

::::::::::::::::::::::::::::::::::::: instructor
Both of the below ways of specifying the contrast are essentially equivalent.
The `name` parameter can be accessed using `resultsNames()`.


```r
resTime <- results(dds, contrast = c("time", "Day8", "Day0"))
resTime <- results(dds, name = "time_Day8_vs_Day0")
```

:::::::::::::::::::::::::::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::::: challenge 

Explore the DE genes between males and females.

*Hint: Use `resultsNames()` to get the correct contrast.*

:::::::::::::::::::::::: solution 



```r
## Male vs Female
resSex <- results(dds, contrast = c("sex", "Male", "Female"))
summary(resSex)
```

```{.output}

out of 32652 with nonzero total read count
adjusted p-value < 0.1
LFC > 0 (up)       : 53, 0.16%
LFC < 0 (down)     : 71, 0.22%
outliers [1]       : 10, 0.031%
low counts [2]     : 13717, 42%
(mean count < 6)
[1] see 'cooksCutoff' argument of ?results
[2] see 'independentFiltering' argument of ?results
```

```r
head(resSex[order(resSex$pvalue), ])
```

```{.output}
log2 fold change (MLE): sex Male vs Female 
Wald test p-value: sex Male vs Female 
DataFrame with 6 rows and 6 columns
               baseMean log2FoldChange     lfcSE      stat       pvalue
              <numeric>      <numeric> <numeric> <numeric>    <numeric>
Xist         22603.0359      -11.60429  0.336282  -34.5076 6.16852e-261
Ddx3y         2072.9436       11.87241  0.397493   29.8683 5.08722e-196
Eif2s3y       1410.8750       12.62514  0.565216   22.3369 1.62066e-110
Kdm5d          692.1672       12.55386  0.593627   21.1477  2.89566e-99
Uty            667.4375       12.01728  0.593591   20.2451  3.92780e-91
LOC105243748    52.9669        9.08325  0.597624   15.1989  3.59432e-52
                     padj
                <numeric>
Xist         1.16739e-256
Ddx3y        4.81378e-192
Eif2s3y      1.02237e-106
Kdm5d         1.37001e-95
Uty           1.48667e-87
LOC105243748  1.13371e-48
```


:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: callout

### Multiple testing correction

Due to the high number of tests (one per gene) our DE results will contain a substantial number of __false positives__. For example, if we tested 20,000 genes at a threshold of $\alpha = 0.05$ we would expect 1000 significant DE genes with no differential expression.

To account for this expected high number of false positives, we can correct our results for __multiple testing__. By default `DESeq2` uses the [Benjamini-Hochberg procedure](https://link.springer.com/referenceworkentry/10.1007/978-1-4419-9863-7_1215)
to calculate __adjusted p-values__ (padj) for DE results.

:::::::::::::::::::::::::::::::::::::::::::::::::


## Independent Filtering and log-fold shrinkage

We can visualize the results in many ways. A good check is to explore the relationship between log2fold changes, significant results and the genes mean count.
`DESeq2` provides a useful function to do so, `plotMA()`.


```r
plotMA(resTime)
```

<img src="fig/05-differential-expression-rendered-unnamed-chunk-16-1.png" style="display: block; margin: auto;" />

We can see that genes with a low mean count tend to have larger log fold changes.
This is caused by counts from lowly expressed genes tending to be very noisy. 
We can *shrink* the log fold changes of these genes with low mean and high dispersion, as they contain little information.


```r
resTimeLfc <- lfcShrink(dds, coef = "time_Day8_vs_Day0", res = resTime)
```

```{.error}
Error in lfcShrink(dds, coef = "time_Day8_vs_Day0", res = resTime): type='apeglm' requires installing the Bioconductor package 'apeglm'
```

```r
plotMA(resTimeLfc)
```

```{.error}
Error in h(simpleError(msg, call)): error in evaluating the argument 'object' in selecting a method for function 'plotMA': object 'resTimeLfc' not found
```
Shrinkage of log fold changes is useful for visualization and ranking of genes, but for result exploration typically the `indepent_filtering` argument is used to remove lowly expressed genes.

::::::::::::::::::::::::::::::::::::: challenge 

By default `indepent_filtering` is set to `TRUE`. What happens without filtering lowly expressed genes? Use the `summary()` function to compare the results. Most of the lowly expressed genes are not significantly differential expressed (blue in the above MA plots). What could cause the difference in the results then?

:::::::::::::::::::::::: solution 


```r
resTimeNotFiltered <- results(dds,
                              contrast = c("time", "Day8", "Day0"), 
                              independentFiltering = FALSE)
summary(resTime)
```

```{.output}

out of 32652 with nonzero total read count
adjusted p-value < 0.1
LFC > 0 (up)       : 4472, 14%
LFC < 0 (down)     : 4276, 13%
outliers [1]       : 10, 0.031%
low counts [2]     : 8732, 27%
(mean count < 1)
[1] see 'cooksCutoff' argument of ?results
[2] see 'independentFiltering' argument of ?results
```

```r
summary(resTimeNotFiltered)
```

```{.output}

out of 32652 with nonzero total read count
adjusted p-value < 0.1
LFC > 0 (up)       : 4130, 13%
LFC < 0 (down)     : 3962, 12%
outliers [1]       : 10, 0.031%
low counts [2]     : 0, 0%
(mean count < 0)
[1] see 'cooksCutoff' argument of ?results
[2] see 'independentFiltering' argument of ?results
```

Genes with very low counts are not likely to see significant differences typically due to high dispersion. Filtering of lowly expressed genes thus increased detection power at the same experiment-wide false positive rate. 

:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::


## Visualize selected set of genes

The amount of DE genes can be overwhelming and a ranked list of genes can still be hard to interpret with regards to an experimental question. Visualizing gene expression can help to detect expression pattern or group of genes with related functions. We will perform systematic detection of over represented groups of genes in the [next section](../episodes/06-gene-set-analysis.Rmd). Before this visualization can already help us to get a good intuition about what to expect.

We will use transformed data (see [exploratory data analysis](../episodes/04-exploratory-qc.Rmd)) and the top differentially expressed genes for visualization. A heatmap can reveal expression pattern across sample groups (columns) and automatically orders genes (rows) according to their similarity. 



```r
# Transform counts
vsd <- vst(dds, blind = TRUE)

# Get top DE genes
genes <- rownames(head(resTime[order(resTime$pvalue), ], 10))
heatmapData <- assay(vsd)[genes, ]

# Scale counts for visualization
heatmapData <- t(scale(t(heatmapData)))

# Add annotation
heatmapColAnnot <- data.frame(colData(vsd)[, c("time", "sex")])
idx <- order(vsd$time)
heatmapData <- heatmapData[, idx]
heatmapColAnnot <- HeatmapAnnotation(df = heatmapColAnnot[idx, ])

# Plot as heatmap
ComplexHeatmap::Heatmap(heatmapData,
                        top_annotation = heatmapColAnnot,
                        cluster_rows = TRUE, cluster_columns = FALSE)
```

<img src="fig/05-differential-expression-rendered-unnamed-chunk-19-1.png" style="display: block; margin: auto;" />

::::::::::::::::::::::::::::::::::::: challenge 

Check the heatmap and top DE genes. Do you find something expected/unexpected?

*Hint: Google gene names, if you don't know them.*
::::::::::::::::::::::::::::::::::::::::::::::::


:::::::::::::::::::::::::::::::::::::::: keypoints

- With DESeq2, the main steps of a differential expression analysis (size factor estimation, dispersion estimation, calculation of test statistics) are wrapped in a single function: DESeq().
- Independent filtering of lowly expressed genes is often beneficial. 


::::::::::::::::::::::::::::::::::::::::::::::::::


