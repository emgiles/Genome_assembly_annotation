# GENOME ASSEMBLY AND STRUCTURAL ANNOTATION

### DATA PREPRATION
Create working directory with sequence files
Unzip fastq.gz subread files
### GENOME ASSEMBLY
I have used Canu and wtdbg2. Both take several hours to run assembly with PacBio data and using 16 cores (genome size approx 500M).
Run assembly using fastq files using Canu

```canu  -p project_prefix -d project_directory genomeSize=500m -pacbio pacbio_file.fastq```

.contigs.fasta is final filtered assembly results for diploid genomes
.unitigs.fasta is file that includes all alternative paths
.unassembled.fasta is a file of contigs with poor support
.report gives statistics of the raw data and final assembly including N50s and length

Run assembly using wtdbg2

```nohup ./wtdbg2 -x sq -g 500m -t 16 -i ../folder/pacbio.fastq.gz -fo output_name```

Assembly with finish creating a .ctg.lay.gz file that is approx 1-20G. Then use next step:

```nohup ./wtpoa-cns -t 16 -i output_name.ctg.lay.gz -fo output_file_name.ctg.fa &```

###### Check assembly with QUAST

```conda activate quast_purgedups_minimap```

```nohup quast -r /{PWD}/reference_genome.fasta -s -e --large -k -f -b new_assembly.contigs.fasta & ```

Compare results to reference assembly
Output is report.pdf

###### Check assembly with BUSCO
This example uses the mollusca database from ncbi. Also try with metazoa db.

```conda activate busco```

```nohup busco -o VG2_busco_mollusca -i /{PWD}/new_assembly.contigs.fasta -l mollusca_odb10 -m genome & ```

