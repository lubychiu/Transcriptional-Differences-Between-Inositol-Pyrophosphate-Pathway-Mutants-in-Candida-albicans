# Transcriptional Differences Between Inositol Pyrophosphate Pathway Mutants in Candida albicans
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
$ RAW_DIR="${BASE_DIR}/raw"
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

## Visualizations
Done in RStudio.
### Heatmap
Top 50 genes with largest fold changes relative to WT.
```bash
# This installs the manager tool needed for biological packages (only do this once)
$ if (!requireNamespace("BiocManager", quietly = TRUE)) {
  install.packages("BiocManager")
}
$ library(DESeq2)
$ library(EnhancedVolcano)
$ library(pheatmap)
$ library(RColorBrewer)
$ library(ggplot2)

# PARSE DATABASE COORDINATES USING RAW CHROMOSOMAL LOOKUP TAB
$ print("Loading CGD database file and mapping numeric coordinates...")
$ cgd_raw <- read.table("cgd_features.tab", header = FALSE, sep = "\t", quote = "", comment.char = "!")

$ mapping_table <- data.frame(
  systematic_id = trimws(as.character(cgd_raw[, 1])),
  gene_symbol   = trimws(as.character(cgd_raw[, 2])),
  stringsAsFactors = FALSE
)
$ mapping_table$gene_symbol[mapping_table$gene_symbol == "" | mapping_table$gene_symbol == mapping_table$systematic_id] <- NA
$ mapping_table$numeric_key <- as.numeric(gsub("[^0-9]", "", mapping_table$systematic_id))
$ mapping_table <- mapping_table[!is.na(mapping_table$numeric_key), ]

# REFINED INDIVIDUAL VOLCANO PLOTS (BIGGER AXES + CLEANER TEXT)
$ print("Generating scaled Volcano Plots...")
$ volcano_targets <- list("Dbl_vs_WT" = res_dbl, "Kcs1_vs_WT" = res_kcs1, "Vip1_vs_WT" = res_vip1)

$ for (comp_name in names(volcano_targets)) {
   res_df <- as.data.frame(volcano_targets[[comp_name]])
   res_df$raw_id <- rownames(res_df)
   res_df$numeric_key <- as.numeric(gsub("[^0-9]", "", res_df$raw_id))
   res_df <- merge(res_df, mapping_table[, c("numeric_key", "gene_symbol")], by = "numeric_key", all.x = TRUE)
  
  # Format names cleanly: shorten non-characterized locus codes
$  short_code <- gsub("CA$", "", gsub("^CAALFM_", "", res_df$raw_id))
$  base_label <- ifelse(!is.na(res_df$gene_symbol), res_df$gene_symbol, short_code)
$  res_df$final_label <- ifelse(!is.na(res_df$padj) & res_df$padj < 0.05, base_label, "")
  
  # Export to folder
$   png(filename = paste0("Volcano_Plot_", comp_name, ".png"), width = 1200, height = 1000, res = 150)
  
$   p_vol <- EnhancedVolcano(res_df,
                           lab = res_df$final_label,
                           x = 'log2FoldChange', y = 'padj',
                           pCutoff = 0.05, FCcutoff = 1.0,
                           pointSize = 1.2, labSize = 3.0, # Shrunk point and label scales to stop overlaps
                           col = c('grey50', 'forestgreen', 'royalblue3', 'firebrick2'),
                           title = paste("Comparison:", comp_name),
                           subtitle = "Labelled keys reflect adjusted p-value < 0.05 only",
                           legendPosition = 'bottom',
                           drawConnectors = TRUE, widthConnectors = 0.3,
                           max.overlaps = 80)              # Loosened overlapping parameters considerably
  
  # INJECT RE-FORMATTED ENHANCED AXIS SIZE MODIFICATIONS HERE
$   p_vol <- p_vol + theme(
    axis.title.x = element_text(size = 16, face = "bold"),
    axis.title.y = element_text(size = 16, face = "bold"),
    axis.text.x = element_text(size = 14, color = "black"),
    axis.text.y = element_text(size = 14, color = "black"),
    title = element_text(size = 16, face = "bold")
  )
  
  $ print(p_vol)
  $ dev.off()
}

# CONSTRUCT A UNIFIED LOG2 FOLD-CHANGE HEATMAP (TOP 50 GENES)
print("Compiling a single comparative Log2 Fold Change Heatmap...")

# Create a clean master dataframe combining all log2FoldChanges across the 3 tests
$ fc_matrix_df <- data.frame(
   Dbl_Log2FC  = res_dbl$log2FoldChange,
   Kcs1_Log2FC = res_kcs1$log2FoldChange,
   Vip1_Log2FC = res_vip1$log2FoldChange,
   Dbl_padj    = res_dbl$padj,
   Kcs1_padj   = res_kcs1$padj,
   Vip1_padj   = res_vip1$padj,
   raw_id      = rownames(res_dbl)
)

# Extract matching text titles using numeric coordinates
$ fc_matrix_df$numeric_key <- as.numeric(gsub("[^0-9]", "", fc_matrix_df$raw_id))
$ fc_master <- merge(fc_matrix_df, mapping_table[, c("numeric_key", "gene_symbol")], by = "numeric_key", all.x = TRUE)

# Clean up locus identifiers
$ row_shortened <- gsub("CA$", "", gsub("^CAALFM_", "", fc_master$raw_id))
$ fc_master$display_name <- ifelse(!is.na(fc_master$gene_symbol), fc_master$gene_symbol, row_shortened)

# TARGET THE TOP 50 GENES: Find the 50 genes with the strongest statistical change 
# across ANY of the three mutant strains (sorting by lowest overall p-value)
$ fc_master$min_padj <- pmin(fc_master$Dbl_padj, fc_master$Kcs1_padj, fc_master$Vip1_padj, na.rm = TRUE)
$ fc_master_sorted <- fc_master[!is.na(fc_master$min_padj), ]
$ top50_genes <- head(fc_master_sorted[order(fc_master_sorted$min_padj), ], 50)

# Build the plotting grid matrix using just the fold changes
$ heatmap_matrix <- as.matrix(top50_genes[, c("Dbl_Log2FC", "Kcs1_Log2FC", "Vip1_Log2FC")])
$ rownames(heatmap_matrix) <- top50_genes$display_name
$ colnames(heatmap_matrix) <- c("Dbl Mutant vs WT", "Kcs1 Mutant vs WT", "Vip1 Mutant vs WT")

# Formulate a symmetric color palette centered strictly on zero (Blue=Down, Red=Up)
$ max_val <- max(abs(heatmap_matrix), na.rm = TRUE)
$ color_breaks <- seq(-max_val, max_val, length.out = 101)

# Export the master Heatmap file
$ png(filename = "Unified_Mutants_vs_WT_Log2FC_Heatmap.png", width = 1100, height = 1200, res = 150)

$ pheatmap(heatmap_matrix, 
         scale = "none",              # No row scaling needed since data is already in log2FC units
         cluster_cols = TRUE,         # Clusters mutants together based on behavior similarities
         cluster_rows = TRUE,         # Hierarchical grouping of gene patterns
         color = colorRampPalette(c("royalblue3", "white", "firebrick2"))(100), # White means zero change
         breaks = color_breaks,       # Align color changes cleanly around 0
         fontsize_row = 9,            # Clear label sizes
         fontsize_col = 11,           
         border_color = "grey95",
         main = "Comparative Fold Changes of Top 50 Genes Relative to WT")

$ dev.off()
$ print("SUCCESS: Consolidated multi-mutant heatmap and expanded volcano charts exported successfully!")
```

