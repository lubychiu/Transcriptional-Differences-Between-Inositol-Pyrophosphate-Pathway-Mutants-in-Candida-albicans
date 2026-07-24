# Transcriptional-Differences-Between-Inositol-Pyrophosphate-Pathway-Mutants-in-Candida-albicans
This repository contains a bioinformatics pipeline designed to evaluate global transcriptional differences between inositol pyrophosphate pathway mutants in Candida albicans. In this study, raw single-end RNA-seq data collected from eight samples across four genetic backgrounds grown in standard YPD media was processed through an upstream Unix-based shell script pipeline before importing downstream count matrices into R for advanced statistical modeling.The workflow is divided into two major phases executed sequentially:

## Phase 1: Upstream HPC Preprocessing
Executed on a high-performance computing cluster, this phase takes raw multiplexed FASTQ reads down to a clean, structured count matrix:
- Quality Control (FastQC): Evaluates raw paired-end sequence data for adapter contamination and quality decay.
- Note: Skipped trimmimg (Trimmomatic) since data was high quality.
- Reference Genome Indexing & Alignment (STAR): Builds a comprehensive genomic index from the official Candida albicans assembly and aligns the trimmed RNA-seq reads across splice junctions.
- Coordinate Sorting & Quantitation (SAMtools & featureCounts): Converts heavy SAM alignments into binary BAM format, sorts them by genomic coordinates, and quantifies raw integer read counts mapped per gene to generate the primary candida_counts_matrix.txt.

## Phase 2: Downstream R Statistical & Functional Analysis
Executed locally in R, this phase transforms raw counts into clean biological insights:
- Deduplication & Biological Filtering: Imports the 8-sample text matrix, collapses duplicate heterozygous C. albicans A and B locus alleles via summation, and prunes all structural transfer RNAs (tRNA) and ribosomal RNAs (rRNA) prior to statistical modeling to optimize normalization and eliminate multiple-testing penalties.
- Differential Gene Expression Statistics (DESeq2): Fits a negative binomial generalized linear model across the multiplexed data frame, locking the Wild Type (WT) strain as the baseline control reference, and conducts pairwise statistical tests to output sorted, false discovery rate (FDR)-adjusted spreadsheet matrices for the kcs1, vip1, and dbl mutants.
- Advanced Genomic ID Mapping: Cross-references raw systematic IDs against the official Candida Genome Database (CGD) using a robust, multi-strategy approach that scans symbols, systematic tags, and historic synonyms simultaneously to translate alphanumeric strings into human-readable gene nomenclature without causing lossy row collapsing.
- Directional Gene Ontology Enrichment (clusterProfiler & GO.db): Splits significantly altered protein-coding genes by regulatory direction—Up-Regulated (\(\log_2\text{FC} \ge 1.0\)) versus Down-Regulated (\(\log_2\text{FC} \le -1.0\))—and queries the universal GO.db ontology map using a custom hypergeometric enrichment model to generate publication-ready dotplots using true biological pathway names on the chart axes.

For full pipeline, refer to PIPELINE.md.
