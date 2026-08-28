# POOCH

*** COMING SOON!!! ***

Parent-Of-Origin Classification of Haplotypes.

POOCH is a Snakemake-based pipeline for assigning chromosome labels and determining the parent of origin (maternal or paternal) of near-complete genome assemblies.

```bash
 / \__          ┌─────────────────────────────────────┐
(    @\___      │               POOCH                 │
 /         O    │ Parent Of Origin Classification     │
/   (_____/     │          Of Haplotypes              │
/_____/   U     └─────────────────────────────────────┘

```

## What It Does

- Aligns ONT or HiFi reads to a personalized diploid genome
- Calls methylation from primary alignments
- Lifts methylation back to reference coordinates (CHM13)
- Builds chain-based chromosome assignments
- Predicts parent-of-origin labels per chromosome

## Pipeline At A Glance
```bash
job                                   count
----------------------------------  -------
all                                       1
step00_methylation_tag_check              1
step01_generate_personalized_genome		  1 # skip if diploid assembly is provided
step02_align                              1 # skip if BAM is provided
step03_make_chain                         1 # skip if chain files are provided
step04_call_methylation                   1 # skip if methylation bed files are provided
step05_liftover_hap1                      1
step05_liftover_hap2                      1
step06_chromAssign                        1
step07_check_XX_XY                        1
step08_parent_of_origin_prediction       23
step09_cleaning_output                    1
total                                    33
```

The most time-consuming step is `step02_align`, and its runtime depends on the sequencing coverage of the ONT or HiFi data. The most computationally intensive step, requiring the highest memory usage, is `step03_make_chain`.


## Requirements

POOCH expects the following tools available in your environment (or via your cluster module system):

- `snakemake`
- `samtools`
- `meryl`
- `winnowmap` (if using `--aligner Winnowmap2`)
- `minimap2` (if using `--aligner Minimap2`)
- `crossmap`

If your environment supports modules, the scripts will try to `module load` several tools automatically.

## Input Modes

Choose exactly one genome mode:

1. Personalized genome mode
- `--personalized_genome` plus hap names `--hap1` and `--hap2`

2. Split FASTA mode
- `--hap1_fa` and `--hap2_fa`

At least one data input is required:

- `--fastq` (alignment + downstream steps)
- `--bam` (skip alignment)
- `--metbed` (skip alignment and methylation calling)

## Quick Start

### 1) Personalized genome (diploid) + FASTQ

```bash
./pooch \
	--reference ref.fa \
	--personalized_genome personalized.fa \
	--hap1 haplotype1 \
	--hap2 haplotype2 \
	--fastq reads_1.fastq.gz,reads_2.fastq.gz \
	--platform ONT \
	--aligner Winnowmap2 \
	--threads 20 \
	--output_prefix SAMPLE_A \
	--outdir /path/to/output
```

### 2) Split FASTA (haploids) + BAM

```bash
./pooch \
	--reference ref.fa \
	--hap1_fa sample.hap1.fa \
	--hap2_fa sample.hap2.fa \
	--bam sample.pri.bam \
	--platform ONT \
	--output_prefix SAMPLE_A \
	--outdir /path/to/output
```

### 3) Dry run before execution

```bash
./pooch \
	--reference ref.fa \
	--personalized_genome personalized.fa \
	--hap1 haplotype1 \
	--hap2 haplotype2 \
	--fastq reads.fastq.gz \
	--platform ONT \
	--output_prefix SAMPLE_A \
	--outdir /path/to/output \
	--dry-run
```

### 4) If you need touch files

```bash
./pooch \
	--reference ref.fa \
	--personalized_genome personalized.fa \
	--hap1 haplotype1 \
	--hap2 haplotype2 \
	--fastq reads.fastq.gz \
	--platform ONT \
	--output_prefix SAMPLE_A \
	--outdir /path/to/output \
	--snakeopts "--touch"
```

## Core Arguments

- `--reference`: reference genome FASTA
- `--outdir`: output directory
- `--output_prefix`: sample/output prefix
- `--platform`: platform used for methylation calling (`ONT` or `HiFi`)
- `--aligner`: `Winnowmap2` or `Minimap2` (default: `Winnowmap2`)
- `--model_dir`: model directory (default points to project model set)
- `--threads`: total cores for Snakemake

Run `./pooch --help` for full argument documentation.

## Primary Outputs

The workflow targets:

- `OUTDIR/chain/SAMPLE.chromAssign.all.txt`
- `OUTDIR/prediction/SAMPLE.predict.out`

Example rows from `SAMPLE.predict.out`:

| Sample | LeftOpHapLabel | RightOpHapLabel | LeftHapPredictedOrigin | RightHapPredictedOrigin | PredictProb_Maternal-Paternal | PredictProb_Paternal-Maternal | PredictionCorrect | Chrom |
| --- | --- | --- | --- | --- | ---: | ---: | --- | --- |
| GM03417_ONT_Winnowmap2 | Hap1 | Hap2 | Paternal | Maternal | 0.00103682 | 0.998963 | Unknown | chr1 |
| GM03417_ONT_Winnowmap2 | Hap1 | Hap2 | Maternal | Paternal | 0.999887 | 0.000113318 | Unknown | chr2 |
| GM03417_ONT_Winnowmap2 | Hap1 | Hap2 | Maternal | Paternal | 0.658104 | 0.341896 | Unknown | chr3 |
| GM03417_ONT_Winnowmap2 | Hap1 | Hap2 | Paternal | Maternal | 0.00135033 | 0.99865 | Unknown | chr4 |

The probability is calculated for the maternal–paternal comparison, where LeftOpHapLabel represents the left side of the comparison. Thus, the comparison is interpreted as LeftOpHapLabel–RightOpHapLabel. Accordingly, the PredictProb_Maternal-Paternal column gives the probability that LeftOpHapLabel is maternal (and RightOpHapLabel is paternal), while the PredictProb_Paternal-Maternal column gives the probability that LeftOpHapLabel is paternal (and RightOpHapLabel is maternal). For example, for chromosome 1, Hap1 is predicted to be paternal and Hap2 is predicted to be maternal.


Example rows from `SAMPLE.chroms.txt`:

| Chrom | Contig | Strand | AlignBlock |
| --- | --- | --- | ---: |
| chr1 | haplotype1-0000015 | + | 79653 |
| chr1 | haplotype1-0000022 | + | 729435 |
| chr1 | haplotype1-0000031 | + | 228362455 |
| chr1 | haplotype2-0000078 | + | 107998824 |
| chr1 | haplotype2-0000079 | + | 122440753 |

Three contigs from haplotype 1 and two contigs from haplotype 2 are assigned to chromosome 1. Haplotype 1 is predicted to be paternal, and haplotype 2 is predicted to be maternal. Therefore, the first three contigs correspond to the paternal copy of chromosome 1, and the last two contigs correspond to the maternal copy of chromosome 1.

## Running Individual Steps

Use `--step` to pass a specific Snakemake target/rule argument.

Example:

```bash
./pooch ... --step step04_call_methylation
```

## License

See `LICENSE.md`.