### Refining Visualizations
Excluded non-coding RNAs and assigns real gene names.
```bash
library(DESeq2)
library(EnhancedVolcano)
library(pheatmap)
library(RColorBrewer)
library(ggplot2)

# ============================================================
# 1. LOAD CGD DATABASE & IDENTIFY tRNA / rRNA FEATURE TYPES
# ============================================================
print("Parsing Candida Genome Database feature entries...")
cgd_raw <- read.table("cgd_features.tab", header = FALSE, sep = "\t",
                      quote = "", comment.char = "!")

# Map columns out cleanly based on CGD structural indexing
# Column 1: Systematic ID, Column 2: Gene Name, Column 4: Feature Type (e.g. tRNA, rRNA, ORF)
mapping_table <- data.frame(
  systematic_id = trimws(as.character(cgd_raw[, 1])),
  gene_symbol   = trimws(as.character(cgd_raw[, 2])),
  feature_type  = trimws(as.character(cgd_raw[, 4])),
  stringsAsFactors = FALSE
)

# Convert empty symbol fields back to clean missing NA formats
mapping_table$gene_symbol[mapping_table$gene_symbol == "" |
                            mapping_table$gene_symbol == mapping_table$systematic_id] <- NA

# Normalize matching strings by dropping hyphens and underscores
normalize_id <- function(x) {
  toupper(gsub("[_\\-]", "", trimws(x)))
}
mapping_table$id_norm <- normalize_id(mapping_table$systematic_id)

# Deduplicate the mapping table to preserve precise 1-to-1 lookups
mapping_table <- mapping_table[order(is.na(mapping_table$gene_symbol)), ]
mapping_table <- mapping_table[!duplicated(mapping_table$id_norm), ]


# ============================================================
# 2. LOAD DATA MATRIX & PURGE NON-CODING HIT TYPES upfront
# ============================================================
print("Reading local data matrix...")
counts_raw <- read.table("candida_counts_matrix.txt", header = TRUE, comment.char = "#", check.names = FALSE)

# Extract Geneid (column 1) and your 8 raw count columns (columns 7 through 14)
counts_clean <- counts_raw[, c(1, 7:14)]

# Collapse duplicate gene names if present in the alignment output
counts_fixed <- aggregate(. ~ Geneid, data = counts_clean, FUN = sum)

# --- ADVANCED BIOLOGICAL FILTER ---
# Normalize matrix keys to match CGD database entries
stripped_keys <- gsub("^CAALFM_", "", counts_fixed$Geneid)
normalized_matrix_keys <- normalize_id(stripped_keys)

# Map feature types directly using vector matches
matched_features <- mapping_table$feature_type[match(normalized_matrix_keys, mapping_table$id_norm)]

# Explicitly identify rows flagged as tRNA or rRNA inside either the key text OR the CGD metadata feature type
is_trna_or_rrna <- grepl("tRNA|rRNA", counts_fixed$Geneid, ignore.case = TRUE) | 
  grepl("tRNA|rRNA", matched_features, ignore.case = TRUE)

print(paste("Total transcripts before biological non-coding filtering:", nrow(counts_fixed)))
counts_fixed <- counts_fixed[!is_trna_or_rrna, ]
print(paste("Total transcripts remaining after absolute tRNA/rRNA exclusion:", nrow(counts_fixed)))
# ----------------------------------

rownames(counts_fixed) <- counts_fixed$Geneid
counts <- counts_fixed[, -1]


# ============================================================
# 3. MANUALLY SET UP TARGET METADATA STRUCTURE
# ============================================================
bam_names <- colnames(counts)
strains_vector <- c("WT", "WT", "kcs1", "kcs1", "vip1", "vip1", "dbl", "dbl")

metadata <- data.frame(
  Sample_ID = bam_names,
  Strain = factor(strains_vector),
  row.names = bam_names
)
metadata$Strain <- relevel(metadata$Strain, ref = "WT")

print("--- RECONFIGURED METADATA DESIGN PROJECTIONS ---")
print(metadata)


# ============================================================
# 4. EXECUTE UPSTREAM DESeq2 STATISTICAL PIPELINE
# ============================================================
print("Building the DESeq2 data object...")
dds <- DESeqDataSetFromMatrix(countData = counts, colData = metadata, design = ~ Strain)

# Drop any globally dead passenger genes with lower than 10 global reads across all libraries
keep <- rowSums(counts(dds)) >= 10
dds <- dds[keep, ]

# Process differential expression parameters
dds <- DESeq(dds)

print("--- COMPLETED DESEQ2 CONTRAST COEFFICIENTS ---")
print(resultsNames(dds))

# Save individual calculation files straight out to CSV tables
res_dbl  <- results(dds, name = "Strain_dbl_vs_WT")
res_kcs1 <- results(dds, name = "Strain_kcs1_vs_WT")
res_vip1 <- results(dds, name = "Strain_vip1_vs_WT")

write.csv(as.data.frame(res_dbl[order(res_dbl$padj), ]), file = "Dbl_vs_WT_Differential_Expression.csv")
write.csv(as.data.frame(res_kcs1[order(res_kcs1$padj), ]), file = "Kcs1_vs_WT_Differential_Expression.csv")
write.csv(as.data.frame(res_vip1[order(res_vip1$padj), ]), file = "Vip1_vs_WT_Differential_Expression.csv")


# ============================================================
# 5. GENERATE CLEAN DISPLAY NAME MAPS FOR THE PLOTS
# ============================================================
add_gene_names <- function(df, id_col = "raw_id") {
  stripped <- gsub("^CAALFM_", "", df[[id_col]])
  df$id_norm <- normalize_id(stripped)
  df$gene_symbol <- mapping_table$gene_symbol[match(df$id_norm, mapping_table$id_norm)]
  short_code <- gsub("CA$", "", stripped)
  df$display_name <- ifelse(!is.na(df$gene_symbol), df$gene_symbol, short_code)
  df
}


# ============================================================
# 6. EXPORT OPTIMIZED SCALED VOLCANO PLOTS
# ============================================================
print("Generating updated Volcano Plots...")
volcano_targets <- list("Dbl_vs_WT" = res_dbl, "Kcs1_vs_WT" = res_kcs1, "Vip1_vs_WT" = res_vip1)

for (comp_name in names(volcano_targets)) {
  res_df <- as.data.frame(volcano_targets[[comp_name]])
  res_df$raw_id <- rownames(res_df)
  res_df <- add_gene_names(res_df, "raw_id")
  
  # Map labels to NA explicitly so EnhancedVolcano doesn't draw blank overlaps
  res_df$final_label <- ifelse(!is.na(res_df$padj) & res_df$padj < 0.05 & abs(res_df$log2FoldChange) >= 1.0, 
                               res_df$display_name, NA)
  
  png(filename = paste0("Volcano_Plot_", comp_name, ".png"), width = 1200, height = 1000, res = 150)
  
  p_vol <- EnhancedVolcano(res_df,
                           lab = res_df$final_label,
                           x = 'log2FoldChange', y = 'padj',
                           pCutoff = 0.05, FCcutoff = 1.0,
                           pointSize = 1.2, labSize = 3.0,
                           col = c('grey50', 'forestgreen', 'royalblue3', 'firebrick2'),
                           title = paste("Comparison:", comp_name),
                           subtitle = "Labelled keys reflect padj < 0.05 and |Log2FC| >= 1.0 (Non-coding Excluded)",
                           legendPosition = 'bottom',
                           drawConnectors = TRUE, widthConnectors = 0.3,
                           max.overlaps = 80)
  
  p_vol <- p_vol + theme(
    axis.title.x = element_text(size = 16, face = "bold"),
    axis.title.y = element_text(size = 16, face = "bold"),
    axis.text.x = element_text(size = 14, color = "black"),
    axis.text.y = element_text(size = 14, color = "black"),
    title = element_text(size = 16, face = "bold")
  )
  
  print(p_vol)
  dev.off()
}


# ============================================================
# 7. UNIFIED VARIANCE-SORTED LOG2FC HEATMAP
# ============================================================
print("Compiling a single comparative Log2 Fold Change Heatmap...")

fc_matrix_df <- data.frame(
  raw_id      = rownames(res_dbl),
  Dbl_Log2FC  = res_dbl$log2FoldChange,
  Kcs1_Log2FC = res_kcs1$log2FoldChange,
  Vip1_Log2FC = res_vip1$log2FoldChange
)

fc_master <- add_gene_names(fc_matrix_df, "raw_id")

# Drop any lines carrying empty missing coordinates
fc_master_clean <- fc_master[!is.na(fc_master$Dbl_Log2FC) & 
                               !is.na(fc_master$Kcs1_Log2FC) & 
                               !is.na(fc_master$Vip1_Log2FC), ]

# Compute biological variance across conditions to reveal true expression shifts
gene_variances <- apply(fc_master_clean[, c("Dbl_Log2FC", "Kcs1_Log2FC", "Vip1_Log2FC")], 1, var)
fc_master_clean$variance <- gene_variances

top50_genes <- head(fc_master_clean[order(-fc_master_clean$variance), ], 50)

heatmap_matrix <- as.matrix(top50_genes[, c("Dbl_Log2FC", "Kcs1_Log2FC", "Vip1_Log2FC")])
rownames(heatmap_matrix) <- top50_genes$display_name
colnames(heatmap_matrix) <- c("Dbl Mutant vs WT", "Kcs1 Mutant vs WT", "Vip1 Mutant vs WT")

# Balance colors precisely around zero 
max_val <- max(abs(heatmap_matrix), na.rm = TRUE)
color_breaks <- seq(-max_val, max_val, length.out = 101)

png(filename = "Unified_Mutants_vs_WT_Log2FC_Heatmap.png", width = 1100, height = 1200, res = 150)

pheatmap(heatmap_matrix,
         scale = "none",
         cluster_cols = TRUE,
         cluster_rows = TRUE,
         color = colorRampPalette(c("royalblue3", "white", "firebrick2"))(100),
         breaks = color_breaks,
         fontsize_row = 9,
         fontsize_col = 11,
         border_color = "grey95",
         main = "Comparative Fold Changes of Top 50 Most Variable Genes (Non-coding Excluded)")

dev.off()
print("SUCCESS: Full pipeline executed. Non-coding RNAs are cleared and verified images are written out!")
```

