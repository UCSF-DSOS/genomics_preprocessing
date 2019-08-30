<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [First thoughts](#first-thoughts)
- [FastQC to assess read quality](#fastqc-to-assess-read-quality)
- [Adaptor trimming](#adaptor-trimming)
- [Trim reads to phred Q10](#trim-reads-to-phred-q10)
  - [Set up the files I need for the cleaning for loop. You can skip to the next subsection.](#set-up-the-files-i-need-for-the-cleaning-for-loop-you-can-skip-to-the-next-subsection)
  - [Skip to me: Running cleanup command](#skip-to-me-running-cleanup-command)
- [Generate genome for alignment](#generate-genome-for-alignment)
- [Align test fastq pairs](#align-test-fastq-pairs)
- [Assign reads to genes.](#assign-reads-to-genes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# First thoughts

- https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4728800/ Roughly the steps I will follow; however I add some QC steps they don't really mention
- Much of my knowledges comes from having applied these steps to scRNA seq. there is no dealing with UMIs/specific cells here, for which I am greatful (:

# FastQC to assess read quality

The following lines of code download the popular FastQC program, prep it for running, and run it on all the sequencing files.
```bash
$ wget https://www.bioinformatics.babraham.ac.uk/projects/fastqc/fastqc_v0.11.8.zip
$ unzip fastqc_v0.11.8.zip
$ chmod +x fastqc
$ ./FastQC/fastqc /mnt/DATA/LairdLab/rebecca/alana/fastq_AMH_SV_02/* -o /mnt/DATA/LairdLab/rebecca/alana/fastqc_quality_check/
```

The following samples showed poor quality reads:
- 454_hip_S198_R1_001_fastqc.html
- 454_hip_S198_R2_001_fastqc.html

Also every sample was found to have Nextera Transposase Sequence Adapter contents, as expected. Values ranged from ~10-20%. So I will trim those next.

# Adaptor trimming

References:
- http://seqanswers.com/forums/showthread.php?t=45814
- http://sites.psu.edu/biomonika/2015/07/03/trim-now-or-never-pe-and-mp-adaptor-removal/
- https://jgi.doe.gov/data-and-tools/bbtools/bb-tools-user-guide/bbduk-guide/

https://jgi.doe.gov/data-and-tools/bbtools/bb-tools-user-guide/bbduk-guide/

BBDuk can be downloaded here: https://sourceforge.net/projects/bbmap/

`cd` to directory where BBMap_##.##.tar.gz was downloaded.
```bash
$ tar xvzf BBMap_38.43.tar.gz
# example command: bbduk.sh in=reads.fq out=clean.fq ref=adapters.fa ktrim=r k=23 mink=11 hdist=1 tpe tbo
```

I set up a file containing the in1, in2, out1, out2 fastq.gz terms: `fastq_for_loop.txt`

```bash
cd /mnt/DATA/LairdLab/rebecca/alana/fastq_AMH_SW_02
cat ../fastq_loop_list.txt | while read LINE; do /home/rebecca/bbmap/bbduk.sh $LINE ref=/home/rebecca/bbmap/resources/adapters.fa ktrim=r k=23 mink=11 hdist=1 tpe tbo; done
```

Left this running overnight, although bbduk is pretty fast there are 40 pairs of fastqs to process. (:
Had family in town the week of 3/25, so took the week off. Picked this back up 4/1, no foolin.

454_R1_trim_fastqc.html and 454_R2_trim_fastqc.html are still the only read with alarming quality scores. Fortunately, bbduk stripped out the adaptor content like a pro! Now I'll go through and quality trim the reads. I'll let bbduk handle sample 454 and see what gets left behind after quality trimming.

# Trim reads to phred Q10

## Set up the files I need for the cleaning for loop. You can skip to the next subsection.
Duplicated file `fastq_loop_list.txt` to `ins_clean_loop.txt`. In `ins_clean_loop.txt`, I manually removed the `in1=* in2=* ` front of every line, resulting in a file that looked like this:

```bash
$ cp fastq_loop_list.txt ins_clean_loop.txt
$ emacs ins_clean_loop.txt
$ cat ins_clean_loop.txt
out1=3205_R1_trim.fastq.gz out2=3205_R2_trim.fastq.gz
[...redacted for length...]
out1=5046_R1_trim.fastq.gz out2=5046_R2_trim.fastq.gz
```

Next, I copied that file to a file called `outs_clean_loop.txt`. Back in `ins_clean_loop.txt`, I did a manual find and replace: all `out` instances were replaced with `in`. `ins_clean_loop.txt` now reads the following:

```bash
$ cp ins_clean_loop.txt outs_clean_loop.txt
$ emacs ins_clean_loop.txt
$ cat ins_clean_loop.txt
in1=3205_R1_trim.fastq.gz in2=3205_R2_trim.fastq.gz
[...redacted for length...]
in1=5046_R1_trim.fastq.gz in2=5046_R2_trim.fastq.gz
```

This gives us our input lines for the cleaning for loop I will write.

In `outs_clean_loop.txt`, I did a manual find and replace for `trim`, modifying every instance of `trim` to become `clean`:

```bash
$ emacs outs_clean_loop.txt
$ cat outs_clean_loop.txt
out1=3205_R1_clean.fastq.gz out2=3205_R2_clean.fastq.gz
[...redacted for length...]
out1=5046_R1_clean.fastq.gz out2=5046_R2_clean.fastq.gz
```

To combine the new `ins` and `outs` files, I used the paste command, and then manually deleted the extra spaces in between the ins and outs column, resulting in the following file. I think actually I could have run the loop without removing the spaces, but the removal results in a prettier file and we are all about appearences.

```bash
$ paste ins_clean_loop.txt outs_clean_loop.txt > clean_fastq_loop.txt
$ emacs clean_fastq_loop.txt
$ cat clean_fastq_loop.txt
in1=3205_R1_trim.fastq.gz in2=3205_R2_trim.fastq.gz out1=3205_R1_clean.fastq.gz out2=3205_R2_clean.fastq.gz
[...redacted for length...]
in1=5046_R1_trim.fastq.gz in2=5046_R2_trim.fastq.gz out1=5046_R1_clean.fastq.gz out2=5046_R2_clean.fastq.gz
```

Is this the fastest/best way to do this? LOL NO I'm the fakest coder ever. Continuing. Also I just remembered that I used a `while` loop before, not a `for` loop.

## Skip to me: Running cleanup command
Next, using BBTools `bbduk`, I quality-trim extracted reads to Q10 using Phred algorithm. Some notes:
- Pairs are always kept together – either both reads are kept, or both are discarded
- `qtrim=rl` tells the command to quality trim from both the left & right side of the read
- `trimq=10` will quality-trim to Q10 using the Phred algorithm, which is more accurate than naive trimming. I quality trim conservatively, rather than aggressively, per instructions like this: https://www.basepairtech.com/blog/trimming-for-rna-seq-data/
- you can really get into the weeds here: https://jgi.doe.gov/data-and-tools/bbtools/bb-tools-user-guide/bbduk-guide/

```bash
$ cd /mnt/DATA/LairdLab/rebecca/alana/fastq_AMH_SV_02
$ cat ../clean_fastq_loop.txt | while read LINE; do echo $LINE; done # uses the echo command to tests that loop reports lines of the file correctly; it does
$ cat ../clean_fastq_loop.txt | while read LINE; do /home/rebecca/bbmap/bbduk.sh $LINE qtrim=rl trimq=10; done
```

Left this running overnight.

I wrote the log that printed to the console to `bbduk_phred10.log`. Then I wrote this abomination of a one-liner to clean that file up a bit. If you are interested in seeing percents of the fastq that were trimmed due to quality, you can look at `bbduk_phred10.log2`
```bash
$ cat bbduk_phred10.log | awk '/^$/;/^j/;/^Input:/;/^Q/;/^To/;/^Res/' | awk 'NF > 0' | sed '/99...%)/{G;}' | sed '1~6s/^.\{74\}//g' > bbduk_phred10.log2
# the first awk keeps all lines that start with any of the things right after the "^"
# the second awk removes all blank lines
# the first sed adds back a blank line to seperate the fastq pairs into discrete text chunks
# the last sed removes from every 6th line starting with the 1st, the first 74 characters. this puts the ins/outs at the front of the line
```

# Generate genome for alignment

Download ensemble release 95 of mus_musculus data.
1. GTF - annotated genes
2. .fa files - literally the genome

```bash
$ wget ftp://ftp.ensembl.org/pub/release-95/gtf/mus_musculus/Mus_musculus.GRCm38.95.gtf.gz
$ wget ftp://ftp.ensembl.org/pub/release-95/fasta/mus_musculus/dna/Mus_musculus.GRCm38.dna_sm.primary_assembly.fa.gz
$ gunzip *.gz
$  STAR --runThreadN 24 --runMode genomeGenerate --genomeDir mm10_ensembl_GRCm38_95_sjdbO99/ --genomeFastaFiles Mus_musculus.GRCm38.dna_sm.primary_assembly.fa --sjdbGTFfile Mus_musculus.GRCm38.95.gtf --sjdbOverhang 99
# Finishes in just a few minutes because my server is a boss.
```

# Align test fastq pairs

```bash
STAR --runThreadN 10 --genomeDir /mnt/DATA/LairdLab/rebecca/alana/mm10/mm10_ensembl_GRCm38_95_sjdbO99 --readFilesIn /mnt/DATA/LairdLab/rebecca/alana/fastq_AMH_SV_02/3205_R1_clean.fastq.gz /mnt/DATA/LairdLab/rebecca/alana/fastq_AMH_SV_02/3205_R2_clean.fastq.gz --readFilesCommand zcat --outFilterMultimapNmax 1 --outSAMtype BAM SortedByCoordinate
```

Setting up loop:
```bash
$ for i in *_R1_clean.fastq.gz; do echo $i ${i%_R1_clean.fastq.gz}_R2_clean.fastq.gz; done
$ for i in *_R1_clean.fastq.gz; do echo ${i%_R1_clean.fastq.gz}; done
```
The first command tests a regular expressions loop that gets all the fastq pairs. The second command tests a regular expressions loop that creates unique file names for each fastq pair; it will append the number of the sample to all the files output by star. They are working well and require you to be in the directory where all the fastq files are.

```bash
$ for i in *_R1_clean.fastq.gz; do STAR --runThreadN 10 --runMode alignReads --genomeLoad LoadAndKeep --readFilesCommand zcat --outSAMtype BAM Unsorted --genomeDir /mnt/DATA/LairdLab/rebecca/alana/mm10/mm10_ensembl_GRCm38_95_sjdbO99 --readFilesIn $i ${i%_R1_clean.fastq.gz}_R2_clean.fastq.gz --outFileNamePrefix ${i%_R1_clean.fastq.gz}; done
$ for i in *_R1_clean.fastq.gz; do echo ${i%_R1_clean.fastq.gz}; done > tempnumber.txt
$ cat *final* | awk 'NR % 34 == 10' | awk '{ print $6 }' > temppercent.txt
$ paste tempnumber.txt temppercent.txt > ../percent_mapped_reads.txt
$ rm tempnumber.txt temppercent.txt
```

If you are curious about what percent of the fastqs were successfully mapped to the genome, I extracted the read pair names and % uniquely mapped to file `percent_mapped_reads.txt`. Also now including graph!
![mapped_reads](percentReadMappedToGenome.png)

# Assign reads to genes.

Right now the reads are aligned to the genome. Any reads that `STAR` couldn't line up with the mouse genome are 'unmapped' and won't be used going forward. Typical % of successfully aligned reads trend in the 60-80% range. For our fastq pairs, I extracted the read pair name and the % successfully mapped into the `percent_mapped_reads.txt` file. This is a good thing to graph to show PIs progress lol.

Before assigning reads to genes, I like to exclude reads from the sequencer that were not in their proper pair.

`samtools` command explainations bc I always forget:
- `-h` keeps the header
- `-f` keeps the flag
- `-F` rejects the flag
- `-b` outputs as .bam


```bash
$ samtools view 3205Aligned.out.bam | cut -f 2 | sort | uniq -c
      18 137 #mate unmapped (0x8)
10156380 147 #read mapped in proper pair (0x2)
      17 153 #mate unmapped (0x8)
10140368 163 #read mapped in proper pair (0x2)
       8 329 #mate unmapped (0x8); not primary alignment (0x100)
 1171448 339 #read mapped in proper pair (0x2); not primary alignment (0x100)
       8 345 #mate unmapped (0x8); not primary alignment (0x100)
 1166603 355 #read mapped in proper pair (0x2); not primary alignment (0x100)
       4 393 #mate unmapped (0x8); not primary alignment (0x100)
 1166603 403 #read mapped in proper pair (0x2); not primary alignment (0x100)
       8 409 #mate unmapped (0x8); not primary alignment (0x100)
 1171448 419 #read mapped in proper pair (0x2); not primary alignment (0x100)
      70 73 #mate unmapped (0x8)
10140368 83 #read mapped in proper pair (0x2)
      57 89 #mate unmapped (0x8)
10156380 99 #read mapped in proper pair (0x2)
$ for i in *Aligned.out.bam; do echo $i ${i%Aligned.out.bam}aligned.paired.bam; done
$ for i in *Aligned.out.bam; do samtools view -h -f 0x2 -b $i > ${i%Aligned.out.bam}aligned.paired.bam; done
```

Now properly paired reads are kept:
```bash
$ samtools view 3205aligned.paired.bam | cut -f 2 | sort | uniq -c
10156380 147
10140368 163
1171448 339
1166603 355
1166603 403
1171448 419
10140368 83
10156380 99
```

I also want to exclude paired reads with non-primary alignment, because these ambiguious reads should be excluded. I'll do that in the next step.

For the successfully aligned reads, we now have to assign them to specific genes. Just because they are aligned to the genome doesn't automatically mean they will be informative - sometimes they land in a regulatory region or something. (Oof just made a bunch of intronic dna people angry). Assigning to genes allows us to export a count matrix... the holy grail of all ^ this nonsense.

I'll use the `featureCounts` package, which is a part of the Subread software suite to do this.

Here is a loop to generate counts files. `featureCounts` automatically only takes reads which are in a primary alignment, so i'm not doing a seperate filter to remove the 0x100 flag reads.

```bash
$ for i in *aligned.paired.bam; do echo ${i%aligned.paired.bam}counts.txt $i; done
$ for i in *aligned.paired.bam; do featureCounts -T 10 -p -t exon -g gene_name -a /mnt/DATA/LairdLab/rebecca/alana/mm10/Mus_musculus.GRCm38.95.gtf -o ${i%aligned.paired.bam}counts.txt $i; done
```

Copied and pasted the terminal output of the loop into the file `fcScreenOuts.txt`. Distilled info for graphing with the following commands:

```bash
$ cat fcScreenOuts.txt | sed -n '44~50p' | awk {'print $7'} | sed 's/(//g;s/%)//g' > percentFC.txt
$ cat fcScreenOuts.txt | sed -n '14~50p' | awk {'print $3'} | sed 's/aligned.paired.bam//g' > fcnames.txt
$ paste fcnames.txt percentFC.txt > percentGenes.txt
$ rm percentFC.txt fcnames.txt
```

![assigned_to_genes](genePercentsAssignedToReads.png)
This % is of the properly mapped to genome.

We now have counts files! Hurray!

To edit count files for reading into R, I need to remove the top line which is a record of the command, remove columns 2 through 6 which are extra info I don't want, and write a few file with just column 1 (gene) and 7 (counts).

First, I backup the count matrices folder. Then, I rename the count matrix files to reflect that they contain the 'full' information. Then, I remove that first bad line with `tail`, delete some columns with `awk`, and write new files as ####counts.txt.
```
$ cd alana/countsMatrices
$ cp -r . ../countsMatrices_bkup
$ ls
3205counts.txt  3227counts.txt  3240counts.txt  3286counts.txt  3298counts.txt  365counts.txt   4073counts.txt  4087counts.txt  438counts.txt  5017counts.txt
3210counts.txt  3228counts.txt  3243counts.txt  3291counts.txt  3305counts.txt  383counts.txt   4075counts.txt  4088counts.txt  449counts.txt  5032counts.txt
3213counts.txt  3231counts.txt  3246counts.txt  3295counts.txt  3306counts.txt  384counts.txt   4078counts.txt  4271counts.txt  453counts.txt  5036counts.txt
3216counts.txt  3238counts.txt  3283counts.txt  3297counts.txt  363counts.txt   4072counts.txt  4079counts.txt  4272counts.txt  454counts.txt  5046counts.txt
$ for file in *.txt; do mv $file `basename $file .txt`_full.txt; done
$ ls
3205counts_full.txt  3228counts_full.txt  3246counts_full.txt  3297counts_full.txt  365counts_full.txt   4075counts_full.txt  4271counts_full.txt  454counts_full.txt
3210counts_full.txt  3231counts_full.txt  3283counts_full.txt  3298counts_full.txt  383counts_full.txt   4078counts_full.txt  4272counts_full.txt  5017counts_full.txt
3213counts_full.txt  3238counts_full.txt  3286counts_full.txt  3305counts_full.txt  384counts_full.txt   4079counts_full.txt  438counts_full.txt   5032counts_full.txt
3216counts_full.txt  3240counts_full.txt  3291counts_full.txt  3306counts_full.txt  4072counts_full.txt  4087counts_full.txt  449counts_full.txt   5036counts_full.txt
3227counts_full.txt  3243counts_full.txt  3295counts_full.txt  363counts_full.txt   4073counts_full.txt  4088counts_full.txt  453counts_full.txt   5046counts_full.txt
$ for i in *counts_full.txt; do tail -n +2 $i | awk '{print $1,$7}' > ${i%counts_full.txt}counts.txt; done
```