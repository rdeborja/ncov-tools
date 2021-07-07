# ncov-tools

Tools and plots for perfoming quality control on coronavirus sequencing results.

## Installation

Download the package:
```
git clone https://github.com/jts/ncov-tools
cd ncov-tools
```

To use this package, install the dependencies using conda:
```
conda env create -f workflow/envs/environment.yml
```

Alternatively, if install times are very slow using conda, we recommend using
the conda wrapper: [mamba](https://github.com/TheSnakePit/mamba).

Install mamba as follows:
```
conda install -c conda-forge mamba
```

Then create the ncov-tools environment using mamba

```
mamba env create -f workflow/envs/environment.yml
```

Either way, if you used conda directly or mamba, activate the conda package:

```
conda activate ncov-qc
```

## Required Configuration

This package is implemented as a snakemake pipeline, so requires a `config.yaml` file to describe where the input files are. To generate QC plots, a bam file with reads mapped to a reference genome is required. Consensus sequences (FASTA) are needed to generate a phylogenetic tree with associated mutations.

As an example, let's say your data is laid out in the following structure:

```
   run_200430/
     sampleA.sorted.bam
     sampleA.consensus.fasta
     sampleB.sorted.bam
     sampleB.consensus.fasta
   resources/
     artic_reference.fasta
     V3/
        nCoV-2019.bed
```

Then your config.yaml should look like:

```
# path to the top-level directory containing the analysis results
data_root: run_200430

# optionally the plots can have a "run name" prefix. If this is not defined the prefix will be "default"
run_name: my_run

# path to the nCov reference genome
reference_genome: resources/artic_reference.fasta

# the sequencing platform used, can be "oxford-nanopore" or "illumina"
platform: "oxford-nanopore"

# path to the BED file containing the primers, this should follow the format downloaded from
# the ARTIC repository
primer_bed: resources/V3/nCoV-2019.bed
```

The pipeline is designed to work with the results of `ivar` (illumina) or the artic-ncov2019/fieldbioinformatics workflow (oxford nanopore). It will automatically detect the names of the output files (BAMs, consensus fasta, variants) from these workflows using the `platform` value. If you used a different workflow, you can set the following options to help the pipeline find your files:

```
# the naming convention for the bam files
# this can use the variables {data_root} (as above) and {sample}
# As per the example above, this will expand to run_200430/sampleA.sorted.bam for sampleA
bam_pattern: "{data_root}/{sample}.sorted.bam"

# the naming convention for the consensus sequences
consensus_pattern: "{data_root}/{sample}.consensus.fasta"

# the naming convention for the variants file, NF illumina runs typically use
# "{data_root}/{sample}.variants.tsv and oxford nanopore runs use "{data_root}/{sample}.pass.vcf.gz"
variants_pattern: "{data_root}/{sample}.variants.tsv
```

## Metadata (optional)

Some plots and QC statistics can be augmented with metadata like the qPCR Ct values, or the date the sample was collected. To enable this feature, add the path to the metadata to config.yaml:

```
metadata: "/path/to/metadata.tsv"
```

The expected metadata file is a simple TSV with a `sample` field and optional `ct` and `date` fields. Other fields can be provided but will be ignored.

```
sample   ct     date
sampleA  20.8   2020-05-01
sampleB  27.1   2020-06-02
```

When providing the metadata, the value `NA` can be used for missing data.

## Other optional configuration

Additional features can be turned on by adding to the config if desired:

```
#
# if a list of sample IDs for negative controls is provided, a report containing the amount
# of coverage detected in the negative controls can be generated
#
negative_control_samples: [ "NTC-1", "NTC-2" ]

#
# when building a tree of the consensus genomes you can optionally include other sequences
# in the tree by providing a fasta file here
#
tree_include_consensus: some_genomes_from_gisaid.fasta

# list the type of amplicon BED file that will be created from the "primer_bed".  This can include:
# full -- amplicons including primers and overlaps listed in the primer BED file
# no_primers -- amplicons including overlaps but with primers removed
# unique_amplicons -- distinct amplicons regions with primers and overlapping regions removed
bed_type: unique_amplicons

# minimum completeness threshold for inclusion to the SNP tree plot, if no entry
# is provided the default is set to 0.75
completeness_threshold: 0.9

# the set of mutations to automatically flag in the QC reports
# this can be the name of one of the watchlists built into ncov-watch
# or the path to a local VCF file. 
# Built in lists: https://github.com/jts/ncov-watch/tree/master/ncov_watch/watchlists
mutation_set: spike_mutations

# user specifiable output directory 
# defaults to just current working directory but otherwise
# will write output files to the specified directory
output_directory: run1_output
```

## Running

After configuration, you can run the pipeline using Snakemake

```
# Build the sequencing QC plots (coverage, allele frequencies)
snakemake -s workflow/Snakefile all_qc_sequencing

# Build the analysis QC plots (tree with annotated mutations)
snakemake -s workflow/Snakefile all_qc_analysis

# Build the quality report tsv files (in qc_reports directory) 
snakemake -s workflow/Snakefile all_qc_reports
```

There is also an  `all` rule that executes the three rules noted above in one `snakemake` command:
```
# Build all the reports and plots
snakemake -s workflow/Snakefile all
```

You can also build a single PDF summary with the main plots and results. This requires a working 
installation of pdflatex, which is not provided through the environment

```
snakemake -s workflow/Snakefile all_final_report
```


## Output

```
# A plot containing the coverage depth across the SARS-CoV-2 reference genome for each sample in the run
plots/run_name_depth_by_position.pdf

# A plot containing the coverage across all samples, plotted as a heatmap across amplicons
plots/run_name_amplicon_coverage_heatmap.pdf

# A plot with the variation found within each sample, plotted as a tree with associated SNP matrix
plots/run_name_tree_snps.pdf

# A report on per-sample quality metrics and pass/warn/fail criteria
qc_reports/run_name_summary_qc.tsv

# A report on coverage within each negative control
qc_reports/run_name_negative_control_report.tsv

# A report on positions within the genome that are consistently ambiguous across samples (an indicator of possible contamination)
qc_reports/run_name_ambiguous_report.tsv

# A report on samples that have evidence for a mixture of alleles at multiple positions (this code is experimental and still being tested)
qc_reports/run_name_mixture_report.tsv
```

## Variant Annotation
SNVs and Indels are annotated using SNPEff.  The `MN908947.3` SNPEff database
is part of the standard set of genomes.

Currently the database is not available for download and requires building.  To
download the NCBI gene file and build the database, run the following:

```
snakemake -s workflow/Snakefile --cores 1 build_snpeff_db
```

Once the database has been built, the workflow can be run using:

```
snakemake -s workflow/Snakefile --cores 2 all_qc_annotation
```

Variant annotation output can be found in `qc_annotation` and the recurrent
amino acid change heatmap can be found in `plots/<prefix>_aa_mutation_heatmap.pdf`.


## Credit and Acknowledgements

* The tree-with-SNPs plot was inspired by a plot shared by Mads Albertsen.

* The script to convert `variants.tsv` files into `.vcf` files was obtained
  from: `https://github.com/nf-core/viralrecon/blob/dev/bin/ivar_variants_to_vcf.py`

