# Week 11 Summary: Read Mapping & Genome Assembly

## Topic
Read mapping theory and practice, SAM/BAM format, SAMtools, de novo genome assembly, and assembly quality control — culminating in Project 11, a real-world de novo assembly and species/strain identification exercise.

## Daily breakdown

**Day 1 — Read mapping theory (seed-and-extend, BWT/FM-index, BWA-MEM)**
Mapped real *M. tuberculosis* H37Rv WGS reads (ERR17622113, Institut Pasteur Lille) to the reference genome: 99.73% mapped, 99.50% properly paired. Noted a small number of FF-orientation read pairs as a possible library-prep artifact worth watching in later work.

**Day 2 — SAM/BAM format (flags, CIGAR, mate pairs)**
Manually parsed BAM FLAG bits with pysam and cross-checked against `samtools flagstat`. Found and explained a 360-read discrepancy in "properly paired" counts: caused by supplementary alignments inheriting their primary record's proper-pair bit, resolved by filtering to primary-only alignments.

**Day 3 — SAMtools (sort, index, flagstat, depth)**
Computed per-base coverage depth (mean 65.45x). Investigated 147 statistically-impossible zero-coverage positions and traced all nine contiguous blocks to three distinct causes: an IS6110 insertion-sequence inverted repeat, two PE_PGRS paralogous genes, and six loci with extreme local GC/base-composition bias confirmed by direct sequence inspection.

**Day 4 — De novo assembly (de Bruijn graphs, SPAdes)**
Assembled H37Rv from scratch: 98.9% genome recovered, N50 63,218bp, only 4 misassemblies. Traced each misassembly to a specific, distinct mechanism (PE_PGRS4, the mma operon, the ribosomal S10-spc-alpha operon, and a genuine local coverage dip confirmed against Day 3's own depth data) rather than a generic "repeats caused it" explanation.

**Day 5 — Assembly QC (N50/L50, BUSCO) and SPAdes vs. MEGAHIT comparison**
Installed and ran BUSCO and MEGAHIT (neither has a standard package manager entry point) as a second assembler for comparison. SPAdes: 153 contigs, 4 misassemblies, 3.72 mismatches/100kbp. MEGAHIT: 171 contigs, 9 misassemblies, 10.69 mismatches/100kbp, marginally higher genome fraction. Concluded SPAdes was the better choice for Project 11's epidemiological use case.

**Project 11 — De novo assembly of a real outbreak isolate**
Two significant findings:
1. The first candidate isolate (BioSample-labeled as *Salmonella* from Ghana) was found, via full 16S rRNA BLAST and MLST evidence, to actually be *E. coli* — a real sample mislabeling or cross-contamination issue caught only through direct sequence verification, after a full pipeline run had already completed on it.
2. Switched to a second, pre-verified isolate (a real *Salmonella* Typhi sample from Nigeria), assembled it successfully (N50 153,664bp, 65 contigs), and confirmed species/sequence type as *Salmonella enterica* ST2 via MLST. Documented that the original goal of comparing against the H58 epidemic clone requires GenoTyphi (whole-genome SNP typing), not classical MLST, since Typhi is near-clonal at MLST resolution — flagged as the clear next step.

## Key takeaways from the week

- Verifying a sample's actual identity (16S BLAST, k-mer sketching) before committing to a multi-hour pipeline run is essential; BioSample metadata alone is not sufficient QC.
- Coverage gaps, misassemblies, and MLST scheme ties each have specific, traceable mechanistic explanations — the discipline of chasing each one down to a concrete cause (rather than a generic explanation) was the throughline connecting every day this week.
- Environment/tooling issues were frequent and varied (dead download links, incomplete Debian packages, disk-quota limits, missing package-manager entries) and each was resolved by identifying the actual root cause rather than guessing.

## Environment
All heavy compute for this week ran on Kaggle notebooks (local PC insufficient for WGS-scale read mapping and assembly). Tools requiring non-standard installation this week: SPAdes (GitHub release, not the dead cab.spbu.ru link), BUSCO (GitLab source + pip install), MEGAHIT (GitHub release binary), mlst (GitHub source + any2fasta), Prokka (GitHub source, since the Debian apt package ships incomplete databases).
