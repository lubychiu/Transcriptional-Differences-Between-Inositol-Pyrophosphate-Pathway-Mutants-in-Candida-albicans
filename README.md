# Transcriptional-Differences-Between-Inositol-Pyrophosphate-Pathway-Mutants-in-Candida-albicans
summary...

## Paths
- Raw Fastq files: /home/yc1201/rnaseq_ypd1/raw
- FastQC results: /home/yc1201/rnaseq_ypd1/results/qc
- Reference genome: /home/yc1201/rnaseq_ypd1/reference
- Slurm scripts: /home/yc1201/rnaseq_ypd1/slurm

## Download RNA-Sequencing Results
After downloading RNA-sequencing results from Plasmidsaurus, open zip file.
Upload raw fastq files to the HPC from your local computer. Run the following command from your local computer terminal and not the HPC command line.
```bash
$ gcloud compute scp /Users/lubychiu/Downloads/2GK5PB_fastq/*.fastq.gz m12-controller:/home/yc1201/rnaseq_ypd1/raw
```

## Quality Control (FASTQC)
```bash
# Enter interactive mode on a compute node (from where you are)
$ srun --pty bash

# Make output directory
$ mkdir -p /home/yc1201/rnaseq_ypd1/results/qc

# Load fastqc
$ module load fastqc

# Confirm fastqc is available:
$ fastqc -h

# Run FastQC on all uncompressed fastq files in data/raw/
$ fastqc -o /home/yc1201/rnaseq_ypd1/results/qc/ /home/yc1201/rnaseq_ypd1/raw/*.fastq.gz

$ echo "Quality control complete."
```

### Evaluating FastQC Results
Download the HTML file to visualize FastQC results. Run the following command after exiting from the HPC login node back onto your local computer.
```bash
# Exit the HPC
$ exit

# Download the HTML file
$ gcloud compute scp m12-controller:/home/yc1201/rnaseq_ypd1/results/qc/*.html ~/Downloads/rnaseq_ypd1
```
Results of all 8 samples:
- Phred quality scores: >38 over entire plot
- Adapter content: flat line at 0% across the entire read length
- Per sequence GC content: peaks at ~GC signature of Candida albicans
- Overrepresented sequences: all ~0.1 to 0.2%
- Per base N content: flat at 0% across the entire read length
- Sequences flagged as poor quality: 0

Verdict: can move onto alignment without trimming.

## Alignment
Download the Candida albicans reference genome from NCBI (both FASTA and annotation file). Link: https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_000182965.3/

Use the .fna file that begins with GCF.
Upload the reference genome to the HPC.
```bash
# Upload reference genome
$ gcloud compute scp /Users/lubychiu/Downloads/ncbi_dataset_full/ncbi_dataset/data/GCF_000182965.3/GCF_000182965.3_ASM18296v3_genomic.fna m12-controller:/home/yc1201/rnaseq_ypd1/reference

# Upload annotation file
$ gcloud compute scp /Users/lubychiu/Downloads/ncbi_dataset_full/ncbi_dataset/data/GCF_000182965.3/genomic.gtf m12-controller:/home/yc1201/rnaseq_ypd1/reference
```
### Building the STAR index
To open window to write slurm script:
```bash
$ nano star_index1
```

Once window is open, paste in slurm script.
```bash
#!/bin/bash
#SBATCH --job-name=star_index1
#SBATCH --output=/home/yc1201/rnaseq_ypd1/slurm/z01.%x_%j.out
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=yc1201@georgetown.edu
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --time=02:00:00
#SBATCH --mem=30G

# Establish directory paths
$ BASE_DIR="/home/yc1201/rnaseq_ypd1"
$ REF_DIR="${BASE_DIR}/reference"
$ INDEX_DIR="${REF_DIR}/star_index"

# Create the index directory
$ mkdir -p ${INDEX_DIR}

# Load the STAR module
$ module load star

# Run STAR genomeGenerate
STAR --runMode genomeGenerate \
     --runThreadN 8 \
     --genomeDir ${INDEX_DIR} \
     --genomeFastaFiles ${REF_DIR}/GCF_000182965.3_ASM18296v3_genomic.fna \
     --sjdbGTFfile ${REF_DIR}/genomic.gtf \
     --sjdbOverhang 99 \
     --genomeSAindexNbases 10

echo "STAR Genome Indexing Complete."
```
Click ^x for exit, y to confirm file name, then return. To run the script:
```bash
$ sbatch /home/yc1201/rnaseq_ypd1/slurm/star_index1
```
Check the output file to see if the index was built successfully.
```bash
$ less z01.star_index1_248026.out
```
### STAR Alignment
To open window to write slurm script:
```bash
$ nano star_alignment
```

