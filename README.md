
# minialign [![Build Status](https://travis-ci.org/ocxtal/minialign.svg?branch=master)](https://travis-ci.org/ocxtal/minialign) [![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg)](http://bioconda.github.io/recipes/minialign/README.html)

Minialign is a little bit fast and moderately accurate nucleotide sequence alignment tool designed for PacBio and Nanopore long reads. It is built on three key algorithms, minimizer-based index of the [minimap](https://github.com/lh3/minimap) overlapper, array-based seed chaining, and SIMD-parallel Smith-Waterman-Gotoh extension.

## Announcements

* Latest: 0.5.2; stable: 0.4.4
* **2016/4/27: Version 0.5.2:** Fix a bug in falcon\_sense fomat (`-Ofalcon`).
* **2016/4/3: Version 0.5.0 is released.** New features: SA (supplementary alignment) and MD (mismatch position) tags are enabled with `-TSA` and `-TMD` flags.
* **Please do not use 0.4.5 and 0.4.6**, since they have bugs in multithreading.

## Getting started

C99 compiler (gcc / clang / icc) is required to build the program.

```
$ make && make install	# PREFIX=/usr/local by default
$ minialign -xont reference.fa reads.[fa,fq,bam] > read_to_ref.sam
$ minialign -X -xava reads.[fa,fq,bam] > all_versus_all.paf
```

Reference sequence index can be stored in separate. Using prebuilt index saves around a minute per run for a human haploid (~3G) genome.

```
$ minialign -d index.mai reference.fa	# build index
$ minialign -l index.mai reads.[fa,fq,bam] > out.sam	# mapping on prebuilt index
```

Frequently used options are: scoring parameters, minimum score cut-offs, and number of threads.

```
$ minialign -a1 -b2 -p2 -q1		# match, mismatch, gap-open and gap-extend
$ minialign -s1000	# set minimum score threshold to 1000
$ minialign -m0.8	# set report threshold at 0.8 of the highest score for every read
$ minialign -t10	# minialign is now 10x faster!!!
```

## Benchmarks

All the following benchmarks were took on Intel i5-6260U (Skylake, 2C4T, 2.8GHz, 4MBL3) with 32GB (DDR4, 2133MHz) RAM.

### Speed

|                      Time (sec.)                     |  minialign  |   DALIGNER  |   BWA-MEM   |
|:----------------------------------------------------:|:-----------:|:-----------:|:-----------:|
| E.coli (MG1655) x100 simulated read (460Mb) to ref.  |        10.0 |        39.5 |        6272 |
| S.cerevisiae (sacCer3) x100 sim. (1.2Gb) to ref.     |        29.3 |       1134* |       10869 |
| D.melanogaster (dm6) x20 sim. (2.75Gb) to ref.       |        82.6 |           - |       31924 |
| Human (hg38) x3 sim. (9.2Gb) to ref.                 |         648 |           - |           - |

Notes: Execution time was measured with the unix `time` command, shown in seconds. Dashes denote untested conditions. Program version information: minialign-0.4.6, DALIGNER-ca167d3 (commit on 2016/9/27), and BWA-MEM-0.7.15-r1142-dirty. All the programs were compiled with gcc/g++-5.4.0 providing the optimization flag `-O3`. PBSIM (PacBio long-read simulator), [modified version based on 1.0.3 (1.0.3-nfree)](https://github.com/ocxtal/pbsim/tree/nfree) not to generate reads containing N's, was used to generate read sets. Simulation parameters (len-mean, len-SD, acc-mean, acc-SD) were fixed at (20k, 2k, 0.88, 0.07) in all the samples. Arguments passed to the programs; minialign: `-t4 -xpacbio`, DALIGNER: `-T4`, and BWA-MEM: `-t4 -xpacbio -A1 -B2 -O2 -E1` (overriding scoring parameters based on the PacBio defaults). Index construction (minialign and BWA-MEM) and format conversion time (DALIGNER: fasta -> DB, las -> sam) are excluded from the measurements. Peak RAM usage of minialign was around 12GB in human read-to-ref mapping with four threads. Starred sample, S.cerevisiae on DALIGNER, was splitted into five subsamples since the whole concatenated fastq resulted in an out-of-memory error. Calculation time of the subsamples were 61.6, 914.2, 56.8, 50.3, and 51.5 seconds, where the second trial behaved a bit strangely with too long calculation on one (out of four) threads.

### Recall-precision trend

![Recall-precision trend (D.melanogaster)](https://github.com/ocxtal/minialign/blob/master/fig/rec_prec.dm6.png)

(a). D.melanogaster (dm6) x10 (1.4Gb)

![Recall-precision trend (Human)](https://github.com/ocxtal/minialign/blob/master/fig/rec_prec.hg38.png)

(b). Human (hg38) x0.3 (1Gb)

Notes: Recall is defined as: a proportion of reads whose originating region, represented by (spos, epos) coordinate pair on the reference, has at least one non-zero-length intersection with its alignments. Precision is defined as: a proportion of alignment records which has a non-zero-length intersection with its originating region. The recall and precision pairs were calculated from the output sam files, filtered with different mapping quality thresholds between 0 and 60. Duplicate alignments were not filtered out from the output sam files. Program version information: minialign-0.4.6, GraphMap-0.4.0, BLASR-0014a57 (the last commit with the SAM output), and BWA-MEM-0.7.15-r1142-dirty. Read set was generated by the PBSIM-1.0.3-nfree with the parameters (len-mean, len-SD, acc-mean, acc-SD) set to (10k, 10k, 0.8, 0.2) without ALT / random contigs. Reads were mapped onto the corresponding reference genomes including ALT / random contigs. Arguments passed to the programs; minialign: `-t4 -xpacbio`, GraphMap: `-t 4`, BLASR: `--nproc 4`, and BWA-MEM: `-t4 -xpacbio`. Calculation time and peak RAM usages are shown in the table below.

|     Time (sec.) / Peak RAM (GB)       |  minialign  |   GraphMap  |    BLASR    |   BWA-MEM   |
|:-------------------------------------:|:-----------:|:-----------:|:-----------:|:-----------:|
| D.melanogaster (dm6) x10 sim. (1.4Gb) |  51.3 / 2.2 |  6482 / 4.3 | 30081 / 1.0 | 37292 / 0.5 |
| Human (hg38) x0.3 sim. (1Gb)          |   87.4 / 12 |           - |           - | 34529 / 5.5 |

### Effect of read length and score threshold on recall

![readlength-recall trend (Human)](https://github.com/ocxtal/minialign/blob/master/fig/rec_len.hg38.png)

Notes: The solid lines in the figure shows the proportions of mapped reads. The dashed lines shows the recalls, defined as above. The minimum alignment score threshold (`-s`) was differed among 50, 100, 200, and 400. Reads were generated from hg38 without ALT / random contigs using PBSIM-1.0.3-nfree with the (len-mean, len-SD, acc-mean, acc-SD) parameters set to (2000, 2000, 0.88, 0.07). Reads were mapped onto the reference with ALT / random contigs included. Minialign-0.4.4 was run with the `-t4 -xpacbio` flags and the additional `-s` parameters.

## Algorithm overview

The algorithm is roughly based on the seed-and-extend strategy with additional seed filtering and chaining stages. The seed filtering and chaining stages are essential to filter out futile or redundant extension trials, where long and erroneous query sequences resulting in numerous false positives and a small amount of consecutive correct hits. The effectiveness of the seed chaining is first shown by Chaisson and Tessler in the BLASR algorithm [1], and also in the BWA-MEM [2]. The additional filtering stage is introduced by the GraphMap [3], improved overall computational performance combined with the chaining algorithm. The minimap algorithm [4] adopted the minimizer [5] and an invertible hash function [6] to reduce seeds to be enumerated at the indexing stage.

### Minimizer-based index structure

Minialign also adopted the invetible hash function [6] and the minimizer-based seed filtering [4] of the minimap. The same indexing parameters, the k-mer length and the window size, are adopted as the default value, sustaining the equivalent information as the minimap program. Some modifications were applied to improve simplicity and support a reference subsequence retrieval query.

### Array-based seed chaining

An array-based seed chaining algorithm was introduced to efficiently detect a stream of seeds without capturing large insertions and deletions. In each chain extension trial, the downstream seeds are filterd with a "chainable window", which is a 30-degree-angled parallelogram placed at the right-bottom direction of the tailmost seed, preventing it from introducing unrealistically large gaps in the chain.

The whole chaining algorithm with an incremental repetitive-seed filtering was implemented as follows;

```
0.  occ[] is a precalculated array of occurrences correspond to the top [5%, 1%, 0.1%]
    frequent minimizers on the reference.
1.  Collect minimizers on the query to a seed bin.
2.  for (i = 0; there remains elements in the seed bin; i++) {
3.    Clear a result bin.
4.    Move seeds which occurs less than a threshold, occ[i], to a chain array.
5.    Sort the elements in the chain array by their `rpos - 2*qpos` values.
      Mark them as unchained.
6.    while (there remains unchained elements in the chain array) {
7.      Mark the first unchained seed in the array as a root of a chain.
        Initialize a tail-of-chain pointer to point at the root.
8.      if (the next seed in the chain array is inside the chainable window of the tail) {
9.        Mark it as chained and move the tail pointer to it.
10.     } else if (there is no seed in the chainable window) {
11.       Terminate the extension and save the resulting chain in the result bin, continue to 6.
        }
      }
12.   if (there is no meaningful chain in the result bin) { Then continue to 2. }
13.   Return the result bin;
    }
```

The rectangular-inclusion test on the line 8 is implemented with comparisons of `rpos - qpos/2` values of the two adjacent seeds, which are ordered by `rpos - 2*qpos` in the previous sort.

### Smith-Waterman-Gotoh extension

The second head seed of each chain is extended upward (3' on the reference side) then downward from the maximum score position found. If the resulting path is shorter than the seed chain span, similar extension is repeatedly performed on the next neighboring seed of the tail of the obtained alignment path. Each extension is carried out by the GABA library [7], which implements the adaptive-banded semi-global Smith-Waterman-Gotoh algorithm [8] with the difference recurrence relations [9].

### References

1. Mark J Chaisson and Glenn Tesler (2012). Mapping single molecule sequencing reads using basic local alignment with successive refinement (BLASR): application and theory. *BMC bioinformatics*, 13(1), 238.
2. Heng Li (2013). Aligning sequence reads, clone sequences and assembly contigs with BWA-MEM. *arXiv preprint* arXiv:1303.3997.
3. Ivan Sović, et.al. (2016). Fast and sensitive mapping of nanopore sequencing reads with GraphMap. *Nature communications*, 7.
4. Heng Li (2016). Minimap and miniasm: fast mapping and de novo assembly for noisy long sequences. *Bioinformatics*, btw152.
5. Michael Roberts, et. al. (2004). Reducing storage requirements for biological sequence comparison. *Bioinformatics*, 20(18), 3363-3369.
6. Heng Li (2014). Invertible integer hash functions. *GitHub Gist*, [https://gist.github.com/lh3/974ced188be2f90422cc](https://gist.github.com/lh3/974ced188be2f90422cc)
7. Hajime Suzuki (2016). libgaba: Adaptive semi-global banded alignment on string graphs. *GitHub*, [https://github.com/ocxtal/libgaba](https://github.com/ocxtal/libgaba)
8. Hajime Suzuki (2016). Assessment on adaptive-banded dynamic programming algorithm for the nucleotide semi-global alignment. *GitHub*, [https://github.com/ocxtal/adaptivebandbench](https://github.com/ocxtal/adaptivebandbench)
9. Hajime Suzuki (2016). Benchmark on difference recurrence relation. *GitHub*, [https://github.com/ocxtal/diffbench](https://github.com/ocxtal/diffbench)

## FAQs and recipes

### Is the minialign applicable to Illumina datasets?

Generally, no. The seed-chaining algorithm is not good at detecting short-spanning and dense seed chain caused by short and high-identity Illumina reads. The read length-recall trend (the red and green lines in the figure in the third benchmark) shows that the minialign tends to fail collecting alignments when the read lengths are shorter than 1000 bases regardless the score thresholds.

### Mapped read ratio is slightly low

First of all, check the post-filter thresholds `-s`: minimum score and `-m`: report threshold to the highest scoring alignment. The default values `-s50` and `-m0.3` is roughly works well on typical PacBio long reads and ONT 1D/2D reads whose read lengths range above 300 bases but sometimes not being suitable for specific datasets.

If the result did not improve after adjusting (generally lowering) `-s` and `-m` values, try smaller k-mer lengths (`-k`) and minimizer window sizes (`-w`). Smaller values like `-k14` and `-w5` will slightly improve sensitivity (especially on the all-versus-all alignment tasks) but significantly increase calculation time.

Adjusting alignment scoring parameters `-a`, `-b`, `-p`, and `-q` (match award, mismatch penalty, gap-open penalty, and gap-extension penalty) is also preferred if you are aware of identity between read sets and reference sequences. Increasing `-b` and `-p` by one or two (like `-a1 -b2 -p2 -q1`) may perform better detecting correct mapping position on relatively high-identity (90-95% between reads and references) read sets.

### Transferring optional information in input files (quality strings, bam tags, ...) to output sam

* **Quality strings in fastq / bam** Add `-Q` flag. Note that minialign does not utilize, nor verify the content of the quality strings in its alignment routine.
* **Comments in the input fasta/q** Add `-UCO` flag. Each tab (`\t`) in the lines will be replaced by a space.
* **Optional tags in the input bam** Add `-U` flag with a list of tags to be transferred, for example, `-UAX,XS`.

### Adding @RG tag in the sam header

Passing `-TRG` flag adds the default `@RG	ID:1` line and the corresponding `RG:Z:1` tag in each alignment record. If you need more specific line, pass `-R` flag, which is the same as the BWA-MEM's `-R` option, like `-R"@RG\tID:foo\tSM:bar"`.

### Applying multiple query / reference files

#### Index construction mode

When minialign is invoked with the `-d index_filename` flag, it switches into the index construction mode. All the extra positional arguments are interpreted as reference sequence filenames, converted to indices, and dumped into a single file.

```
$ minialign -d index.mai chr1.fa chr2.fa ... chrX.fa	# put all chromosomes into one index
```

#### Read-to-reference mapping mode

In the read-to-reference mapping mode (the default configuration), extra positional arguments are interpreted as a list of query files. The output is a single sam file with all the input reads contained.

```
$ minialign reference.fa read1.fa read2.fa ... readN.fa > out.sam
$ minialign -l index.mai read1.fa read2.fa ... readN.fa > out.sam	# also compatible with prebuilt index
$ minialign -l index.mai reads/* > out.sam		# map all files in a directory
```

#### All-versus-all mapping mode

Minialign attempts to calculate all-versus-all mappings when invoked with the `-X` flag. Each file in the arguments is converted to an index one by one and all the files are mapped on the index. A sam header is created on every index construction thus the resulting sam stream must be splitted at each header with the `samsplit` utility (compiled along with the minialign).

```
$ minialign -X read.fa > out.sam		# all-versus-all in a file
$ minialign -X read1.fa read2.fa ... readN.fa | samsplit out	# generates out.0000.sam, out.0001.sam, ..., out.xxxx.sam
```

When the `-X` flag is combined with the `-l index_filename` flag, minialign behaves similarly as the read-to-reference mapping mode with a prebuilt index, except that the mapping quality calculation formula is switched to the all-versus-all model. This setting makes it easy to distribute all-versus-all alignment tasks among computer clusters with prebuilt indices loaded from the shared filesystem.

```
$ minialign -X -l index.mai read1.fa read2.fa ... readN.fa > out.sam	# map read[1..N].fa onto index.mai, generating single sam file
```

## Notes, issues and limitations

* k-mer length (`k`) and minimizer window size (`w`) cannot be changed when the index is loaded from file. If you frequently adjust the two parameters, please prepare indices for each value or use the on-the-fly index construction mode.
* Large gap open penalty (> 5) and large X-drop penalty (> 64) are disallowed due to the limitation of the GABA library.
* SDUST masking is removed from the original minimap implementation.
* Repetitive seed-hit region detection is also removed.
* Index file format is incompatible among releases (nor with minimap). Please rebuild index files when a new version of minialign is installed. (otherwise result in SEGV)

## Updates

* 2016/4/21 (0.5.1) Add falcon\_sense input format (`-Ofalcon`), max #alignments (`-M`), sequence length filter (`-L`, `-H`), and name-as-id option (`-N`). Fix a bug in fasta/q parser (comment is leaked at the head of sequence).
* 2016/4/3 (0.5.0) Add MD (mismatch position) and SA (supplementary alignment) tag options for SAM format. Use CRC32 instruction for 64-bit integer hash. Faster FASTA and FASTQ parsing and SAM formatting. Fix bugs in multithreading.
* 2016/2/21 (0.4.6) Fix bugs in the indexing routines and the hashmap. Remove `samsplit`. Index file format is modified (now compressed by deflate).
* 2016/2/9 (0.4.5) Add support for BLAST6 / BLASR1 / BLASR4 / PAF formats. Change the default output format to PAF in the all-versus-all mode. Add support for NH, IH, XS, and NM tags in the sam format. Replaced internal implementations (hashmap and queue) to eliminate overheads.
* 2016/1/25 (0.4.4) Add all-versus-all alignment mode (enabled by `-X -xava` flags), change -xpacbio scoring params to -a1 -b2 -p2 -q1 (performed better on recent PacBio reads).
* 2016/1/14 (0.4.3) Add bam parser, quality string output, AS tag output, and RG line modification option. Default parameters are also modified to collect shorter  alignments.
* 2016/12/6 (0.4.2) Add splitted alignment rescuing algorithm.
* 2016/12/1 (0.4.1) Fix bug in sam output (broken CIGAR with both reverse-complemented and secondary flags).
* 2016/11/27 (0.4.0) Added mapping quality output, fix bug in chaining, and change output threshold measure from length to score (note: `-s` flag is changed to minimum score, `-r` is interpreted as score ratio).
* 2016/11/24 (0.3.3) Fix bugs in index load / dump functions.
* 2016/11/24 (0.3.2) Fix bugs in the chaining routine, make minimum score threshold option deprecated.
* 2016/11/1 (0.3.1) Changed minimum path length threshold option from '-M' to '-s'.
* 2016/11/1 (0.3.0) First release of 0.3 series, with a better chaining algorithm.
* 2016/10/5 (0.2.1) Last tagged commit of the version 0.2 series.
* 2016/9/13 (0.2.0) First tagged commit (unstable).

## Gallery

#### *Fast and Accurate* logo

![metcha hayaiyo](https://github.com/ocxtal/minialign/blob/master/pic/hayai.png)

#### *Kakizome*, Happy New Year 2017

![kakizome](https://github.com/ocxtal/minialign/blob/master/pic/kakizome.png)

#### Intel nuc, my main development machine

![he is also powerful](https://github.com/ocxtal/minialign/blob/master/pic/nuc.png)

## Copyright and license

The original source codes of the minimap program were developed by Heng Li and licensed under MIT, modified by Hajime Suzuki. The other codes, libgaba and ptask, were added by Hajime Suzuki. The whole repository except for the pictures in the gallery section (contents of pic directory) is licensed under MIT, following that of the original repository.
