# Transcriptional-Differences-Between-Inositol-Pyrophosphate-Pathway-Mutants-in-Candida-albicans
summary...

## Paths
Raw Fastq files: /home/yc1201/rnaseq_ypd1/raw
FastQC results: /home/yc1201/rnaseq_ypd1/results/qc
Reference genome: /home/yc1201/rnaseq_ypd1/reference

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
Verdict: can move onto alignment without trimming