Once window is open, paste in slurm script. Run slurm array job to align all 8 samples simultaneously.
```bash
#!/bin/bash
#SBATCH --job-name=star_alignment
#SBATCH --output=/home/yc1201/rnaseq_ypd1/slurm/z02.%x_%A_%a.out
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=yc1201@georgetown.edu
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=6
#SBATCH --time=04:00:00
#SBATCH --mem=30G
#SBATCH --array=1-8

# Establish directory paths
$ BASE_DIR="/home/yc1201/rnaseq_ypd1"
$ RAW_DIR="${BASE_DIR}/rnaseq_ypd1/raw"
$ INDEX_DIR="${BASE_DIR}/reference/star_index"
$ OUT_DIR="${BASE_DIR}/results/aligned"

$ mkdir -p ${OUT_DIR}

# 2. Map SLURM Array IDs (1-8) to specific filenames
$ if [ ${SLURM_ARRAY_TASK_ID} -eq 1 ]; then
    SAMPLE="2GK5PB_1_WT1"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 2 ]; then
    SAMPLE="2GK5PB_2_WT2"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 3 ]; then
    SAMPLE="2GK5PB_3_kcs1-1"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 4 ]; then
    SAMPLE="2GK5PB_4_kcs1-2"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 5 ]; then
    SAMPLE="2GK5PB_5_vip1-1"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 6 ]; then
    SAMPLE="2GK5PB_6_vip1-2"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 7 ]; then
    SAMPLE="2GK5PB_7_dbl-1"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 8 ]; then
    SAMPLE="2GK5PB_8_dbl-2"
$ fi

# Load STAR module
$ module load star

# 4. Run STAR alignment for single-end reads
# --readFilesCommand zcat allows STAR to read compressed .gz files directly
$ STAR --runMode alignReads \
     --runThreadN 6 \
     --genomeDir ${INDEX_DIR} \
     --readFilesIn ${RAW_DIR}/${SAMPLE}.fastq.gz \
     --readFilesCommand zcat \
     --outFileNamePrefix ${OUT_DIR}/${SAMPLE}_ \
     --outSAMtype BAM SortedByCoordinate
     --limitBAMsortRAM 1200000000

$ echo "Alignment for ${SAMPLE} complete."
```

Click ^x for exit, y to confirm file name, then return. To run the script:
```bash
$ sbatch /home/yc1201/rnaseq_ypd1/slurm/star_alignment
```

To check status of job, use:
```bash
$ squeue -u yc1201
```

Check one of the output files to see if the job ran successfully.
```bash
$ less z02.star_alignment_248035_1.out
```

## Generating Gene Counts
To open window to write slurm script:
```bash
$ nano featurecounts
```

Once window is open, paste in slurm script.
```bash
#!/bin/bash
#SBATCH --job-name=featurecounts
#SBATCH --output=/home/yc1201/rnaseq_ypd1/slurm/z03.%x_%j.out
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=yc1201@georgetown.edu
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --time=01:00:00
#SBATCH --mem=10G

# Establish directory paths
$ BASE_DIR="/home/yc1201/rnaseq_ypd1"
$ BAM_DIR="${BASE_DIR}/results/aligned"
$ GTF_FILE="${BASE_DIR}/reference/genomic.gtf"
$ OUT_DIR="${BASE_DIR}/results/counts"

$ mkdir -p ${OUT_DIR}

# Load the subread module (which contains featureCounts)
$ module load subread

# Run featureCounts on all 8 BAM files simultaneously
# -T 4: uses 4 processors
# -t exon: counts reads mapping to exon features
# -g gene_id: groups exons by their Gene ID attribute to give you gene-level counts
$ featureCounts -T 4 \
              -a ${GTF_FILE} \
              -t exon \
              -g gene_id \
              -o ${OUT_DIR}/candida_counts_matrix.txt \
              ${BAM_DIR}/*_Aligned.sortedByCoord.out.bam

$ echo "FeatureCounts execution complete."
```

