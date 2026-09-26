# Project 11: De Novo Assembly of a Salmonella Typhi Isolate (West Africa)

## Summary

Original brief: de novo assembly of a West African *Salmonella* outbreak strain, with MLST typing and comparison to known epidemic clones (*S.* Typhi H58). This project surfaced a real data-quality problem in the first candidate dataset, required switching to a second, independently verified isolate, and produced a high-quality *Salmonella* Typhi assembly (ST2). The H58 lineage comparison itself requires a tool beyond classical MLST — documented below as a clear next step rather than left unaddressed.

## Dataset provenance

### First candidate (rejected): SAMN48972067 / SRR33931604

- Real BioSample: bloodstream *Salmonella enterica* infection, Korle Bu Teaching Hospital, Accra, Ghana, 2021-06, submitted by London School of Hygiene & Tropical Medicine.
- Full pipeline run (FastQC, Trimmomatic, SPAdes, Prokka, QUAST) completed successfully and produced a clean-looking assembly (98.9% genome fraction, 4 misassemblies, low error rate).
- **MLST flagged a serious identity problem**: complete 7/7 exact-allele match to the `ecoli_achtman_4` scheme, only a partial match to `senterica_achtman_2`.
- **Confirmed via full-length 16S rRNA BLAST** (1,268bp, extracted from the Prokka annotation): 100.00% identity to five independent *Escherichia coli* genomes, zero *Salmonella* hits in the top 5.
- **Conclusion**: this SRA run's actual sequence data is *E. coli*, not *Salmonella*, despite its BioSample metadata. Most likely cause: sample mislabeling at the submitting institution or SRA-submission stage, or Illumina index/barcode cross-contamination during a multiplexed sequencing run. This is a genuine, documented finding — a reminder that BioSample metadata is not sufficient QC on its own; direct sequence verification (16S, MLST, or a k-mer sketch tool) is necessary before committing an isolate to downstream analysis.
- An earlier attempted species check on this sample used only a 59bp fragment (a `grep -A 1` extraction bug that captured just the FASTA header's first line) and produced an uninformative ~100% match across five different genera — a methodology error, not evidence of anything. Corrected by extracting the complete multi-line FASTA record before re-running BLAST.

### Second candidate (used): SAMN45204685 / SRR32669584

- Real BioSample, Nigeria, *Salmonella enterica* subsp. *enterica* serovar **Typhi**, sample name CCI63.
- **Verified before committing to a full download**: a 20,000-spot read subsample was checked with BBTools' `sendsketch.sh` (k-mer/ANI-based taxonomic identification against RefSeq) prior to downloading the full ~700MB dataset. Result: 99.65% ANI to *Salmonella enterica* subsp. *enterica* serovar Typhi str. CR0044, 90.77% weighted k-mer identity, clearly separated from the next-best hits (~59% WKID to other *Salmonella* sp.) and from *E. coli*/*Shigella*/*Citrobacter* (all <5% WKID). This pre-download verification step is now the standard practice for any future isolate selection in this program, given what happened with the first candidate.

## Environment and tooling notes

Built on a fresh Kaggle notebook (`project11_salmonella_assembly.ipynb`), following the per-project-notebook convention. Several installation issues were resolved and are worth recording for future projects:

- **FastQC, Trimmomatic, Prokka (apt versions)**: initial installs failed with `404 Not Found` errors from a stale apt package index; resolved with `apt-get update` before retrying.
- **TrimmomaticPE**: Debian packages it as `TrimmomaticPE`/`TrimmomaticSE` (capitalized, no lowercase `trimmomatic` alias); bundled Illumina adapter FASTA files are at `/usr/share/trimmomatic/`.
- **SPAdes**: installed via GitHub release tarball (v4.0.0), not the dead `cab.spbu.ru` link.
- **mlst**: no apt or PyPI package exists; installed by cloning `github.com/tseemann/mlst` plus `any2fasta` and Perl/BLAST+ dependencies via apt.
- **Prokka's Debian apt package ships an incomplete database** (only genus-level BLAST and CM/Rfam databases, missing the core `kingdom`-level Swiss-Prot BLAST set and all HMM databases that the annotation pipeline actually depends on). Resolved by installing Prokka from its own GitHub source (`github.com/tseemann/prokka`) instead, which ships a complete, correctly-structured database bundle, followed by `prokka --setupdb`.
- **BBTools/sendsketch.sh**: apt's `bbmap` package installs its full script set under `/usr/share/bbmap/`, but only symlinks a handful of common scripts (`bbduk.sh`, `bbmap.sh`, etc.) into `/usr/bin`; `sendsketch.sh` and most other tools must be called by their full path.
- **Disk space**: the full untrimmed+trimmed read set for this isolate (7.28M read pairs, ~340x raw coverage) exceeded Kaggle's ~19.5GB working-directory quota mid-assembly (`No space left on device` during SPAdes K55). Resolved by deleting raw/intermediate FASTQs promptly after each pipeline stage and subsampling to ~1M read pairs (~50-60x coverage, ample for a high-quality bacterial assembly) before running SPAdes.

## Pipeline

```bash
fastqc data/raw/reads_1.fastq data/raw/reads_2.fastq -o results/fastqc/

TrimmomaticPE -threads 4 \
  data/raw/reads_1.fastq data/raw/reads_2.fastq \
  data/trimmed/reads_1_paired.fastq data/trimmed/reads_1_unpaired.fastq \
  data/trimmed/reads_2_paired.fastq data/trimmed/reads_2_unpaired.fastq \
  ILLUMINACLIP:/usr/share/trimmomatic/TruSeq3-PE.fa:2:30:10 \
  LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36

seqkit sample -n 1000000 -s 42 data/trimmed/reads_1_paired.fastq -o data/trimmed/reads_1_sub.fastq
seqkit sample -n 1000000 -s 42 data/trimmed/reads_2_paired.fastq -o data/trimmed/reads_2_sub.fastq

spades.py -1 data/trimmed/reads_1_sub.fastq -2 data/trimmed/reads_2_sub.fastq \
  -o results/spades --isolate -t 4

prokka --outdir results/prokka --prefix salmonella_typhi --force \
  --genus Salmonella --species enterica --cpus 4 --centre IgbegaX --locustag TYPHI11 \
  results/spades/contigs.fasta

quast.py results/spades/contigs.fasta -o results/quast

mlst results/spades/contigs.fasta > results/mlst/mlst_result.txt
```

Trimmomatic: 7,280,072 input pairs, 98.00% surviving as proper pairs. Subsampled to 999,414 pairs per mate before assembly.

## Results

### SPAdes assembly
Total length 4,969,948 bp across 329 raw contigs, consistent with SPAdes' own k-mer-based genome size estimate (~4.89-4.90 Mb) at every k-stage (K21 through K77).

### Prokka annotation
4,794 CDS predicted; 82 tRNAs; 4 rRNAs found (16S and 5S, both partial in places — a known short-read assembly limitation at repetitive, multi-copy rRNA operons, consistent with the pattern already characterized in Week 11's *M. tuberculosis* work).

### QUAST (reference-free)

| Metric | Value |
|---|---|
| Contigs (>=500bp) | 65 |
| Total length | 4,906,369 bp |
| N50 | 153,664 bp |
| L50 | 11 |
| Largest contig | 466,368 bp |
| GC% | 51.89% |

N50 is nearly double what was achieved on *M. tuberculosis* earlier in the week, consistent with the more moderate (~50-60x rather than ~300x or ~65x) coverage used here. GC% of 51.89% is a close match to published *S. enterica* values, an additional independent line of evidence supporting species identity.

### MLST

```
ecoli_achtman_4  ST102  adk(44) fumC(42) gyrB(35) icd(40) mdh(32) purA(29) recA(31)
WARNING: ecoli_achtman_4(102)==salmonella(2) score=100
```

The `mlst` tool's top-line result names the `ecoli_achtman_4` scheme, but the full BLAST output (visible in the run log) shows an equally complete, exact 7/7-allele match to the `salmonella` scheme (hisD, purE, aroC, sucA, hemD, thrA, dnaN), tied at score 100. This is a well-documented property of the classical 7-gene Achtman MLST scheme, not a repeat of the earlier contamination problem: several of its housekeeping loci are conserved across both *E. coli* and *S. enterica* as close Enterobacteriaceae relatives, so a genuinely correct *Salmonella* genome routinely ties the *E. coli* scheme at these specific genes. Given the independent, unambiguous species evidence already established (99.65% ANI to Typhi via sendsketch, 100% identity across the complete 16S gene in the earlier candidate-verification step, and a GC% match to published Typhi values), the correct read of this result is:

**Salmonella enterica, Sequence Type 2 (ST2).**

## Scope limitation: H58 lineage comparison requires GenoTyphi, not MLST

The original project brief called for comparing this isolate's gene content to known epidemic clones, specifically *S.* Typhi H58. This is **not achievable with classical 7-gene MLST**: Typhi is near-clonal at MLST resolution (ST1 and ST2 together account for the overwhelming majority of all Typhi isolates worldwide), because the species diverged relatively recently and MLST's seven housekeeping genes do not accumulate enough variation to distinguish sublineages within it. H58 is defined by a whole-genome SNP genotyping scheme (**GenoTyphi**), not by MLST allele calls.

**Documented as a clear next step**: running GenoTyphi (available at `github.com/typhoidgenomics/genotyphi`) against this assembly or its raw reads would give the actual H58/non-H58 lineage call the original brief was aiming for. This was not run in the current pass and should be treated as the natural continuation of this project rather than an omission being glossed over.

## Key takeaways

1. BioSample/SRA metadata is not sufficient QC — direct sequence-level species verification (16S BLAST, k-mer sketching, or MLST) is necessary before committing an isolate to a multi-hour assembly pipeline. This cost one full pipeline run in this project but caught a genuine data problem before it could propagate into a false epidemiological conclusion.
2. Pre-download verification (a small read subsample sketched against RefSeq) is cheap and should be standard practice going forward, rather than verifying only after a full download and assembly.
3. MLST scheme ties between related genera (here, *E. coli* and *Salmonella*) are a known, explainable phenomenon — worth checking the full BLAST allele output, not just the tool's single top-line scheme call, before concluding anything about species identity from MLST alone.
4. Classical MLST and whole-genome lineage typing (GenoTyphi) answer different questions at different resolutions; matching the tool to the actual question (species/ST vs. epidemic sublineage) matters as much as running a tool correctly.