### GENOME ASSEMBLY POST-PROCESSING
##### Remove duplications with purge_dups
See [PIPELINE INSTRUCTIONS in github](https://github.com/dfguan/purge_dups). 
Note, this software is difficult to install and run due to minimal documentation online.

```conda activate quast_purgedups_minimap```

##### Check purged assembly with QUAST

```conda activate quast_purgedups_minimap```

```/home/pablo/anaconda2/envs/emily/bin/quast -r /{PWD}/reference_genome.fasta -s -e --large -k new_assembly.contigs_purged_purged.fa```

compare to reference
output is report.pdf

##### Check purged assembly with BUSCO

```conda activate busco```

```nohup busco -o VG2_busco_mollusca -i /{PWD}/new_assembly.contigs_purged_purged.fa -l mollusca_odb10 -m genome &```

using mollusca db

##### Genome polishing with Polypolish
Index genome with bwa-mem2

```conda activate bwa-mem2```

```bwa-mem2 index -p species_assembly1 my_genome_assembly.fasta```

Align paired end short reads to genome individually. Here the -a parameter is very important. This retains all alignments.

```bwa-mem2 mem -t 30 -a species_assembly1 my-short-reads_1.fastq > alignment_to_genome_1.sam```

```bwa-mem2 mem -t 30 -a species_assembly1 my-short-reads_2.fastq > alignment_to_genome_2.sam```

Polish genome using alignments

```path/to/polypolish polish my_genome_assembly.fasta alignment_to_genome_1.sam alignment_to_genome_2.sam --debug polishing_summary_results.tsv --careful > my_genome_assembly_polished.fasta```

Now you can stop here and move on to scaffolding if all you want is to polish.

If you want to know the results of how well the polishing did...
Determine amount of bases to change in the genome assembly

```awk -F'\t' '{print $8}' polishing_summary_results.tsv | sort | uniq -c```

Let's take a look at just the changed bases. Parse results

```awk -F'\t' '$8 == "changed"' polishing_summary_results.tsv > polished-changed.tsv```

Create a bed file from your parsed polished results

```awk -F'\t' '{print $1, $2, $2}' OFS='\t' polished-changed.tsv > polished-changed.bed```

Retain first two columns of assembly index file

```cut -f1,2 my_genome_assembly.fasta.fai > my_genome_assembly.fasta_first2cols.fai```

If you already have an annotation file for your assembly (ie you have proceeded through all of the subsequent steps in this repository)
Sort the annotation file based on the genome index

```sortBed -i my_annotation_v1.gff3 -faidx my_genome_assembly.fasta.fai > my_annotation_v1_sorted.gff3```

Create an intergenic bed file

```complementBed -i my_annotation_v1_sorted.gff3 -g my_genome_assembly.fasta_first2cols.fai > my_annotation_v1_sorted_intergenic.bed```

Sort the polishing results bed file just like the gff was sorted

```bedtools sort -i polished-changed.bed -faidx my_genome_assembly.fasta_first2cols.fai > polished-changed-sorted.bed```

Determine which nucleotides changed by polishing are in genic vs intergenic regions

```bedtools intersect -a polished-changed-sorted.bed -b my_annotation_v1_sorted.gff3 my_annotation_v1_sorted_intergenic.bed -wao -f 0.9 -names genic intergenic > polished-changed_intergenic_genic.txt```

Retain only the gene regions

```awk '$4 == "genic"' polished-changed_intergenic_genic.txt > polished-changed_genic.txt```

Count nucleotides changed in genes

```awk '$7 == "gene" {count++} END {print count}' polished-changed_genic.txt```

Count nucleotides changed in exons

```awk '$7 == "exon" {count++} END {print count}' polished-changed_genic.txt```

##### Scaffold with RAGTAG. Decide whether or not reference based scaffolding is a good idea for your research aim. Note, introduces structural bias, might be better off not scaffolding if you don't have independent proximity data, such as HiC.

```conda activate ragtag_blobtools_bwa```

```nohup ragtag.py scaffold /{PWD}/reference_genome.fasta /{PWD}/new_assembly.contigs_purged_purged.fa &```

using closely related species as a reference

###### Checking purged scaffolded assembly with QUAST

```conda activate quast_purgedups_minimap```

```nohup quast -r /{PWD}/Sscurra_genome.fasta -e --large -k -f -b /{PWD}/new_assembly.contigs_purged_purged.ragtag.scaffold.fasta &```

###### Checking purged scaffolded assembly with BUSCO

```conda activate busco```

```nohup busco -o VG2_busco_mollusca -i /{PWD}/new_assembly.contigs_purged_purged.ragtag.scaffold.fasta -l mollusca_odb10 -m genome &``` 

```nohup busco -o VG2_busco_metazoa -i /{PWD}/new_assembly.contigs_purged_purged.ragtag.scaffold.fasta -l metazoa_odb10 -m genome &```

### TRANSCRIPTOME AND PROTEIN LIST RECOVERY
##### Upload transcriptomes and check with busco

```conda activate busco```

```nohup busco -o species_trinity_metazoa -i /{PWD}/Ensamble_transcriptoma_SViridula.fasta -l metazoa_odb10 -m transcriptome &```

Should have high busco completeness. Duplications can be moderately high.

##### Check quality of raw reads

```conda activate fastqc```

```Fastqc 30j_1.fq.gz```

If the raw reads are of low quality be cautious about using transcriptome.

##### Align transcriptome to genome
HiSat and BWA-MEM are not recommended for alignments of transciptomes to genomes. Transcriptomes have multiple transcripts for the same genomic region. These types of alignments are best done with GMAP which can be installed via conda. 
Example command:

```gmap_build -d dbname  -D dbloc dbfasta```

GMAP recomends creating objects for -d and -D and dbfasta. This did not work so I ran with full paths.

###### Build GMAP database

```conda activate gmap```

```gmap_build -d viridula_masked -D /{PWD}/VG2_canu_purged_purged_ragtag.scaffold.fasta.masked.nosemicolon  File was written to /{PWD}/viridula_masked.fasta```

###### Align transcripts with output to .sam
```gmap -D /{PWD}/gmap/ -d viridula_masked -B 5 -t 10 --input-buffer-size=1000000 --output-buffer-size=1000000 -f samse /{PWD}/Ensamble_s_viridula_31_y_143.fasta > viridula_masked.nosemicolon.transcriptome2.sam```
###### Use samtools (installed in another conda env) to convert from .sam to .bam
```conda activate samtools_blast```
```samtools view -S -b viridula_masked.nosemicolon.transcriptome2.sam > viridula_masked.nosemicolon.transcriptome2.bam```
###### Sort .bam
```samtools sort -t 10 -o viridula_masked.nosemicolon.transcriptome2_sorted.bam -T Gmap_temp viridula_masked.nosemicolon.transcriptome2.bam```
###### Check stats of alignment
```samtools flagstat viridula_masked.nosemicolon.transcriptome2_sorted.bam```
It is important that a high percentage of the transcripts map to the genome. In the case of viridula, 96.67% mapped.
##### Extract lists of proteins from Transcriptome
First, install TransDecoder and all dependencies.
###### Extract long open reading frames from transcriptome
```conda activate transdecoder```
```TransDecoder.LongOrfs -t Ensamble_s_viridula_31_y_143.fasta```
###### Predict the likely coding regions
```TransDecoder.Predict -t Ensamble_s_viridula_31_y_143.fasta```
Transdecoder finishes correctly if .bed .cds .gff3 and .pep files are created
###### Remove stop codons (*)
```conda activate sed```
```sed -e 's/\*//g' Ensamble_s_viridula_31_y_143.fasta.transdecoder.pep > Ensamble_s_viridula_31_y_143.fasta.transdecoder.faa```
###### Cluster and remove sequences that are 95% similar to an existing sequence
Install cd-hit via conda
```conda activate cd-hit```
```cd-hit -i Ensamble_s_viridula_31_y_143.fasta.transdecoder.faa -o Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95.faa -c 0.95```
###### remove special characters from headers
```conda activate sed```
```sed -e 's/:/_/g' Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95.faa | sed -e 's/=/_/g' | sed -e 's/:/_/g' >Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95_nospcharac.faa```
###### remove tildas from headers
```sed -e 's/~/_/g' Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95_nospcharac.faa > Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95_nospcharac2.faa```
### REPEAT MASKING
First install RepeatModeler and all associated software. Installation via conda didn't work, had to install manually. Installation takes a long time, many dependencies. See [this tutorial](https://darencard.net/blog/2022-10-13-install-repeat-modeler-masker/) for installation guide.

##### Build database for repeat modeling
```nohup ~/envs/repeatmodeler02/repeat-annotation/RepeatModeler-2.0.3/BuildDatabase -name viridula /{PWD}/VG2_canu_purged_purged_ragtag.scaffold.fasta```
This is quite fast, it is basically indexing the reference genome

##### Run RepeatModeler
```nohup ~/RepeatModeler-2.0.3/RepeatModeler -pa 8 -database viridula &```
This takes a long time... 72+hr to run
Important output files are viridula-families.fa and viridula-families.stk

##### Run RepeatClassifier
```nohup ~/RepeatModeler-2.0.3/RepeatClassifier -consensi viridula-families.fa -stockholm viridula-families.stk -engine ncbi &```
This takes 12-24hr.
Important output files are viridula-families-classified.stk and viridula-families.fa.classified

##### Run RepeatMasker
```nohup ~/RepeatMasker -pa 8 -lib /{PWD}/repeatmask04/viridula-families.fa.classified /{PWD}/VG2_canu_purged_purged_ragtag.scaffold.fasta -dir softmask_viridula -xsmall -poly &```
output files are placed in folder softmask_viridula. Most important output is the .tbl file that contains a summary of the masked elements and .fasta.masked file that is genome assembly softmasked.
This takes several hours to half a day.
copy softmasked assembly to working directory for postprocessing.

### PREPARE FOR ANNOTATION 
##### Remove all semicolons from sequence names
Install program sed previously using conda.
```conda activate sed```
```sed -e 's/;/_/g' VG2_canu_purged_purged_ragtag.scaffold.fasta.masked > VG2_canu_purged_purged_ragtag.scaffold.fasta.masked.nosemicolon```
very quick, no need for nohup
output file is file.semicolon

##### Acquire BRAKER via Singularity
BRAKER can also be installed manually, but installation of dependency AUGUSTUS is difficult. I decided to go for the Singularity container. Singularity must be installed previously.

Build BRAKER via Singularity in /home/usr
```singularity build braker3.sif docker://teambraker/braker3:latest```
Make BRAKER executable in /home/usr
```singularity exec braker3.sif braker.pl```
Install license key for GeneMark-ETP in home directory in /home/usr
```singularity exec braker3.sif print_braker3_setup.py```
Get test script 1 for BRAKER via Singularity in /home/usr
```singularity exec -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test1.sh .```
Get test script 2 for BRAKER via Singularity in /home/usr
```singularity exec -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test2.sh .```
Get test script 3 for BRAKER via Singularity in /home/usr
```singularity exec -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test3.sh .```
Export two bash environments to run test scripts
```export BRAKER_SIF=/home/pablo/braker3.sif```
Execute test script 1
```bash test1.sh```
Execute test script 2
```bash test2.sh```
Execute test script 3
```bash test3.sh```
### STRUCTURAL ANNOTATION i.e. PROTEIN ID
```singularity exec braker3.sif braker.pl --species=Scurria_viridula05 --genome=VG2_canu_purged_purged_ragtag.scaffold.fasta.masked.nosemicolon --prot_seq=Ensamble_s_viridula_31_y_143.fasta.transdecoder_CDHIT_95_nospcharac2.faa --softmasking --workingdir=braker_14apr2023 --threads=15```
BRAKER finishes correctly if braker.aa, braker.gtf, augustus.hints.aa, and augustus.gtf are created.
###### Count number of proteins recovered
```grep -c ">" augustus.hints.aa```
###### Run busco on augustus.hints.aa
```conda activate busco```
```nohup busco -o VG2_busco_metazoa -i /{PWD}/braker_14apr2023/Augustus/augustus.hints.aa -l metazoa_odb10 -m proteins &```
Busco results in this case were C:96.8%, S:90.4%, D:6.4%
Can also run busco metazoa to verify. 
### FUNCTIONAL ANNOTATION
In brief, this step involves identifying functions and GO terms for the list of proteins recovered from BRAKER.
This can be done with OmicsBox https://www.biobam.com/omicsbox/ or alternatively with B2Go standalone but apparently it is very slow without DiamondBlast.