Click ^x for exit, y to confirm file name, then return. To run the script:
```bash
$ sbatch /home/yc1201/rnaseq_ypd1/slurm/featurecounts
```

Download the count matrix to your local computer. Run from local computer terminal.
```bash
$ gcloud compute scp m12-controller:/home/yc1201/rnaseq_ypd1/results/counts/candida_counts_matrix.txt /Users/lubychiu/Downloads/Ca_RNA_Seq/Count_matrix
```

## Differential Expression Analysis
Done in R studio.
```bash
# This installs the manager tool needed for biological packages (only do this once)
$ if (!requireNamespace("BiocManager", quietly = TRUE)) {
   install.packages("BiocManager")
}

# This installs DESeq2 and a graphing tool (this might take a few minutes)
$ BiocManager::install("DESeq2")
$ install.packages("ggplot2")

$ library(DESeq2)
$ library(ggplot2)

# Load the 8-sample count matrix text file
$ counts_raw <- read.table("candida_counts_matrix.txt", header = TRUE, comment.char = "#", check.names = FALSE)

# Extract Geneid (column 1) and your 8 raw count columns (columns 7 through 14)
$ counts_clean <- counts_raw[, c(1, 7:14)]

# Collapse any duplicate gene names by adding their counts together
$ counts_fixed <- aggregate(. ~ Geneid, data = counts_clean, FUN = sum)
$ rownames(counts_fixed) <- counts_fixed$Geneid
$ counts <- counts_fixed[, -1]

# MANUALLY BUILD A PERFECT METADATA TABLE TO BYPASS DISK FILE GLITCHES
# This extracts the exact 8 long BAM names straight from your clean counts matrix columns
$ bam_names <- colnames(counts)

# Define your 4 strains corresponding to samples 1 through 8 in order
strains_vector <- c("WT", "WT", "kcs1", "kcs1", "vip1", "vip1", "dbl", "dbl")

# Create the clean dataframe
$ metadata <- data.frame(
   Sample_ID = bam_names,
   Strain = factor(strains_vector),
   row.names = bam_names
)

# Set 'WT' as your baseline control benchmark
$ metadata$Strain <- relevel(metadata$Strain, ref = "WT")

# Verify the data layout looks spectacular before running the math
$ print("--- CLEAN METADATA VERIFICATION ---")
$ print(metadata)

# Build the DESeq2 data object
$ dds <- DESeqDataSetFromMatrix(countData = counts,
                              colData = metadata,
                              design = ~ Strain)

# Filter out lowly expressed genes (less than 10 total reads)
$ keep <- rowSums(counts(dds)) >= 10
$ dds <- dds[keep, ]

# Run the core differential expression analysis pipeline (Takes ~30 seconds)
$ dds <- DESeq(dds)

# Print the final available strain comparisons to your screen!
$ print("--- AVAILABLE DESEQ2 COMPARISONS ---")
$ resultsNames(dds)


# Extract, sort, and save dbl vs WT results
$ res_dbl <- results(dds, name = "Strain_dbl_vs_WT")
$ res_dbl_sorted <- res_dbl[order(res_dbl$padj), ]
$ write.csv(as.data.frame(res_dbl_sorted), file = "Dbl_vs_WT_Differential_Expression.csv")

# Extract, sort, and save kcs1 vs WT results
$ res_kcs1 <- results(dds, name = "Strain_kcs1_vs_WT")
$ res_kcs1_sorted <- res_kcs1[order(res_kcs1$padj), ]
$ write.csv(as.data.frame(res_kcs1_sorted), file = "Kcs1_vs_WT_Differential_Expression.csv")

# Extract, sort, and save vip1 vs WT results
$ res_vip1 <- results(dds, name = "Strain_vip1_vs_WT")
$ res_vip1_sorted <- res_vip1[order(res_vip1$padj), ]
$ write.csv(as.data.frame(res_vip1_sorted), file = "Vip1_vs_WT_Differential_Expression.csv")

$ print("All 3 results spreadsheets successfully saved to your computer!")
```
