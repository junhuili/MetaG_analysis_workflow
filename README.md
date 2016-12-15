# DESMAN

### Install DESMAN
```
module load gsl python-anaconda2
cd
git clone https://github.com/chrisquince/DESMAN.git
cd DESMAN
CFLAGS="-I$GSL_INCLUDE -L$GSL_LIB" python setup.py install --user
```
It is now installed and can be called like:
```
~/.local/bin/desman -h
```
Or add it to your path:
```
export PATH=~/.local/bin:$PATH
```
### Other software
Look [here] (https://github.com/rprops/DESMAN/wiki/Software-installations) to find out if you need to install anything else for the analysis. Make sure all these modules are loaded.

### Step 1: Quality trimming of reads

Make sure that you have non-interleaved fastq.gz files of forward and reverse reads. These should have an *R1* tag in their filename and saved in a directory called *sample*, e.g.: sample/sample.R1.fastq.gz.

**IMPORTANT** The adapter trimming in qc.sh is standard set for Truseq paired-end libraries. You will have to adjust this if this is different for you by changing the path in the shell script to the correct fasta file of the adapters located at /home/your_username/Trimmomatic-0.36/adapters.

**IMPORTANT** In case you are unsure which adapters are present in the sequences, you can download bbtools
(https://sourceforge.net/projects/bbmap/) unzip the tar.gz and add the directory to your export path. Then run the following code on a subsample of a sample (e.g., 1m reads). The resulting consensus sequences of the adapters will be stored in <code>adapters.fa</code>. 
```
bbmerge.sh in1=$(echo *R1.fastq) in2=$(echo *R2.fastq) outa=adapters.fa reads=1m
```
Run quality trimming:
```
bash /nfs/vdenef-lab/Shared/Ruben/scripts_metaG/wrappers/Assembly/qc.sh sample_directory
```
Alternatively modify the <code>run_quality.sh</code> or <code>run_quality.pbs</code> scripts to run sequentially.

Copy Fastqc files to new folder (adjust the paths)
```
rsync -a --include '*/' --include '*fastqc.html' --exclude '*' /scratch/vdenef_fluxm/rprops/DESMAN/metaG/Nextera /scratch/vdenef_fluxm/rprops/DESMAN/metaG/FASTQC --progress
```
#### Random subsampling of reads. Works if you are interested in abundant taxa.

You can check the number of reads in the interleaved fasta file with the <code>sample_size.sh</code>. This will store the sample sizes in the <code>sample_sizes.txt</code> file. Run this in the directory where your samples are located. Then use seqtk to randomly subsample your fasta files to the desired sample size. <code>-s</code> sets seed for random sampling.
```
bash sample_size.sh
seqtk sample -s 777 *.fastq 5000000 > *.fastq
```
### Step 2: Start co-assembly
At this point you have multiple assemblers to choose from (IDBA-UD/Megahit/...). We choose here for IDBA-UD.

#### IDBA-UD assembly
Merge all interleaved files to one file with the following shell script:
```
bash assembly_prep.sh
```
**Optional:** normalize data based on coverage (BBNorm)
This can be required for co-assemblies which are too big (see: [here] (http://jgi.doe.gov/data-and-tools/bbtools/bb-tools-user-guide/bbnorm-guide/)). First estimate memory requirements based on number of unique kmers using <code>loglog.sh</code>. Run this in a pbs script (due to memory requirements).
```
loglog.sh in=merged_dt_int.fasta
bbnorm.sh in=merged_dt_int.fasta out=merged_dt_int_normalized.100.5.fasta target=100 min=5
```
Due to the normalization some paired reads will have lost their mate. Therefore we must split the normalized fasta file and recombine them to a single fasta file (without the unpaired sequences). Run the below code in a pbs script. Adjust the pattern <code>:1</code> or <code>:2</code> to reflect the R1/R2 read names in your sequence data. Also adjust 
```
# Generate list of R1 and R2 reads
grep " 1:" merged_dt_int_normalized.100.5.fasta | sed "s/>//g" > list.R1
grep " 2:" merged_dt_int_normalized.100.5.fasta | sed "s/>//g" > list.R2
# Extract R1 and R2 reads from interleaved file using the filterbyname.sh script from BBmap
filterbyname.sh in=merged_dt_int_normalized.100.5.fasta names=list.R1 out=merged_dt_int_normalized.100.5.R1.fasta -include t
filterbyname.sh in=merged_dt_int_normalized.100.5.fasta names=list.R2 out=merged_dt_int_normalized.100.5.R2.fasta -include t
# Interleave R1/R2 reads using inhouse perl script (author: Sunit Jain)
/nfs/vdenef-lab/Shared/Ruben/scripts_metaG/SeqTools/interleave.pl -fwd merged_dt_int_normalized.100.5.R1.fasta -rev merged_dt_int_normalized.100.5.R2.fasta -o remerged_dt_int_normalized.100.5
```

Now run idba_ud (you may have to adjust the parameters.
```
idba_ud -o idba_k52_100_s8 -r remerged_dt_int_normalized.100.5.fasta --num_threads ${PBS_NP} --mink 52 --maxk 100 --step 8
```
In case you run this on flux you'll have to add the following line of code above your actual code to allow openmpi to run multithreaded.
```
# On Flux, to use OpenMP, you need to explicitly tell OpenMP how many threads to use.
export OMP_NUM_THREADS=${PBS_NP}
```

#### Ray assembly
Make sure <code>gcc</code> and <code>openmpi</code> modules are loaded.

#### Megahit assembly
Megahit is optimized for metagenomic assemblies, uses low memory and is insensitive to coverage-based normalization. So you can run this on quality trimmed read files. Read files should not be interleaved. When using the <code>qc.sh</code> for quality trimming you will have to remove single reads by using the <code>fastqCombinePairedEnd.py</code> script found [here] (https://github.com/enormandeau/Scripts/blob/master/fastqCombinePairedEnd.py). Example usage:
```
python /nfs/vdenef-lab/Shared/Ruben/scripts_metaG/SeqTools/fastqCombinePairedEnd.py *rev* *fwd*
```
Put all your paired end fastq files from all samples in one folder. Create R1.csv and R2.csv files for Megahit:
```
ls *R1.fastq | head -c -1 | tr '\n' ',' > R1.csv
ls *R2.fastq | head -c -1 | tr '\n' ',' > R2.csv
```
Assemble the reads (do this on flux).
Note to self: required 1.5 TB of RAM for my samples...
```
megahit -1 $(<R1.csv) -2 $(<R2.csv) -t 40 -o Assembly --presets meta-sensitive > megahit.out
```
### Step 3: Generating assembly stats
We use quast for this purpose (adjust path for your specific assembly):
```
quast.py -f --meta -t 20 -l "Contigs"  megahit_assembly_sensitive/final.contigs.fa
```
For IDBA-UD this can be run both on the scaffolds and the contigs. In general, the contigs are more reliable.

### Step 4A: CONCOCT binning
CONCOCT binning is not optimal for large co-assemblies since it combines both coverage and kmer/GC% in one step. This will result in very slow clustering for complex data sets. So go for Binsanity, if this is the case.

#### Megahit
Now we will bin the contigs using CONCOCT.
First cut your large contigs before running CONCOCT (make sure CONCOCT is in your path).
```
module load python-anaconda2/201607
module load gsl

mkdir contigs
cut_up_fasta.py -c 10000 -o 0 -m megahit_assembly_sensitive/final.contigs.fa > contigs/final_contigs_c10K.fa
```
Then map reads back onto the cut contigs (**warning:** fastq files must not contain unpaired reads). First make index:
```
cd contigs
bwa index final_contigs_c10K.fa
cd -
```
Then perform the mapping (will take a while: put this in shell script and submit as job):
```
for file in *R1.fastq
do

   stub=${file%_R1.fastq}

   echo $stub

   file2=${stub}_R2.fastq

   bwa mem -t 20 contigs/final_contigs_c10K.fa $file $file2 > Map/${stub}.sam
done
~/DESMAN/scripts/Collate.pl Map | tr "," "\t" > Coverage.tsv
```
#### IDBA-UD
IDBA-UD will also make scaffolds based on the contigs. So the two primary output files are:
```
scaffold.fa
contig.fa
```
First cut your large contigs before running CONCOCT (make sure CONCOCT is in your path).
```
module load python-anaconda2/201607
module load gsl

mkdir contigs
cut_up_fasta.py -c 10000 -o 0 -m idba_k52_100_s8/contig.fa > contigs/final_contigs_c10K.fa
```
Make an index for your contigs fasta.
```
cd contigs
bwa index final_contigs_c10K.fa
cd -
```
Then map reads (**warning:** fastq files must not contain unpaired reads).
```
for file in *R1.fastq
do

   stub=${file%_R1.fastq}

   echo $stub

   file2=${stub}_R2.fastq

   bwa mem -t 20 contigs/final_contigs_c10K.fa $file $file2 > Map/${stub}.sam
done
~/DESMAN/scripts/Collate.pl Map | tr "," "\t" > Coverage.tsv
```
Make sure you install bedtools2 for the next step.

Once you've formatted the coverage files we can start binning using CONCOCT (use 40 core in pbs script to maximize on C-CONCOCT):
```
module load python-anaconda2/201607 gsl
mkdir Concoct
cd Concoct
mv ../Coverage.tsv .
concoct --coverage_file Coverage.tsv --composition_file ../contigs/final_contigs_c10K.fa -s 777 --no_original_data
cd ..
```
### Step 4B: Binsanity binning
### Alternative binning with Binsanity (multistep - coverage/GC/kmer based)
Opted here for the <code>Binsanity-wf</code> function since it automates the refinment using <code>CheckM</code>.
**NOTICE**: be aware that your sample names in the BAM files should be split by anything other than ".".
Make sure that the bam files are sorted and indexed!!!:

*bedtools multicov depends upon index BAM files in order to count the number of overlaps in each BAM file. As such, each BAM file should be position sorted (samtool sort aln.bam aln.sort) and indexed (samtools index aln.sort.bam) with either samtools or bamtools.*

```
Binsanity-profile -o idba-assembly --contigs ../contigs/contigs.id.txt -i ../contigs/final_contigs_c10K.fa -s ../Map --transform Scale
Binsanity-wf -f ../contigs/-l final_contigs_c10K.fa -c idba-assembly.cov.x100.lognorm
```
