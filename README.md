# LeafCutter2

## Introduction

LeafCutter2 is a tool for clustering, functional characterization and quantification of splice junction read counts. It implements a novel dynamic programming algorithm over the standard LeafCutter output to classify splice junctions and alternative splicing events according to their molecular function (i.e.: protein-coding or unproductive).

#### Prerequisites

- The necessary python libraries can be installed with `leafcutter2_env.yml`.

## Clustering, classifying and quantifying splice junctions

#### Input:
- Splice junction BED files. They should contain at least six columns (chrom, start, end, name, score and strand), with the fifth column corresponding to the splice junction read counts. 
- GTF annotation with genes, start codons and stop codons.
- Genome assembly FASTA file. It must correspond to the same assembly as the GTF file.

We provide a basic example from GTEx in the `example/` directory. Assuming that we're working in the `example/` directory, a basic LeafCutter2 run work works as follows:

```
python ../scripts/leafcutter2.py \
    -j junction_files.txt \
    -r output_dir \
    -A annotation/chr10.gtf.gz \
    -G annotation/chr10.fa.gz
```

**Notes:** 
-    `-j junction_files.txt` should be a text file listing path to each junction file, one path per line.
-    `-r output_dir` specifies the directory of output (default is current directory, `./`). 
-    `-A annotation/chr10.gtf.gz` is a gtf file of chromosome 10 obtained from Gencode v43
-    `-G annotation/chr10.fa.gz` a FASTA file of chromosome 10 (GRCh38 assembly)

#### Output:
- `leafcutter2.cluster_ratios.gz`

. The first column of each row shows the genome coordinates of introns and labels them as **UP** (unproductive), **PR** (productive/protein-coding), **NE** (ambiguous in their functional effect) or **IN** (intergenic).

**Note:** 

We recommend using the `.junc` files from obtained from BAM files using [regtools junctions extract](https://regtools.readthedocs.io/en/latest/commands/junctions-extract/). They can also be obtained from [STAR's](https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf) `SJ.out.tab` files with minimal modifications. We include the .


- The splice junction BED files in this example were obtained [from GTEX's open access data](https://gtexportal.org/home/downloads/adult-gtex/bulk_tissue_expression). 
- Splice junction files should be BED formatted (0 based left close, right open).
- Splice junction files can be obtained from BAM files using [regtools junctions extract](https://regtools.readthedocs.io/en/latest/commands/junctions-extract/). E.g.: `regtools junctions extract -a 8 -i 50 -I 500000 bamfile.bam -o outfile.junc` . They can also be obtained from [STAR's](https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf) `SJ.out.tab` files with minimal modifications. We include the .
- LeafCutter2 will still run if you skip the `-A` and `-G` parameters, but it will not classify the splice junctions.


Main output files:
- `{out_prefix}_perind.counts.noise.gz`: output functional introns (intact), and  noisy introns. Note the start and end coordinates of noisy introns are merged to the min(starts) and max(ends) of all functional introns within cluster.
- `{out_prefix}_perind_numers.counts.noise.gz`: same as above, except write numerators.
- `{out_prefix}_perind.counts.noise_by_intron.gz`: same as the first output, except here noisy introns' coordinates are kept as their original coordinates. This is useful for diagnosis purposes. 


#### Parameters

```
python scripts/leafcutter2.py -h

usage: leafcutter2.py [-h] -j JUNCFILES 
                              [-o OUTPREFIX] [-q] [-r RUNDIR]
                              [-l MAXINTRONLEN] [-m MINCLUREADS]
                              [-M MINREADS] [-p MINCLURATIO]
                              [-c CLUSTER] [-k] [-C]
                              [-N NOISECLASS] [-f OFFSET] [-T]

required arguments:
  -j JUNCFILES, --juncfiles JUNCFILES
                        text file with all junction files to be processed
                        
optional arguments:
  -h, --help            show this help message and exit
  -r RUNDIR, --rundir RUNDIR
                        write to directory (default ./)
  -o OUTPREFIX, --outprefix OUTPREFIX
                        output prefix (default leafcutter2)
  -q, --quiet           don't print status messages to stdout
  -A ANNOTATION, --annot ANNOTATION
                        GTF annotation file
  -G GENOME, --genome GENOME
                        Genome fasta file
  -N MAXJUNCS, --max_juncs MAXJUNCS
                        skip solveNMD function if gene contains more than N juncs. 
                        Juncs in skipped genes are assigned Coding=False. Default 10000
  -l MAXINTRONLEN, --maxintronlen MAXINTRONLEN
                        maximum intron length in bp (default 100,000bp)
  -m MINCLUREADS, --minclureads MINCLUREADS
                        minimum reads in a cluster (default 30 reads)
  -M MINREADS, --minreads MINREADS
                        minimum reads for a junction to be considered for
                        clustering(default 5 reads)
  -p MINCLURATIO, --mincluratio MINCLURATIO
                        minimum fraction of reads in a cluster that support a
                        junction (default 0.001)
  -D MINREADSTD, --minreadstd MINREADSTD
                        minimum standard deviation of reads across samples for a \
                        junction to be included in output (default 0.5)
  -p MINCLURATIO, --mincluratio MINCLURATIO
                        minimum fraction of reads in a cluster that support a junction \
                        (default 0.001)
  -c CLUSTER, --cluster CLUSTER
                        refined cluster file when clusters are already made
  -k, --checkchrom      check that the chromosomes are well formated e.g. chr1,
                        chr2, ..., or 1, 2, ...
  -C, --includeconst    also include constitutive introns
  -f OFFSET, --offset OFFSET
                        Offset sometimes useful for off by 1 annotations. (default 0)
  -T, --keeptemp        keep temporary files. (default false)
  -L, --keepleafcutter1 keep temporary LeafCutter1 files. Useful for running differential splicing 
                        analysis with leafcutter's R package. (default false)
  -P, --keepannot       keep pickle files with parsed GTF annotation of transcripts. 
                        Useful for debugging. (default false)
```

## Pre-clustering splice junctions

Generating intron clusters first can save time form multiple subsequent runs. The script `scripts/leafcutter_make_clusters.py`
makes intron clusters separately that can be later used as input for `scripts/leafcutter2.py`. You can generate clusters by running: 

```
python scripts/leafcutter_make_clusters.py \
    -j junction_files.txt \
    -r output_dir \
    -o leafcutter2 
```
This will generate a file named `output_dir/leafcutter2_refined_noisy` that can be later used as an input for LeafCutter2. This will skip the clustering step:

```
python scripts/leafcutter2.py \
    -j junction_files.txt \
    -r output_dir \
    -o leafcutter2 \
    -A gtf_file.gtf \
    -G genome.fa \
    -c output_dir/leafcutter2_refined_noisy
```

**Note:** The junction-filtering options (--MINCLUREADS, --MINREADS, --MINCLURATIO) will be ignored by `leafcutter2.py` if a pre-defined set of intron clusters is provided.

