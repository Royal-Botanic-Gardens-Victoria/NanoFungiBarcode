# NanoFungiBarcode

## Software requirements
- [Dorado](https://github.com/nanoporetech/dorado) basecaller
  - see [installation](https://github.com/nanoporetech/dorado#Installation)
- [snake_make_ont_consensus_build](https://github.com/ritatam/snakemake_ont_consensus_build)
  - will require the [conda](https://docs.conda.io/en/latest/) package manager
- [chopper](https://github.com/wdecoster/chopper) quality filtering
  - can be installed via conda
    - ```conda create -c bioconda -n chopper chopper```
    - ```conda activate chopper```

## General procedure
1. Convert raw signal from nanopore device into a sequence of nucleotides. 
  This step is called basecalling and is very computationally demanding.  
2. If multiple samples have been sequenced together at the same time (multiplexing),
  the data will need to be demultiplexed to separate sequences from each sample by identifying a barcode adapter.
3. Low quality sequences should be removed before building consensus sequences. 
  We can select only the sequences that meet a minimum mean quality score threshold (in this case Q15 on the [Phred scale](https://en.wikipedia.org/wiki/Phred_quality_score)).
4. Recover consensus sequences for each sample.

## 0 - Prerequisites

- Raw Oxford Nanopore data (in `pod5` file format)
  - if your data is in `fast5` format, convert to pod5 here: https://pod5.nanoporetech.com/
  - Record the directory of your raw data, I will refer to this directory as `"raw-data/pod5-dir"`.
- The barcoding/adapter kit used
  - In this case we are using the `SQK-RBK114-24` (Rapid Barcoding Kit 24 V14).
  - Run `dorado demux --help` to see the list of kits for the `--kit-name` option.
- An output directory to store the results. I will refer to this directory as `"output-dir"`.
  ```bash
  mkdir "output-dir"
  ```

## 1 - Basecalling

Perform the basecalling on your raw `pod5` files in the `raw-data/pod5-dir` directory.
```bash
dorado basecaller \
    --no-trim \
    --recursive \
    --device cpu \
    sup "raw-data/pod5-dir" \
    > output-dir/basecalled_reads.bam
```

The `--no-trim` option tells dorado to not perform any adapter trimming (which will be done in the demultiplexing step).

The `--recursive` option tells dorado to look through all the subfolders of the `raw-data/pod5-dir` for `pod5` files.

The `--device cpu` option tells dorado to use the CPU device to perform the basecalling.
If you have a CUDA enabled GPU or Apple M1/M2 installed run `dorado basecaller --help` for other options.
CPU basecalling will be much slower than GPU basecalling.

The `sup` option tells dorado to use the 'super accurate' basecalling model which will produce the 
highest accuracy sequences at high computational cost.
See which models are available [here](https://github.com/nanoporetech/dorado/?tab=readme-ov-file#dna-models).

## 2 - Demultiplexing

Make a directory to store the demultiplexed sequences.
```bash
mkdir "ouput-dir/demux"
```
Ensure the barcoding kit name (`--kit-name`) is the same as the one used in the library preparation or else most sequences will be unclassified.
```bash
dorado demux \
  --kit-name SQK-RBK114-24 \
  --output-dir output-dir/demux \
  --emit-fastq \
  output-dir/basecalled_reads.bam
```

The `--emit-fastq` option tells dorado to output `.fastq` formatted files.
It will output `.bam` files by default (without this option).

At the end of this step there should be one file per sample in the `output-dir/demux` directory.
Sequences where the barcode could not be classified will be placed in `output-dir/demux/unclassified.fastq`.

## 3 - Quality filtering

This command iterates through the `.fastq` file for each sample and 
removes sequences that have a mean quality score lower than Q15.
```bash
for filename in output-dir/demux/*.fastq; do \
  cat ${filename} | chopper --quality 15 \
    > output-dir/filt-Q15/$(basename $filename);\
done
```

## 4 - Building a polished consensus sequence

`snake_make_ont_consensus_build` is a snakemake pipeline that can do this.
Details on how this pipeline works can be found here: https://github.com/ritatam/nanopore-biosecurity-workshop-2022

Follow the steps on how to run the pipeline here: https://github.com/ritatam/snakemake_ont_consensus_build .
They have been repeated here for completeness:

1. Clone the repository
```bash
 git clone https://github.com/ritatam/snakemake_ont_consensus_build.git
 cd snakemake_ont_consensus_build
```
2. Configure paths in the `config.yaml` file. These paths are relative to the `config.yaml` file.
```yaml
input_reads_dir: "../output-dir/filt-Q15"
output_dir: "../output-dir/ont_consensus"
```
3. Install snakemake into a conda environment
```bash
conda install snakemake
```
4. Install dependencies into a conda environment
```bash
snakemake --use-conda --conda-create-envs-only -c1
```
5. Do a dry-run of the pipeline:
```bash
snakemake --use-conda -np
```
6. Run the pipeline:
```
snakemake --use-conda -c8 
```