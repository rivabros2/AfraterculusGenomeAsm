# AfraterculusGenomeAsm
## “Chromosome-scale genome assembly of the South American fruit fly, Anastrepha fraterculus sp. 1”


# Abstract:


This study presents the results of whole-genome sequencing and chromosome-scale assembly derived from single female and male of the South American fruit fly, Anastrepha fraterculus Brazilian 1 morphotype (or sp.1).  Additionally, we provide a complete assembly of the mitochondrial genome and analyzed the key nuclear and mitochondrial genomic features, including coding genes, repetitive regions, and ribosomal RNA. These findings offer valuable insights for population genomics and comparative evolutionary studies. Furthermore, our research contributes to the characterization of potential target genes for genetic engineering (e.g., gene editing) and RNAi technologies, which could complement existing pest control strategies, such as the sterile insect technique, or aid in the development of novel biocontrol methods.  

# See NCBI Bioprojects to obtain all raw data associated with this project (bottom of this file)


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
11. [9. Contacts & References](#9-contacts--references)

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
  - GetOrganelle v1.7.0,
  - GenSAS website,
  - OrthoVenn3,
  - NGenomeSyn  
- **Hardware:**  
  - ≥ 90 CPUs, ≥ 800 GB RAM (Canu Asm)  
  - ≥ 150 GB tmp (Pairtools sort)  

---

## Project Layout



![image](https://github.com/user-attachments/assets/e0fa3b07-bf2c-43ef-8da4-1266adf747ee)




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


## 2. Genome Assembly
# Female assembly
canu -p afr_female -d results/assembly/female \
  genomeSize=615m \
  -useGrid=false -maxMemory=800g -maxThreads=90 \
  -nanopore-raw data/raw_reads/female_reads.fastq.gz

# Male assembly
canu -p afr_male -d results/assembly/male \
  genomeSize=615m \
  -useGrid=false -maxMemory=800g -maxThreads=90 \
  -nanopore-raw data/raw_reads/male_reads.fastq.gz




## 3. Polishing & Purging
# Polish with Jasper
jasper.sh \
  --jf results/qc/Female.jf \
  -a results/assembly/female/afr_female.contigs.fasta \
  -k 29 -t 40 -p 4
jasper.sh \
  --jf results/qc/Male.jf \
  -a results/assembly/male/afr_male.contigs.fasta \
  -k 29 -t 40 -p 4

# Purge duplicates
purge_dups/run_purge_dups.py \
  -p bash \
  results/assembly/female/afr_female_Polished.json \
  /path/to/purge_dups/bin/ \
  results/purge_dups/female

purge_dups/run_purge_dups.py \
  -p bash \
  results/assembly/male/afr_male_Polished.json \
  /path/to/purge_dups/bin/  results/purge_dups/male

## 4. Hi-C Processing & Scaffolding
# Map & parse
bwa mem -5SP -T0 -t 20 \
  results/purge_dups/male.fa.polished \
  data/hic_reads/male_R1.fastq.gz \
  data/hic_reads/male_R2.fastq.gz \
  > results/hic/male.sam

pairtools parse \
  --min-mapq 40 --walks-policy 5unique --max-inter-align-gap 30 \
  --nproc-in 10 --nproc-out 10 \
  --chroms-path results/purge_dups/male.fa.polished.fai \
  -o results/hic/parsed.txt \
  --output-stats results/hic/parse_stats.txt \
  results/hic/male.sam


# Sort & dedup
pairtools sort \
  --memory 150G --tmpdir=/tmp --nproc 15 \
  -o results/hic/sorted.pairsam \
  results/hic/parsed.txt

pairtools dedup \
  --mark-dups \
  --nproc-in 10 --nproc-out 10 \
  -o results/hic/dedup.pairsam \
  --output-stats results/hic/dedup_stats.txt \
  results/hic/sorted.pairsam


# Scaffold with YAHS
yahs \
  -o results/hic/yahs_scaffolds \
  --no-mem-check \
  -r 2000,8000,15000000 \
  --read-length 150 \
  results/purge_dups/male.fa.polished \
  results/hic/dedup.pairsam.bed


## Reference‐guided scaffolding (RagTag)
# Female → Male
ragtag.py scaffold -t 40 \
  --aligner minimap2 \
  -o results/scaffold/f2m \
  results/purge_dups/female.fa.polished \
  results/scaffold/yahs_scaffolds.fa

# Male → Female (no breaks)
ragtag.py scaffold -t 40 \
  --aligner minimap2 \
  -o results/scaffold/m2f \
  --no-breaks \
  results/scaffold/f2m/ragtag.scaffold.fasta \
  results/purge_dups/male.fa.polished



## 5. Assembly Assessment

busco \
  -m genome -c 30 \
  -i results/scaffold/m2f/ragtag.scaffold.fasta \
  -o busco_male \
  -l diptera_odb10

busco \
  -m genome -c 30 \
  -i results/scaffold/f2m/ragtag.scaffold.fasta \
  -o busco_female \
  -l diptera_odb10


## 6. K-mer Comparisons
kat plot spectra-mx \
  -t "Male Reads vs Assembly" \
  -i results/kat/maleReads_assembly.mx \
  -o results/kat/male_spectra.png


## 7. Mitochondrial Genome
get_organelle_from_reads.py \
  -1 data/raw_reads/male_R1.fastq.gz \
  -2 data/raw_reads/male_R2.fastq.gz \
  -F animal_mt \
  -o results/mito/male_mt \
  -R 10 -t 25


## 8. Annotation & Synteny
# Annotation (GenSAS)
The steps on the flowcharts with the same color represent the major steps of annotation:

    Project creation/uploading sequences and evidence
    Repeat identification and masking (for eukaryotes)
    Structural annotation
    Selection of official gene set (and refinement for eukaryotes)
    Functional annotation
    Manual curation of the annotation
    Preparing files for publication



![image](https://github.com/user-attachments/assets/81103076-9c39-4c04-b87b-45d967725efb)



    Upload results/scaffold/*/ragtag.scaffold.fasta

    Configure gene prediction, repeat masking, functional annotation

# Tools used in GenSas v6:
https://www.gensas.org/tools

see Info in GenSas 


# Comparative genomics (OrthoVenn3 / NGenomeSyn)

GetTwoGenomeSyn.pl \
  -InGenomeA results/annotation/female.fasta \
  -InGenomeB annotation/AnaLude1.1_genomic.fna \
  -OutPrefix results/synteny/female_vs_analu \
  -MappingBin minimap2 \
  -BinDir /path/to/minimap2/ \
  -MinLenA 50000 -MinLenB 50000 \
  -NumThreads 25


## 9. Contacts & References

NCBI BioProjects:

PRJNA1145343 : Anastrepha fraterculus isolate:IGEAF | breed:sp1 Genome sequencing (TaxID: 95504) 


PRJNA1145375 : Anastrepha fraterculus isolate:IGEAF Genome sequencing (TaxID: 95504)




-------------------------------------------------------------------


