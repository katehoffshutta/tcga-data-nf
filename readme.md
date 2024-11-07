# tcga-data-nf
## End-to-end reproducible processing of cancer regulatory networks from TCGA data

![](https://github.com/QuackenbushLab/tcga-data-nf/workflows/build/badge.svg)

version = '0.0.13'

Workflow to download and prepare TCGA data.

The workflow divides the process of downloading the data in two steps:
1. Downloading the raw data from GDC and saving the rds/tables needed later
2. Preparing the data. This step includes filtering the data, normalizing it... 
3. Analysis of gene regulatory networks

The idea is that data should be downloaded once, and then prepared for the task at hand.

:warning: For the moment the NetworkDataCompanion package needs to be installed locally. 

:warning: GitHub is still not allowing the QuackenbushLab to run actions. The docker image is built and pushed 
manually.

## Getting started 

1. First you'll need to [install](https://www.nextflow.io/docs/latest/install.html) nextflow on your machine. Follow the
   `hello world` example to check if Nextflow is up and running.
2. Pull the workflow `nextflow pull QuackenbushLab/tcga-data-nf`
3. Install and pull the docker/singularity container or conda to run the whole pipeline
  3a. **DOCKER**: 
    - [Install](https://docs.docker.com/engine/install/) docker on your machine
    - Pull the container for this workflow `docker pull violafanfani/tcga-data-nf`
    - use the `-profile docker` option when running the workflow
  3b. **CONDA**:
    - Install [conda](https://docs.anaconda.com/miniconda/).
    - Use the `-profile conda` option when running the workflow
4. Run some test workflows
  4a. test the download: `nextflow run QuackenbushLab/tcga-data-nf -profile <docker/conda>, testDownload `
  4a. test the prepare: `nextflow run QuackenbushLab/tcga-data-nf -profile <docker/conda>, testPrepare`
  4a. test the analyze: `nextflow run QuackenbushLab/tcga-data-nf -profile <docker/conda>, testAnalyze`
  4a. test the full workflow: `nextflow run QuackenbushLab/tcga-data-nf -profile <docker/conda>, test `

If you can run all these steps, you can procede defining your own configuration files and run your own analysis. 

:warning: The full workflow can be slow (>45minutes).  


## Running the workflow

### Docker

The docker container is hosted on docker.io. 

```docker pull violafanfani/tcga-data-nf:0.0.12```

More details on the container can be found in the [docs](docs.md#Docker)

### Conda

Alternatively, one can run the workflow with conda environments. 
In order to create and use conda one can pass it as a profile `-profile conda`
as:

```nextflow run QuackebushLab/tcga-data-nf -profile conda,test ...```

For the moment we are using one single environment to be used with all the r
scripts. This allows the pipeline to generate the environment only once (which can 
be time consuming) and then to reuse it.

The three environment are inside the `containers/conda_envs` folder: 
1. merge_tables
2. r_all
3. analysis

However, to improve portability, we use the process selectors labels to specify the different environments, allowing the user to specify their own environments too. 

More details in the [docs](docs.md#conda).

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

Download testing can be done by specifying the `-profile testDownload`
parameter. 

The data to be downloaded is specified in the `testdata/download_json.json` file. 

All data will be saved in the `results/test_download` folder.

Check the [download](#Results:-Download) results section

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

Detailed output folder structure can be found at the [docs](docs.md##result-folders)


## Authors


- Viola Fanfani
- Kate Hoff Shutta
- Panagiotis Mandros
- Jonas Fischer
- Enakshi Saha