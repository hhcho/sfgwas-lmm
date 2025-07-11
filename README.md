# SF-GWAS for Linear Mixed Models

Software for secure and federated genome-wide association studies, as described in:

**Secure and Federated Genome-Wide Association Studies for Biobank-Scale Datasets**\
Hyunghoon Cho, David Froelicher, Jeffrey Chen, Manaswitha Edupalli, Apostolos Pyrgelis, Juan R. Troncoso-Pastoriza, Jean-Pierre Hubaux, Bonnie Berger\
Nature Genetics, 2025

This repository provides the code for linear mixed model (LMM)-based association analysis. For PCA-based GWAS, see [here](https://github.com/hhcho/sfgwas).

## Installation

### Docker

The repository provides both a [Dockerfile](Dockerfile) and a
GitHub Container Registry image ([ghcr.io/hcholab/sfgwas-lmm](ghcr.io/hcholab/sfgwas-lmm)).
If you'd like to re-build the image yourself, please run:

```
docker build -t sfgwas-lmm .
```

See also [Docker demo](#docker-demo) section below on how to run
a series of self-contained tests in Docker.

Alternatively, if you'd like to build the software without Docker,
please follow the instructions below.

### Dependencies

SF-GWAS requires that `go`, `python3`, and `plink2` are available in the exec path in shell. Here are the links for installation:

- [Go](https://go.dev/doc/install) (>=1.18.3)
- Python (>=3.9.2) with [NumPy](https://numpy.org/install/)
- [PLINK2](https://www.cog-genomics.org/plink/2.0/)

### Install SF-GWAS

To install SF-GWAS, clone the repository and try building as follows.
```
git clone https://github.com/hhcho/sfgwas-lmm
cd sfgwas-lmm
cd lmm
go build
```

If `go build` produces an error, run any commands suggested by Go and try again. If the build
finishes without any output, the package has been successfully configured.

## Usage

### Input data

We provide an example synthetic dataset in `example_data/`, generated using the [genotype data simulation routine](https://zzz.bwh.harvard.edu/plink/simulate.shtml) in PLINK1.9
and converted to the PLINK2 PGEN format.
The example data is split between two parties. Each party's local data is stored in
`party1` and `party2` directories. Note that SF-GWAS can be run with more than two parties.

Main input data files include:
- `geno/chr[1-22].[pgen|psam|pvar]`: [PGEN files](https://www.cog-genomics.org/plink/2.0/input#pgen) for each chromosome.
- `pheno.txt`: each line includes the phenotype under study for each sample in the `.psam` file.
- `cov.txt`: each line includes a tab-separated list of covariates for each sample in the `.psam` file.
- `sample_keep.txt`: a list of sample IDs from the `.psam` file to include in the analysis; to be used with the `--keep` flag in PLINK2 (see [here](https://www.cog-genomics.org/plink/2.0/filter#sample) for file specification).

### Data preprocessing

We provide a turn-key script [data_prep.sh](scripts/data_prep.sh) that does all of the required
input data preprocessing:

```sh
./scripts/data_prep.sh [Party ID] path/to/data/party[Party ID]
```

where `[Party ID]` is set to 1 or 2 for the two compute parties.

Note that a third auxiliary party with ID 0 also needs to run together with the main parties to facilitate the protocol.

However, if you wish to preprocess all data manually, the steps to do that are explained below.

#### Converting genotype data to binary block format

For LMM-based workflow, we currently require that the PGEN files be combined and converted to our binary block format as follows:

1. Convert the PGEN files to PLINK1.9 BED format using `plink2 --make-bed` command.
2. Merge the BED files into a single file set using PLINK1.9 with the command `--merge-list`.
3. Run the conversion script we provided to convert to a binary format:

```
python3 scripts/plinkBedToBinary.py combined.bed [Sample count] [SNP count] combined_binary.bin
```

4. Run the block generation script to split the binary file into blocks:

```
python3 scripts/matrix_text2bin_blocks.py [Input directory] [Number of parties] [Number of folds] [Block size] [Output directory]
```

5. Update the config files accordingly to provide the paths to the block-format genotype files.

#### Preparing additional input files

We provide two preprocessing scripts in `scripts/` for producing additional input files needed for SF-GWAS.

1. `createSnpInfoFiles.py` processes the provided `.pvar` files to create a number of files specifying variant information. It can be run as follows:

`python3 createSnpInfoFiles.py PGEN_PREFIX OUTPUT_DIR`

Note that `PGEN_PREFIX` is expected to be a format string including `%d` in place of the chromosome number (e.g., `geno/chr%d` for the example dataset), which the script sequentially replaces with the chromosome numbers 1-22 (note that it's OK to supply only some chromosomes).

This command generates the following three files in `OUTPUT_DIR`:
- `chrom_sizes.txt`: the number of SNPs for each chromosome
- `snp_ids.txt`: a list of variant IDs
- `snp_pos.txt`: a tab-separated, two-column matrix specifying the genomic position of each variant (chromosome number followed by base position)

2. `computeGenoCounts.py` runs PLINK2's [genotype counting routine](https://www.cog-genomics.org/plink/2.0/basic_stats#geno_counts) (`--geno-counts`) on each chromosome to obtain genotype, allele, and missingness counts for each variant. It is run as follows:

`python3 computeGenoCounts.py PGEN_PREFIX SAMPLE_KEEP OUTPUT_DIR`

`PGEN_PREFIX` is the same as before. `SAMPLE_KEEP` points to the `sample_keep.txt` described above as a main input file, including a list of sample IDs to be included in the analysis in a PLINK2-recognized format.

This script generates `all.gcount.transpose.bin` in `OUTPUT_DIR`, which needs to be provided to SF-GWAS. It is a binary file encoding a 6-by-m matrix of precomputed allele, genotype, and missingness counts for all m variants in the dataset.

We provide both variant information files and the genotype counts for the example dataset in `example_data/`.

### Setting the configuration

Example config files are provided in `config/`. There are both global config parameters shared by all parties and party-specific parameters.

### Running the program

Given the computational resource needed for the LMM-based workflow, we recommend running each party on a separate machine (with their IP addresses updated accordingly in the config files). Note that in our experiments, we used a Google Cloud VM with 64 vCPUs and 512GB RAM.

Parallelization in our code requires many file pointers to be open at the same time. We recommend increasing the default limit by running `ulimit -n 12000`.

The LMM workflow is composed of three steps: Level 0, Level 1, and association tests. Levels 0 and 1 implement the two rounds of stacked ridge regression described in our manuscript (following [REGENIE](https://rgcgithub.github.io/regenie/)).

The commands to run each of the three steps are:
```
PID=[Party ID] go test -run TestLevel0 -timeout 96h
PID=[Party ID] go test -run TestLevel1 -timeout 96h
PID=[Party ID] go test -run TestAssoc -timeout 96h
```

#### Docker demo

Rather than running those commands manually, the provided Docker image
executes a [demo.sh](scripts/demo.sh) script that automatically runs
all 3 parties locally inside the container, for each of the test steps:

```
docker run --rm -it --pull always ghcr.io/hcholab/sfgwas-lmm
```

### Output

Once the workflow finishes, it generates `assoc.txt` in the output directory specified in the configuration. This file includes the LMM-based association test chi-square statistics. Details are provided in our manuscript.

## Contact for questions

Hoon Cho, hhcho@broadinstitute.org