### GO Term Analysis
```bash
library(DESeq2)
library(clusterProfiler)
library(ggplot2)
library(GO.db) # Universal ontology translation dictionary

# ============================================================
# 1. PARSE CGD GO FILE (DETERMINE GENE MAPPINGS)
# ============================================================
print("Loading and parsing the official CGD GO file safely...")
gaf_raw <- read.table("gene_association.cgd", sep = "\t", comment.char = "!", 
                      header = FALSE, quote = "", fill = TRUE, stringsAsFactors = FALSE)

col2_ids   <- trimws(as.character(gaf_raw[, 2]))   # DB Object Symbol (e.g., EFG1)
col3_ids   <- trimws(as.character(gaf_raw[, 3]))   # DB Object Name (e.g., C1_10260W_B)
col11_ids  <- trimws(as.character(gaf_raw[, 11]))  # DB Object Synonym Name
go_terms   <- trimws(as.character(gaf_raw[, 5]))   # GO Identifier

normalize_id <- function(x) {
  toupper(gsub("[_\\-]", "", trimws(x)))
}

# Compile robust gene mapping matrix
go2gene_all <- rbind(
  data.frame(GO_ID = go_terms, Match_ID = normalize_id(col2_ids), stringsAsFactors = FALSE),
  data.frame(GO_ID = go_terms, Match_ID = normalize_id(col3_ids), stringsAsFactors = FALSE),
  data.frame(GO_ID = go_terms, Match_ID = normalize_id(col11_ids), stringsAsFactors = FALSE)
)
go2gene_clean <- go2gene_all[go2gene_all$GO_ID != "" & go2gene_all$Match_ID != "", ]
go2gene_clean <- unique(go2gene_clean)

# ============================================================
# 2. DEFINE MASTER MUTANT STRAIN TARGETS LIST
# ============================================================
mutant_targets <- list(
  "Dbl_vs_WT"  = res_dbl,
  "Kcs1_vs_WT" = res_kcs1,
  "Vip1_vs_WT" = res_vip1
)

# ============================================================
# 3. OUTER PIPELINE LOOP: ITERATE THROUGH EACH MUTANT STRAIN
# ============================================================
for (mutant_name in names(mutant_targets)) {
  
  print(paste("============================================================"))
  print(paste("STARTING GENE ONTOLOGY ANALYSIS FOR:", mutant_name))
  print(paste("============================================================"))
  
  res_go_df <- as.data.frame(mutant_targets[[mutant_name]])
  res_go_df$raw_id <- rownames(res_go_df)
  
  # Strip matching prefixes to sync layout matrix rows with database standards
  res_go_df$stripped_id <- gsub("^CAALFM_", "", res_go_df$raw_id)
  res_go_df$id_norm <- normalize_id(res_go_df$stripped_id)
  
  universe_genes <- res_go_df$id_norm
  sig_up <- res_go_df$id_norm[!is.na(res_go_df$padj) & res_go_df$padj < 0.05 & res_go_df$log2FoldChange >= 1.0]
  sig_down <- res_go_df$id_norm[!is.na(res_go_df$padj) & res_go_df$padj < 0.05 & res_go_df$log2FoldChange <= -1.0]
  
  # Sub-select references to limit memory footprint and match dimensions cleanly
  go2gene <- go2gene_clean[go2gene_clean$Match_ID %in% universe_genes, ]
  colnames(go2gene) <- c("GO_ID", "Gene_ID")
  
  print(paste("Synchronized Background Map Rows:", nrow(go2gene)))
  print(paste("Significant Hits Detected -> Up:", length(sig_up), "| Down:", length(sig_down)))
  
  # Split targets into directional vectors
  gene_lists <- list("Up_Regulated" = sig_up, "Down_Regulated" = sig_down)
  
  # ============================================================
  # 4. INNER LOOP: DIRECTIONAL GO ENRICHMENT & GRAPH EXPORTS
  # ============================================================
  for (direction in names(gene_lists)) {
    target_genes <- gene_lists[[direction]]
    print(paste("Processing sub-enrichment loop calculations for:", mutant_name, "-", direction))
    
    if (length(target_genes) < 5) {
      print(paste("Skipping direction:", direction, "- Insufficient target gene volume."))
      next
    }
    
    # Run custom hypergeometric profile matrices
    go_enrich_results <- enricher(
      gene          = target_genes,
      universe      = universe_genes,
      TERM2GENE     = go2gene,
      pvalueCutoff  = 1.0, 
      qvalueCutoff  = 1.0, 
      pAdjustMethod = "BH"
    )
    
    go_df <- as.data.frame(go_enrich_results)
    if (nrow(go_df) == 0) {
      print(paste("No overlap terms resolved for:", mutant_name, "-", direction))
      next
    }
    
    raw_hits <- go_df[go_df$pvalue < 0.05, ]
    print(paste("Successfully recorded", nrow(raw_hits), "trending terms matching filter criteria."))
    
    if (nrow(raw_hits) > 0) {
      
      # Query GO.db dictionary to map accession codes straight to true text pathways
      pathway_terms <- mget(go_df$ID, GOTERM, ifnotfound = NA)
      pathway_names <- sapply(pathway_terms, function(x) {
        if (is.na(x)) return(NA)
        return(Term(x))
      })
      
      # Overlay translated strings or provide fallback layouts
      go_df$Pathway_Description <- pathway_names
      go_df$Pathway_Description <- ifelse(!is.na(go_df$Pathway_Description) & go_df$Pathway_Description != "", 
                                          go_df$Pathway_Description, go_df$ID)
      
      # Construct structured hybrid axis label configuration
      go_df$Final_Plot_Label <- paste0(go_df$Pathway_Description, " (", go_df$ID, ")")
      
      # Save individual data spreadsheet files out to disk folder
      output_csv_name <- paste0(mutant_name, "_GO_Enrichment_", direction, "_AllTerms.csv")
      write.csv(go_df, file = output_csv_name, row.names = FALSE)
      
      # Slice top 15 highest ranking rows to stage plotting variables
      top_hits <- head(go_df[order(go_df$pvalue), ], 15)
      
      # FIXED MATH SPLITTER: Safely divides index 1 by index 2 to create a single numeric decimal value per row
      top_hits$GeneRatio_num <- sapply(top_hits$GeneRatio, function(ratio_str) {
        parts <- as.numeric(unlist(strsplit(ratio_str, "/")))
        return(parts[1] / parts[2])
      })
      
      # Re-order the display factor using the human-readable text labels
      top_hits$Final_Plot_Label <- factor(top_hits$Final_Plot_Label, levels = rev(top_hits$Final_Plot_Label))
      
      # Export expanded canvas graphical frames safely to file directory
      output_png_name <- paste0(mutant_name, "_GO_Enrichment_", direction, "_Dotplot.png")
      png(filename = output_png_name, width = 1600, height = 800, res = 150)
      
      p_custom_go <- ggplot(top_hits, aes(x = GeneRatio_num, y = Final_Plot_Label)) +
        geom_point(aes(size = Count, color = pvalue)) +
        scale_color_gradient(low = "firebrick2", high = "royalblue3") +
        labs(title = paste("Top GO Terms:", mutant_name, "-", direction),
             subtitle = "Calculations derived against custom Candida Genome Database references",
             x = "Gene Ratio (Enriched Genes / Total List)", y = "Enriched Biological Pathways",
             size = "Gene Count", color = "Raw P-Value") +
        theme_bw() +
        theme(
          axis.text.y = element_text(size = 9, color = "black", face = "bold"),
          axis.title.x = element_text(size = 11, face = "bold"),
          axis.title.y = element_text(size = 11, face = "bold"),
          plot.title = element_text(size = 13, face = "bold")
        )
      
      print(p_custom_go)
      dev.off()
      print(paste("Graphics successfully exported out to destination path:", output_png_name))
    }
  }
}
print("ALL MULTI-MUTANT GO ANALYSIS CALCULATIONS EXECUTED CLEANLY!")
```

