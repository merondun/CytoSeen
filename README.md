![CytoSeen](/examples/figs/Logo.png)

Simple tool for assessing technical repeatability and batch effects using Bismark coverage files. 

> What's it for?!

* Assessing missingness across shared sites among Bismark coverage files.
* Validating technical replicates by ensuring intra-sample variation is higher than inter-sample variation.
* Quantifying batch effects with PCA and R<sup>2</sup>. 

## Methylation Reproducibility Reports

- [Installation](#installation)
- [Usage](#usage)
- [Outputs](#outputs)
- [FAQ](#faq)

### Installation

Installation (only unix & win-64 currently tested) can be installed via conda (~100 Mb). You can also run the chunks directly in R using the template script in `/cytoseen/` using some commonly distributed packages installable via CRAN (see FAQ). 

```
conda config --add channels heritabilities
conda create -n CytoSeen cytoseen -y
conda activate CytoSeen

# Now, we must copy the R and Rmd files into the bin directory so they are in our path within this env
# These paths will need to be modified on win and osx!
cp $CONDA_PREFIX/lib/python3.11/site-packages/cytoseen/render_report.R $CONDA_PREFIX/bin/
cp $CONDA_PREFIX/lib/python3.11/site-packages/cytoseen/cytoseen.Rmd $CONDA_PREFIX/bin/
chmod +x $CONDA_PREFIX/bin/*
```

:warning: **IMPORTANT!!!**: the command `cytoseen` needs access to two scripts: `render_report.R` and `cytoseen.Rmd`. Ensure you `cp` them above to add them into your environment's path, **or** simply copy them to your current directory as it will scan for those files. It was successful you can `render_report.R -h`. 


You should be able to call the program with:

``` 
cytoseen -h
usage: cytoseen [-h] [--version] --info INFO [--min_cov MIN_COV] [--max_cov MAX_COV] [--max_missing MAX_MISSING] --outdir OUTDIR --covdir COVDIR

Create CytoSeen 5mC Reproducibility Reports

options:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
  --info INFO           Path to the info CSV file
  --min_cov MIN_COV     Minimum site coverage [default=20]
  --max_cov MAX_COV     Maximum site coverage [default=200]
  --max_missing MAX_MISSING
                        Maximum missing data threshold [default=0.1]
  --outdir OUTDIR       Output directory
  --covdir COVDIR       Directory to the Bismark coverage files
```

A minimal example can be run with 6 `cov.gz` files by cloning the repo and running it:

```
git clone git@github.com:merondun/CytoSeen.git
cd CytoSeen
cytoseen --info examples/info.csv --outdir examples/outdir/ --covdir examples/biscov/
```

Which should print some markdown syntax and create the output files in the outdir directory. 



### Usage

**INPUTS:**

This script analyzes variation between technical replicates and batches, so *a separate .cov.gz is necessary for each library*. This means that you will need to align and extract your files for each replicate individually. This is tested with CpG data, but will work with any `.cov.gz` formatted inputs. 

The script takes a simple input table with details of each libray - **comma separated**. The four columns with headers are:

* `biscov`: file names for each library [Bismark cov.gz](https://felixkrueger.github.io/Bismark/bismark/methylation_extraction/) file. This file has format (chr, pos, pos, methylation percent, read count methylated, read count unmethylated).  
* `runid`: library ID. This is a distinct ID given to each library, synonymous with a 'RUN' ID from the NCBI.  
* `bioid`: sample ID. This ID is a sample ID which connects runs, synonymous with 'BIOSAMPLE' from the NCBI.
* `batchid`: an arbitrary batch ID. Examples could be plate ID, sequencing facility, library protocol. 


```
head info.csv
biscov,runid,bioid,batchid
B2V8841_BL_ADL_F__SRR25143965.cov.gz,B2V8841_BL_ADL_F__SRR25143965,B2V8841_BL_ADL_F,Kiel_EMSeq_MspI
B2X3498_BL_ADL_M__SRR25143972.cov.gz,B2X3498_BL_ADL_M__SRR25143972,B2X3498_BL_ADL_M,Kiel_EMSeq_MspI
B2X3498_BL_ADL_M__SRR25143974.cov.gz,B2X3498_BL_ADL_M__SRR25143974,B2X3498_BL_ADL_M,Kiel_EMSeq_MspI
B2X4825_BL_ADL_M__SRR25143963.cov.gz,B2X4825_BL_ADL_M__SRR25143963,B2X4825_BL_ADL_M,Kiel_EMSeq_MspI
B2X4825_BL_ADL_M__SRR10606826.cov.gz,B2X4825_BL_ADL_M__SRR10606826,B2X4825_BL_ADL_M,Kiel_RRBS
B2X4825_BL_ADL_M__SRR10606843.cov.gz,B2X4825_BL_ADL_M__SRR10606843,B2X4825_BL_ADL_M,Kiel_RRBS
```

The command `CytoSeen -h` accepts these arguments: 

* `--info` library information (above)
* `--min_cov` Minimum read coverage to retain a site for each library. (`countM + countU >= min_cov`) [default: 20]
* `--max_cov` Maximum read coverage to retain a site for each library. (`countM + countU <= min_cov`) [default: 200]
* `--max_missing` Drop sites where more than this proportion of libraries are missing.  [default: 0.1]
* `--outdir` directory for output files. 
* `--covdir` directory with the bismark coverage files. 


**Logic for filtering missingness:**

```
# Calculate the threshold for retaining sites, always round down 
threshold <- floor(nrow(t) * (1 - params$max_missing))

# Identify sites that appear in at least the threshold number of samples
retained_sites <- site_counts[N >= threshold, site]
retained_data <- combined_data[site %in% retained_sites]
```

**Logic for assessing inter- and intra- correlations:**

1. Subset a runid (replicate) for a bioid (biological replicate)
2. Subset one of its other runids
3. Subset n=1,000 sites, and calculate Spearman's correlation using `complete.obs`
4. Use `boot` to repeat the above using 1,000 resampling events
5. Grab an inter-sample comparison (bioid != bioid), and `boot` the correlation again on 1,000 sites
6. Repeat for each runid, and for each bioid 

**Logic for PCA/RDA:**

1. Taking the overlapping, coverage filtered sites filtered for missingness
2. Impute missing values using [missMDA](http://factominer.free.fr/missMDA/)
3. Create a PCA and output 4 PCs using [FactoMineR](http://factominer.free.fr/)
4. Use [vegan](https://github.com/vegandevs/vegan) for RDA: `(PC_axes ~ batchid)`


### Outputs

The script does not produce any console output during processing because it renders the output into a RMarkdown `.html` report. After the script finishes successfully, all the results will be compiled into a [CytoSeen.html](examples/CytoSeen.html) report within the specified output directory (open in browser). 

* `CytoSeen.html` Compiled report, open in any browser. 

All figures and tables are also saved as respective `.pdf` and `.csv`:

* `site_counts.pdf` How many sites are retained based on the number of libraries sequenced at coverage and missingness thresholds. 

![site_counts](/examples/figs/site_counts.png)

* `missingness.pdf` How do variable missngness thresholds impact site retainment? 

![missingnes](/examples/figs/missingness.png)

* `Sample_Missingness.csv` How many sites are retained for each library after coverage filters? 

| runid                         | bioid            | batchid         | filtered_sites |
| ----------------------------- | ---------------- | --------------- | -------------- |
| B2X4825_BL_ADL_M__SRR25143963 | B2X4825_BL_ADL_M | Kiel_EMSeq_MspI | 934326         |
| B2X3498_BL_ADL_M__SRR25143974 | B2X3498_BL_ADL_M | Kiel_EMSeq_MspI | 898362         |
| B2V8841_BL_ADL_F__SRR25143965 | B2V8841_BL_ADL_F | Kiel_EMSeq_MspI | 450185         |

* `Intersample_Correlations.csv` For each sample (biosample), what are the 95% CI correlations compared to other samples? 

| bioid            | inter_lower | inter_upper |
| ---------------- | ----------- | ----------- |
| B2V8841_BL_ADL_F | 0.887912    | 0.912634    |
| B2X3498_BL_ADL_M | 0.849319    | 0.887922    |

* `Intrasample_Correlations.csv` For each replicated sample (biosample), what are the intra-sample correlations, for each replicate? 

:exclamation: If any of your bioid do not have a replicate, they will have 'NA' in this `.csv`, and the report will generate some warnings about non-finite values

| bioid            | runid_pair                                                  | batchid_1       | batchid_2       | intra_lower | intra_upper | BatchComparison                 |
| ---------------- | ----------------------------------------------------------- | --------------- | --------------- | ----------- | ----------- | ------------------------------- |
| B2V8841_BL_ADL_F | B2V8841_BL_ADL_F__SRR25143965-B2V8841_BL_ADL_F__SRR25143965 | Kiel_EMSeq_MspI | Kiel_EMSeq_MspI | NA          | NA          | Kiel_EMSeq_MspI-Kiel_EMSeq_MspI |
| B2X3498_BL_ADL_M | B2X3498_BL_ADL_M__SRR25143972-B2X3498_BL_ADL_M__SRR25143974 | Kiel_EMSeq_MspI | Kiel_EMSeq_MspI | 0.902422    | 0.926456    | Kiel_EMSeq_MspI-Kiel_EMSeq_MspI |
| B2X4825_BL_ADL_M | B2X4825_BL_ADL_M__SRR25143963-B2X4825_BL_ADL_M__SRR10606826 | Kiel_EMSeq_MspI | Kiel_RRBS       | 0.918279    | 0.936742    | Kiel_EMSeq_MspI-Kiel_RRBS       |
| B2X4825_BL_ADL_M | B2X4825_BL_ADL_M__SRR10606826-B2X4825_BL_ADL_M__SRR10606843 | Kiel_RRBS       | Kiel_RRBS       | 0.953668    | 0.964198    | Kiel_RRBS-Kiel_RRBS             |

* `run_correlations.pdf` Within each replicated sample, how do intra-sample (replicates) compare to inter-sample correlations? 

![run_correlations](/examples/figs/run_correlations.png)

* `batch_correlations.pdf` How correlated are intra-sample libraries across batches? 

![batch_correlations](/examples/figs/batch_correlations.png)

* `overall_correlations.pdf` Overall, what are correlations within and between biosamples? 

![overall_correlations](/examples/figs/overall_correlations.png)

* `pc_scores_imputed.csv` PC scores (axes 1 - 4) for individuals. Missing data was imputed with missMDA.

| runid                         | PC    | value    | batchid         |
| ----------------------------- | ----- | -------- | --------------- |
| B2V8841_BL_ADL_F__SRR25143965 | Dim.1 | 152.4433 | Kiel_EMSeq_MspI |
| B2V8841_BL_ADL_F__SRR25143965 | Dim.2 | 201.7434 | Kiel_EMSeq_MspI |
| B2V8841_BL_ADL_F__SRR25143965 | Dim.3 | -99.7554 | Kiel_EMSeq_MspI |
| B2V8841_BL_ADL_F__SRR25143965 | Dim.4 | 78.40086 | Kiel_EMSeq_MspI |
| B2X3498_BL_ADL_M__SRR25143972 | Dim.1 | 136.2201 | Kiel_EMSeq_MspI |

* `pcas_batch.pdf` PCs 1 - 4, color-coded by batch. 

![pcas_batch](/examples/figs/pcas_batch.png)

* `rda_imputed.pdf` RDA quantifying the effects of batch on overall variation.

![rda_imputed](/examples/figs/rda_imputed.png)

* `BSobj_methylationcounts.csv.gz` a gzipped long-form table with the filtered sites, in this format:

```
chr,pos,reads,countM,runid
NW_019780497.1,1972,5,3,B2V8841_BL_ADL_F__SRR25143965
NW_019780497.1,2007,5,5,B2V8841_BL_ADL_F__SRR25143965
NW_019780497.1,2010,5,4,B2V8841_BL_ADL_F__SRR25143965
NW_019783468.1,1554,51,32,B2V8841_BL_ADL_F__SRR25143965
```

For downstream analyses, you can also create a BSobj from this object, e.g. as used in the package [DSS](https://www.bioconductor.org/packages/release/bioc/vignettes/DSS/inst/doc/DSS.html), with this:

```
methdat <- fread(BSobj_methylationcounts.csv.gz) %>% as.data.frame

# Create a list to store each sample's data 
libs <- NULL

# Loop through libraries..
for (library in unique(methdat$runid)) {
  
  libdat <- methdat[grepl(library,methdat$runid),][,1:4]
  
  cat('Saved to variable: ',library,'\n')
  # Assign it to a new variable
  assign(paste0(library),var)
  libs <- unique(c(libs,library))
  
}

#get names
bslibs <- mget(libs)

#create object
BSobj = makeBSseqData(bslibs,c(libs))
BSobj
```

### Help & Contact

The primary `.Rmd` script can be easily debugged in RStudio and run line-by-line using the `/cytoseen/cytoseen_Rtemplate.R` script. 

For any problems or enhancement suggestions please open an issue or contact Justin Merondun: heritabilites [at] gmail.com


### FAQ

* You can run code chunks in R directly with the `/cytoseen/cytoseen_Rtemplate.R` script, which relies on the following packages. Easily installable with conda/mamba:

```
mamba create -n CytoSeen r-tidyverse r-boot r-missmda r-ggpubr r-data.table r-rmarkdown r-knitr r-r.utils r-vegan r-optparse
```


* It is recommended to use the conda environment for R package version control, but here is a sessionInfo from a version that works:

```
sessionInfo()
R version 4.3.2 (2023-10-31)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Ubuntu 22.04.3 LTS

Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3 
LAPACK: /usr/lib/x86_64-linux-gnu/openblas-pthread/libopenblasp-r0.3.20.so;  LAPACK version 3.10.0

locale:
 [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
 [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8    LC_PAPER=en_US.UTF-8       LC_NAME=C                 
 [9] LC_ADDRESS=C               LC_TELEPHONE=C             LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       

time zone: Etc/UTC
tzcode source: system (glibc)

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
 [1] vegan_2.6-6.1     lattice_0.22-6    permute_0.9-7     viridisLite_0.4.2 FactoMineR_2.10   ggpubr_0.6.0     
 [7] missMDA_1.19      boot_1.3-30       data.table_1.15.2 lubridate_1.9.3   forcats_1.0.0     stringr_1.5.1    
[13] dplyr_1.1.4       purrr_1.0.2       readr_2.1.5       tidyr_1.3.1       tibble_3.2.1      ggplot2_3.5.1    
[19] tidyverse_2.0.0  

loaded via a namespace (and not attached):
 [1] tidyselect_1.2.1     fastmap_1.2.0        TH.data_1.1-2        digest_0.6.35        rpart_4.1.23        
 [6] timechange_0.3.0     estimability_1.5     lifecycle_1.0.4      cluster_2.1.6        multcompView_0.1-10 
[11] survival_3.6-4       magrittr_2.0.3       compiler_4.3.2       rlang_1.1.3          tools_4.3.2         
[16] utf8_1.2.4           knitr_1.45           ggsignif_0.6.4       htmlwidgets_1.6.4    scatterplot3d_0.3-44
[21] abind_1.4-5          multcomp_1.4-25      withr_3.0.0          nnet_7.3-19          grid_4.3.2          
[26] fansi_1.0.6          jomo_2.7-6           xtable_1.8-4         colorspace_2.1-0     mice_3.16.0         
[31] emmeans_1.10.2       scales_1.3.0         iterators_1.0.14     MASS_7.3-60          flashClust_1.01-2   
[36] cli_3.6.2            mvtnorm_1.2-5        generics_0.1.3       rstudioapi_0.15.0    tzdb_0.4.0          
[41] minqa_1.2.7          splines_4.3.2        parallel_4.3.2       vctrs_0.6.5          glmnet_4.1-8        
[46] Matrix_1.6-5         sandwich_3.1-0       carData_3.0-5        car_3.1-2            hms_1.1.3           
[51] rstatix_0.7.2        mitml_0.4-5          ggrepel_0.9.5        foreach_1.5.2        glue_1.7.0          
[56] nloptr_2.0.3         pan_1.9              codetools_0.2-20     DT_0.33              stringi_1.8.4       
[61] gtable_0.3.5         shape_1.4.6.1        lme4_1.1-35.3        munsell_0.5.1        pillar_1.9.0        
[66] htmltools_0.5.8.1    R6_2.5.1             doParallel_1.0.17    backports_1.5.0      leaps_3.1           
[71] broom_1.0.6          Rcpp_1.0.12          coda_0.19-4          nlme_3.1-164         mgcv_1.9-1          
[76] xfun_0.41            zoo_1.8-12           pkgconfig_2.0.3     
```