# AfraterculusGenomeAsm
“Chromosome-scale genome assembly of the South American fruit fly, Anastrepha fraterculus sp. 1”


#Abstract:


This study presents the results of whole-genome sequencing and chromosome-scale assembly derived from single female and male of the South American fruit fly, Anastrepha fraterculus Brazilian 1 morphotype (or sp.1).  Additionally, we provide a complete assembly of the mitochondrial genome and analyzed the key nuclear and mitochondrial genomic features, including coding genes, repetitive regions, and ribosomal RNA. These findings offer valuable insights for population genomics and comparative evolutionary studies. Furthermore, our research contributes to the characterization of potential target genes for genetic engineering (e.g., gene editing) and RNAi technologies, which could complement existing pest control strategies, such as the sterile insect technique, or aid in the development of novel biocontrol methods.  


# Anastrepha fraterculus Genome & Hi-C Assembly Pipeline

> **Maintainer:** Maximo Rivarola
> **Last updated:** 2025-04-30

## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Project Layout](#project-layout)  
3. [1. Quality Control](#1-quality-control)  
4. [2. Genome Assembly](#2-genome-assembly)  
5. [3. Polishing & Purging](#3-polishing--purging)  
6. [4. Hi-C Processing & Scaffolding](#4-hi-c-processing--scaffolding)  
7. [5. Assembly Assessment](#5-assembly-assessment)  
8. [6. K-mer Comparisons](#6-k-mer-comparisons)  
9. [7. Mitochondrial Genome](#7-mitochondrial-genome)  
10. [8. Annotation & Synteny](#8-annotation--synteny)  
11. [Contacts & References](#contacts--references)

---

## Prerequisites

- **OS:** Linux (bash)  
- **Tools & Versions:**  
  - FastQC v0.11.9  
  - Jellyfish v2.3.0, GenomeScope v2.0  
  - Canu v2.2  
  - Jasper v1.0  
  - Purge Dups (latest)  
  - BWA v0.7.17, Pairtools v0.3.0  
  - YAHS v1.0, RagTag v2.1.0 (minimap2 aligner)  
  - BUSCO v5.4.5 (`diptera_odb10`)  
  - KAT v2.4.2  
  - GetOrganelle v1.7.0, GenSAS, OrthoVenn3, NGenomeSyn  
- **Hardware:**  
  - ≥ 90 CPUs, ≥ 800 GB RAM (Canu)  
  - ≥ 150 GB tmp (Pairtools sort)  

---

## Project Layout



---

## 1. Quality Control

1. **FastQC**  
   ```bash
   fastqc -t 16 -o results/qc/ data/raw_reads/*.fastq.gz

zcat data/raw_reads/female_reads.fastq.gz | wc -l
zcat data/raw_reads/male_reads.fastq.gz   | wc -l

# Female
zcat female_reads.fastq.gz \
  | jellyfish count -C -m 21 -s 1e11 -t 20 -o results/qc/Female.jf
jellyfish histo -t 30 results/qc/Female.jf > results/qc/Female.histo

# Male
zcat male_reads.fastq.gz \
  | jellyfish count -C -m 21 -s 1e11 -t 20 -o results/qc/Male.jf
jellyfish histo -t 30 results/qc/Male.jf > results/qc/Male.histo

# GenomeScope K-mer analysis
genomescope2 -i results/qc/Female.histo -o results/qc/genomescope_female
genomescope2 -i results/qc/Male.histo   -o results/qc/genomescope_male


2. Genome Assembly
# Female assembly
canu \
  -p afr_female -d results/assembly/female \
  genomeSize=615m \
  -useGrid=false -maxMemory=800g -maxThreads=90 \
  -nanopore-raw data/raw_reads/female_reads.fastq.gz

# Male assembly
canu \
  -p afr_male -d results/assembly/male \
  genomeSize=615m \
  -useGrid=false -maxMemory=800g -maxThreads=90 \
  -nanopore-raw data/raw_reads/male_reads.fastq.gz



