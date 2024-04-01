# tcga-data-nf

![](https://github.com/QuackenbushLab/tcga-data-nf/workflows/build/badge.svg)

Workflow to download and prepare TCGA data.

The workflow divides the process of downloading the data in two steps:
1. Downloading the raw data from GDC and saving the rds/tables needed later
2. Preparing the data. This step includes filtering the data, normalizing it... 
3. Analysis of gene regulatory networks

The idea is that data should be downloaded once, and then prepared for the task at hand.

:warning: For the moment the NetSciDataCompanion package needs to be installed locally. Once the package is ready, it
will be migrated to a public repo and installed from github.

:warning: GitHub is still not allowing the QuackenbushLab to run actions. The docker image is built and pushed 
manually.



## Running the workflow

### Docker

The docker container is hosted on docker.io. 


```docker pull violafanfani/tcga-data-nf:0.0.12```

### Temporarily (until this is published)

1. Clone the nextflow repo 
   ```bash
   git clone git@github.com:QuackenbushLab/tcga-data-nf.git```
2. Build docker locally 
    ```bash
    docker build . -f containers/Dockerfile --build-arg CONDA_FILE=containers/env.base.python.yml --no-cache -t tcga-data-nf:latest```
3. In alternative, check the R packages needed and install them manually.
4. **TO DOWNLOAD**: update the `download_metadata.config` file or pass a new one with the correct parameters (keep track
   of all config files you use)
5. **To PREPARE**: use a correct `expression_prepare_table.csv`
6. Run the workflow from the tcga-data-nf folder    
    ```bash
    nextflow run .  -profile test --resultsDir myresults/ --pipeline download```

### Install or update the workflow

```bash
nextflow pull QuackenbushLab/tcga-data-nf
```

### Run the analysis

```bash
nextflow run QuackenbushLab/tcga-data-nf
```

## Configuration

First there are three main parameters that need to be passed by the user:
1. Pipeline, `$params.pipeline`: the desired pipeline (one between download/prepare/analyze/full)
2. Results dir  `$params.resultsDir`: the parent folder to save the results
3. Batch name   `$params.batchName`: the name of the run

This way all data generated by the pipeline will be found inside the `resultsDir/batchName/` folder.
If nothing is passed, all results will be in the `results/my-batch` folder.

Below we describe all the available pipelines.

### Download

For now the workflow supports downloading expression
from recount3 and mutation data from tcgabiolinks.
An example configuration file is below: add entries for each recount3/mutation dataset you need to download.

```json
params{
  download_metadata{
    expression_recount3{
        tcga_luad{
            project= "LUAD"
            project_home = "data_sources/tcga"
            organism = "human"
            annotation = "gencode_v26"
            type = "gene"
          } 
        gtex_lung{
            ...
          } 
        tcga_brca{
            ...
        }
    }
    mutation_tcgabiolinks{
        tcga_luad{
            project= "TCGA-LUAD"
            data_category = "Simple Nucleotide Variation"
            data_type = "Masked Somatic Mutation"
            download_dir = "gdc_tcga_mutation"
          }
        tcga_brca{
            ...
        }
    }
  }
}
```

#### Testing: testDownload

Download testing can be done by specifying the `--profile testDownload`
parameter. 

The data to be downloaded is specified in the `testdata/download_json.json` file. 

All data will be saved in the `results/test_download` folder. Here is the structure and content one should expect.

```
.
├── downloaded_methylation_metadata.csv
├── downloaded_mutation_metadata.csv
├── downloaded_recount_metadata.csv
├── gtex_pancreas
│   └── recount3
│       └── gtex_pancreas.rds
├── tcga_paad
│   ├── clinical
│   │   ├── clinical_drug_paad.csv
│   │   ├── clinical_follow_up_v4.4_nte_paad.csv
│   │   ├── clinical_follow_up_v4.4_paad.csv
│   │   ├── clinical_nte_paad.csv
│   │   ├── clinical_omf_v4.0_paad.csv
│   │   ├── clinical_patient_paad.csv
│   │   └── clinical_radiation_paad.csv
│   ├── methylation
│   │   ├── tcga_paad_methylation_manifest.txt
│   │   ├── tcga_paad_methylation_metadata.csv
│   │   └── tcga_paad_methylations.txt
│   ├── mutations
│   │   ├── tcga_paad_mutations.txt
│   │   ├── tcga_paad_mutations_metadata.csv
│   │   └── tcga_paad_mutations_pivot.csv
│   └── recount3
│       └── tcga_paad.rds
└── test-log-info.txt
```


### Prepare

For now, only gene expression data needs to be prepared (harmonized with gtex and normalised).

Specify in the configuration file which `recount.metadata_prepare = table.csv` contains the info about the gtex-tcga data. One example is the one in the conf folder.

#### Prepare recount3 expression for TCGA

This workflow uses the NetSciDataCompanion package to manage and filter TCGA data. 

The function `prepare_expression_recount.R` takes the expression rds file and clinical data and generates filtered rds
and txt files. Counts are normalised, non-expressed genes are removed, samples are filtered to remove doublets and
impure samples.

Preprocessing:
 - Normalization: raw counts can be returned as counts/TPM/logTPM/CPM/logCPM
 - Min TPM and frac samples: only genes that have at least a min tpm value in a fraction of the samples are kept. This allows to remove genes that are never expressed
  - purity threshold: cancer samples are filtered based on the purity score
  - Tissue types: separate files can be returned of specific samples. One can pass a specific tissue type, as specified
    by the TCGA GDC conventions (like "Primary Solid Tumor") or one of all/tumor/normal that return either all samples,
    tumor only samples, normal only samples, ignoring the details of the tumor/normal type/.


### Analyze



### Full

The full pipeline downloads, prepare, and analyze samples from TCGA in a single workflow. **We actually recommend to
keep the three steps separate, such that runs are shorter and less prone to errors**.
Since the full pipeline is a combination of the download-prepare-analyze steps, it requires a similar configuration.

```json
params{
  download_metadata{
    expression_recount3{
        tcga_luad{
            project= "LUAD"
            project_home = "data_sources/tcga"
            organism = "human"
            annotation = "gencode_v26"
            type = "gene"
          } 
        gtex_lung{
            ...
          } 
        tcga_brca{
            ...
        }
    }
    mutation_tcgabiolinks{
        tcga_luad{
            project= "TCGA-LUAD"
            data_category = "Simple Nucleotide Variation"
            data_type = "Masked Somatic Mutation"
            download_dir = "gdc_tcga_mutation"
          }
        tcga_brca{
            ...
        }
    }
  }
}
```

## Results

Below is the structure you can expect in the output folder when you run each pipeline. 

For each case we report the output of the testing profile:
- download pipeline: `-profile testDownload`
- prepare pipeline: `-profile testPrepare`
- analyze pipeline: `-profile testAnalyze`
- full pipeline: `-profile test`

### Download


```bash
⚡ tree results/test_download 
results/test_download
├── downloaded_methylation_metadata.csv
├── downloaded_mutation_metadata.csv
├── downloaded_recount_metadata.csv
├── gtex_pancreas
│   └── data_download
│       └── recount3
│           └── gtex_pancreas.rds
└── tcga_paad
    └── data_download
        ├── clinical
        │   ├── clinical_drug_paad.csv
        │   ├── clinical_follow_up_v4.4_nte_paad.csv
        │   ├── clinical_follow_up_v4.4_paad.csv
        │   ├── clinical_nte_paad.csv
        │   ├── clinical_omf_v4.0_paad.csv
        │   ├── clinical_patient_paad.csv
        │   └── clinical_radiation_paad.csv
        ├── methylation
        │   ├── tcga_paad_methylation_manifest.txt
        │   ├── tcga_paad_methylation_metadata.csv
        │   └── tcga_paad_methylations.txt
        ├── mutations
        │   ├── tcga_paad_mutations.txt
        │   ├── tcga_paad_mutations_metadata.csv
        │   └── tcga_paad_mutations_pivot.csv
        └── recount3
            └── tcga_paad.rds


```

### Prepare


This would be the output when you run the `-profile testPrepare` test case

```bash
results/test_prepare
└── tcga_paad
    └── data_prepared
        ├── methylation
        │   ├── tcga_paad_tf_promoter_methylation_clean_all.log
        │   └── tcga_paad_tf_promoter_methylation_raw.csv
        └── recount3
            ├── recount3_pca_tcga_paad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchnull_adjnull.png
            ├── recount3_pca_tcga_paad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchnull_adjnull.png
            ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchnull_adjnull.log
            ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchnull_adjnull.rds
            ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchnull_adjnull.txt
            ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchnull_adjnull.log
            ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchnull_adjnull.rds
            └── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchnull_adjnull.txt

```

### Analyze

This would be the output when you run the `-profile testAnalyze` test case

```bash
results/test_analyze
└── tcga_paad
    └── analysis
        ├── dragon
        │   ├── tcga_paad_dragon.log
        │   ├── tcga_paad_dragon_filtered_expression.csv
        │   ├── tcga_paad_dragon_input.tsv
        │   └── tcga_paad_dragon_mat.tsv
        ├── lioness_dragon
        │   ├── lioness_dragon
        │   │   ├── lioness-dragon-TCGA-2L-AAQL-01A.csv
        │   │   ├── lioness-dragon-TCGA-HV-A5A3-11A.csv
        │   │   ├── lioness-dragon-TCGA-HV-A7OP-01A.csv
        │   │   ├── lioness-dragon-TCGA-IB-A5ST-01A.csv
        │   │   ├── lioness-dragon-TCGA-US-A779-01A.csv
        │   │   └── lioness-dragon-TCGA-YB-A89D-11A.csv
        │   ├── recount3_tcga_paad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchnull_adjnull_shuffle.rds
        │   ├── tcga_paad_lioness_dragon.log
        │   ├── tcga_paad_tf_promoter_methylation_clean.csv
        │   └── tcga_paad_tf_promoter_methylation_clean_shuffle.csv
        ├── panda
        │   ├── panda_tcga_paad.log
        │   └── panda_tcga_paad.txt
        └── panda_lioness
            ├── lioness
            │   ├── lioness.TCGA-2L-AAQL-01A-11R-A38C-07.4.h5
            ...
            └── panda.txt
```

### Full 


Here is an example of the output of the full test.
`tcga_luad` is the uuid of the run , hence inside the `tcga_luad` folder you'll find all relevant files
divided in `data_download`, `data_prepare`, `analysis` folders, that correspond to the steps of the pipeline

Here the general out structure. Below we have the full expected output

```bash
results/test_full
├── analysis
│   ├── dragon
│   ├── lioness_dragon
│   │   └── lioness_dragon
│   └── panda
└── tcga_luad
    ├── analysis
    │   ├── dragon
    │   ├── lioness_dragon
    │   │   └── lioness_dragon
    │   ├── panda
    │   └── panda_lioness
    │       └── lioness
    ├── data_download
    │   ├── clinical
    │   ├── methylation
    │   ├── mutations
    │   └── recount3
    └── data_prepared
        ├── methylation
        └── recount3
```

Complete output folder:

```bash
results/test_full
├── analysis
│   ├── dragon
│   │   ├── tcga_luad_dragon.log
│   │   ├── tcga_luad_dragon_filtered_expression.csv
│   │   ├── tcga_luad_dragon_input.tsv
│   │   └── tcga_luad_dragon_mat.tsv
│   ├── lioness_dragon
│   │   ├── lioness_dragon
│   │   ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
│   │   ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
│   │   ├── tcga_luad_lioness_dragon.log
│   │   └── tcga_luad_tf_promoter_methylation_clean_all.csv
│   └── panda
│       ├── panda_tcga_luad.log
│       └── panda_tcga_luad.txt
├── downloaded_methylation_metadata.csv
├── downloaded_mutation_metadata.csv
├── downloaded_recount_metadata.csv
└── tcga_luad
    ├── analysis
    │   ├── dragon
    │   │   ├── tcga_luad_dragon.log
    │   │   ├── tcga_luad_dragon_filtered_expression.csv
    │   │   ├── tcga_luad_dragon_input.tsv
    │   │   └── tcga_luad_dragon_mat.tsv
    │   ├── lioness_dragon
    │   │   ├── lioness_dragon
    │   │   │   ├── lioness-dragon-TCGA-38-4632-01A.csv
    │   │   │   └── lioness-dragon-TCGA-78-7155-01A.csv
    │   │   ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
    │   │   ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
    │   │   ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
    │   │   ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
    │   │   ├── tcga_luad_lioness_dragon.log
    │   │   └── tcga_luad_tf_promoter_methylation_clean_all.csv
    │   ├── panda
    │   │   ├── panda_tcga_luad.log
    │   │   └── panda_tcga_luad.txt
    │   └── panda_lioness
    │       ├── lioness
    │       │   ├── lioness.TCGA-38-4632-01A-01R-1755-07.3.h5
    │       │   ├── lioness.TCGA-64-5779-01A-01R-1628-07.1.h5
    │       │   ├── lioness.TCGA-69-A59K-01A-11R-A262-07.0.h5
    │       │   └── lioness.TCGA-78-7155-01A-11R-2039-07.2.h5
    │       └── panda.txt
    ├── data_download
    │   ├── clinical
    │   │   ├── clinical_drug_luad.csv
    │   │   ├── clinical_follow_up_v1.0_luad.csv
    │   │   ├── clinical_nte_luad.csv
    │   │   ├── clinical_omf_v4.0_luad.csv
    │   │   ├── clinical_patient_luad.csv
    │   │   └── clinical_radiation_luad.csv
    │   ├── methylation
    │   │   ├── tcga_luad_methylation_manifest.txt
    │   │   ├── tcga_luad_methylation_metadata.csv
    │   │   └── tcga_luad_methylations.txt
    │   ├── mutations
    │   │   ├── tcga_luad_mutations.txt
    │   │   ├── tcga_luad_mutations_metadata.csv
    │   │   └── tcga_luad_mutations_pivot.csv
    │   └── recount3
    │       └── tcga_luad.rds
    └── data_prepared
        ├── methylation
        │   ├── tcga_luad_tf_promoter_methylation_clean_all.csv
        │   ├── tcga_luad_tf_promoter_methylation_clean_all.log
        │   └── tcga_luad_tf_promoter_methylation_raw.csv
        └── recount3
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.log
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.txt
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.log
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
            ├── recount3_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.txt
            ├── recount3_pca_LUAD_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.png
            ├── recount3_pca_LUAD_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.png
            ├── recount3_pca_tcga_luad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.png
            ├── recount3_pca_tcga_luad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.png
            ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.log
            ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
            ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples00001_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.txt
            ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.log
            ├── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.rds
            └── recount3_tcga_luad_purity01_normtpm_mintpm1_fracsamples02_tissueall_batchtcgagdcplatform_adjtcgagdcplatform.txt
```


## Authors

- Kate Hoff Shutta
- Viola Fanfani
- Panagiotis Mandros
- Enakshi Saha
- Jonas Fischer