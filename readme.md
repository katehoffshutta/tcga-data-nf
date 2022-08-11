# tcga-data-nf

![](https://github.com/QuackenbushLab/tcga-data-nf/workflows/build/badge.svg)

Workflow to download and prepare TCGA data
This replaces the recount3-access-nf from Kate.

The workflow divides the process of downloading the data in two steps:
1. Downloading the raw data from GDC and saving the rds/tables needed later
2. Preparing the data. This step includes filtering the data, normalizing it... 

The idea is that data should be downloaded once, and then prepared for the task at hand.

## Configuration

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

### Prepare

For now, only gene expression data needs to be prepared (harmonized with gtex and normalised).

Specify in the configuration file which `recount.metadata_prepare = table.csv` contains the info about 
the gtex-tcga data. One example is the one in the conf folder.

## Running the workflow

### Temporarily (until github actions are working)

1. Clone the nextflow repo 
   ```bash
   git clone git@github.com:QuackenbushLab/tcga-data-nf.git```
2. Build docker locally 
    ```bash
    docker build . -f containers/Dockerfile --build-arg CONDA_FILE=containers/env.base.python.yml --no-cache -t tcga-data-nf:latest```
3. In alternative, check the R packages needed and install them manually.
4. **TO DOWNLOAD**: update the `download_metadata.config` file or pass a new one with the correct parameters (keep track
   of all config files you use)
5. **To prepare**: use a correct `expression_prepare_table.csv`
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

## Results

- `results/analysis.txt`: file with the analysis results.
- `results/tuning.txt`: file with the parameter tuning.
- ...

## Authors

- Kate Hoff Shutta
- Viola Fanfani
- Panagiotis Mandros
- Enakshi Saha
