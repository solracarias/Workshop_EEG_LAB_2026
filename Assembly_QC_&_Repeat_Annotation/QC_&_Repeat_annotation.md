# Genome assembly QC and Genome annotation - SMSC

<!-- TOC depthFrom:2 -->

 * [Folder structure](#Folder-structure)
 * [Input file](#Input-file)
 * [Run BUSCO](#Run-BUSCO)
 * [Run Blobtools](#Run-Blobtools)
 * [Run Blobtools2](#Run-Blobtools2)
 * [Run Merqury](#Run-Merqury)
 * [Masking and annotating repetitive elements with Repeatmodeler and RepeatMasker](#Masking-and-annotating-repetitive-elements-with-Repeatmodeler-and-RepeatMasker)
 * [Running GeMoMa](#Running-GeMoMa)


<!-- /TOC -->

### Folder structure

Let's create our folder structure for this workshop. It's easier to troubleshoot any issues if we are all working within the same framework. First, we will create a folder called `genome_qc_annotation` in your `/scratch/genomics/your_username` folder. 

```
cd /scratch/genomics/your_username/
mkdir genome_qc_annotation
```

Next, we will change directories to the genome_annot folder and we will create a several folders that we will use today and tomorrow. Here's the list of folders:

- busco
- bloobtools
- repeat_annotation
- gemoma

<details><summary>SOLUTION</summary>
<p>

```
mkdir busco blobtools repeat_annotation gemoma

```
If you type the command `tree`, this is what you should see:

```
|__ busco  
|__ blobtools
|__ gemoma
|__ repeat_annotation
 ```

</p>
</details>


If we follow this folder structure, we will have all the results organized by software, which will facilitate finding everything later. 

**Submitting jobs**: With this folder structure, we will save all the job files with each program folder and the job files will be submitted from there.   

### Input file
For this session, we will use our new moth assembly assembly. You can find this file here: `/scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta `


**If you want to run things quickly you can run the programs by Extracting some scaffolds**
<details><summary>SOLUTION</summary>
<p>

To generate this file, we used `bioawk` and `samtools` to extract the sequences from the original assembly:

`ml bio/bioawk`
`ml bio/samtools`

Create a list with the 10 largest sequences. The number of sequences is determined by the number following the `head` command in the end. 

`cat assembly.fasta | bioawk -c fastx '{ print length($seq), $name }' | sort -k1,1rn | head -10 > 10largest.list`

Use `samtools` to extract the list of sequences from the original assembly:

`xargs samtools faidx assembly.fasta < 10largest.list > assembly_10largest.fasta` 

</p>
</details>

### Run BUSCO

BUSCO (Simão et al. 2015; Waterhouse et al. 2017) assesses completeness by searching the genome for a selected set of single copy orthologous genes. There are several databases that can be used with BUSCO and they can be downloaded from here: [https://buscos.ezlab.org](https://buscos.ezlab.org). 


#### Job file: busco_moth.job
- Queue: medium
- PE: multi-thread
- Number of CPUs: 10
- Memory: 6G (6G per CPU, 60G total)
- Module: `module load bio/busco/5.7.0`
- Commands:

```
busco -o Sol_moth -i /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta -l lepidoptera_odb12 -c $NSLOTS -m genome
```

##### Explanation:
```
-o: name of the output folder and files
-i: input file (FASTA)
-l: name of the database of BUSCOs (This will automatically connected and dowloand the database from the BUSCO website).
-c: number of CPUs
-m: mode (options are genome, transcriptome, proteins)

```

##### *** IMPORTANT ***

BUSCO doesn't have an option to redirect the output to a different folder. For that reason, we will submit the BUSCO job from the `busco` folder. Thus, make sure that your busco.job file is in your busco folder.

```
cd ../busco
qsub busco_moth.job
```

**Note about Databases:**

If you do not have internet connection on the node where running the software you can download the database and run the program offline. For instance to download the Mammalia database you can use the command `wget` and extract it.It is important to dowlod and untar the folder on your busco folder. Let's `cd` to the directory `busco` first.

	wget https://busco-data.ezlab.org/v5/data/lineages/lepidopetera_odb12.2026-XX-XX.tar.gz 
	tar -zxf lepidopetera_odb12.2026-XX-XX.tar.gz

In this case, the command to run busco will have to change to: 

```
busco  -o Sol_moth -i /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta -l lepidoptera_odb12 -c $NSLOTS -m genome --offline --download_path /path/to/datasets
```
To Plot the results use the generate_plot.py script provided by BUSCO.

```
generate_plot.py -wd /scratch/genomics/ariasc/smsc_2024/busco_run/Sol_moth
```

##### Explanation:

```
-wd: Path to the BUSCO results directory
```

This will generate a .png file of your BUSCO results, which visually summarizes the completeness of the dataset in terms of the number of complete, fragmented, or missing BUSCOs.


### Run Bloobtools

BlobTools is a command line tool designed for interactive quality assessment of genome assemblies and contaminant detection and filtering.


#### Preparing files for blobtools

First, you need to blast your assembly to know nt databases. For this we will use blastn. 


#### Job file: blast_Guam_Rail.job
- Queue: medium
- PE: multi-thread
- Number of CPUs: 10
- Memory: 6G (6G per CPU, 60G total)
- Module: `module load bio/blast/2.15.0`
- Commands:

```
blastn -db /scratch/dbs/blast/v5 -query /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta -outfmt "6 qseqid staxids bitscore std" -max_target_seqs 20 -max_hsps 1 -evalue 1e-20 -num_threads $NSLOTS -out Sol_moth_blast.out
```

##### Explanation:
```
-db: ncbi nucleotide database
-query: input file (FASTA)
-outfmt: format of the output file (important to for blobtools) 
-max_target_seqs: Number of aligned sequences to keep.
-max_hsps: Maximum number of HSPs (alignments) to keep for any single query-subject pair.
-num_threads: number of CPUs
-out: name of the output file
```

Second, you need to map raw reads to the genome assembly. We will use minimap2 for this. Minimap2 is a versatile sequence alignment program that aligns DNA or mRNA sequences against reference database. Typical use cases include: (1) mapping PacBio or Oxford Nanopore genomic reads to a reference genome; or (2) aligning Illumina single- or paired-end reads to a reference genome. After mapping the reads we need to convert the output file SAM into a BAM file and sort this file. For this we will use the program samtools. Samtools is a suite of programs for interacting with high-throughput sequencing data. 

#### Job file: minimap_Sol_moth.job
- Queue: medium
- PE: multi-thread
- Number of CPUs: 10
- Memory: 6G (6G per CPU, 60G total)
- Module: 
```
  module load bio/minimap2
  module load bio/samtools
```
- Commands:

```
minimap2 -ax map-hifi -t 20 /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta /scratch/genomics/ariasc/Sol_moth_genome/filtered_reads_38G.fastq.gz | samtools view -b | samtools sort -@20 -O BAM -o Sol_moth_sorted.bam - && samtools index Sol_moth_sorted.bam
```

##### Explanation:

```
minimap2
-ax: preset configuration to map hifi reads to genomes.
-t: number of threads to use.
Samtools
sort: sort command
-@: number of threads to use.
-O: output format.
-o: name of the outputformat
index: index bam file
```

#### Creating blobtools database

Now that we have the blast and mapping results we can create the BlobTools database. This can take a few minutes depending on how much coverage you have for your genome assembly.

- Queue: medium
- PE: multi-thread
- Number of CPUs: 1
- Memory: 6G (6G per CPU, 6G total)
- Module: `module load bio/blobtools`


- Commands:

```
blobtools create -i /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta -b Sol_Moth_sorted.bam -t /path/to_hits_output/Sol_moth_blast.out -o Sol_moth_my_first_blobplot

```

##### Explanation:

```
-i: genome assembly (fasta)
-b: mapped reads to genome assembly (bam)
-t: hits output file from a search algorith (i.e blastn). hit file is a TSV file which links sequence IDs in a assembly to NCBI TaxIDs, with a given score.
-o: path and/or name of the blobtools database.

```


#### blob and cov plots

Once you have a BlobDir database, we can plot the blobplot and the covplot. Since this is not computationally speaking intense, we can use an interactive node to run blobtools plot command.

```
qrsh
module load bio/blobtools
mkdir plots
blobtools plot -i Sol_moth_my_first_blobplot.blobDB.json -o plots/

```

This comand generates three files:

* Guam_Rail_my_first_blobplot.blobDB.json.bestsum.phylum.p7.span.100.blobplot.bam0.png
* Guam_Rail_my_first_blobplot.blobDB.json.phylum.p7.span.100.blobplot.read_cov.bam0.png
* Guam_Rail_my_first_blobplot.blobDB.json.bestsum.phylum.p7.span.100.blobplot.stats.txt

Please download these files to your machine. Remember that you can use the ffsend module.

* Is your genome assembly contaminated?
* If yes, What taxa are contaminant of your assembly?
* All your reads are mapping to your genome?


### Run Bloobtools --- Under construction 

I want to use nf-core and nextflow on this section. This will be here soon.


### Kmer-based assembly evaluation with Merqury and Meryl

First, you create a read kmer database from the HiFi, ONT or reads used for the base assembly using meryl.
```
qrsh -pe mthread 4
module load bioinformatics/merqury/1.3
sh $MERQURY/best_k.sh 120000000

meryl count k=21 count /scratch/genomics/ariasc/Sol_moth_genome/filtered_reads_38G.fastq.gz output Sol_moth.meryl

merqury.sh Sol_moth.meryl /scratch/genomics/ariasc/Sol_moth_genome/teuthros.asm.bp.p_ctg.fasta Sol_moth_primary
```

```
qrsh -pe mthread 4
module load bioinformatics/merqury/1.3
sh $MERQURY/best_k.sh 1200000000

meryl count k=21 count /scratch/genomics/ariasc/Sol_moth_genome/filtered_reads_38G.fastq.gz output Sol_moth.meryl
```

Second, you run merqury.sh using the following commands below:

```
merqury.sh Sol_moth.meryl Sol_moth.hap1.p_ctg.fasta Sol_moth.hap2.p_ctg.fasta Sol_moth_merqury
```

### Running a complete pipeline such as sanger/genome assembly 
We need to install:

* NextFlow
* Sigularity - already in Hydar
* nf-core tools
* Sanger-tol/genomeassembly pipeline

Wha is ...
- NextFlow?
A Nextflow pipeline is a workflow management system that allows developers and scientists to orchestrate complex computational tasks into a cohesive, scalable, and reproducible pipeline. It acts as a bridge between different programming scripts (like Bash, Python, or R) and compute infrastructures (like cloud services or local servers).
  
- nf-core?
nf-core is a global community collaborating to build open-source Nextflow components and pipelines. nf-core provides a standardised set of best practices, guidelines, and templates for building and sharing bioinformatics pipelines. These pipelines are designed to be modular, scalable, and portable, allowing researchers to easily adapt and execute them using their own data and compute resources.
+ [nf-core](https://nf-co.re/docs/get_started/nf-core) 

- Singularity ?

A Singularity container is a specialized, open-source container platform designed specifically for scientific computing and High-Performance Computing (HPC) clusters. It packages software, libraries, and dependencies into a single, portable file, making complex research applications easy to reproduce and run across different systems.

- sanger-tol/genomeassembly pipeline?
is a bioinformatics pipeline for de-novo genome assembly from long read data (PacBio HiFi or ONT), long-range Hi-C data, and optionally Illumina WGS and Illumina 10X linked reads. It is capable of producing primary/alternative assembles, Hi-C phased assemblies using Hi-C data, and trio-binned assemblies using data from parental sequencing.
+ [sanger-tol/genomeassembly pipeline](https://pipelines.tol.sanger.ac.uk/genomeassembly/dev) 

We will use conda to install Nexflow and nf-core tool. Lets follow steps in nf-core.




### Masking and annotating repetitive elements with Repeatmodeler and RepeatMasker ------ UNDERCONSTRUCTION

Repeatmodeler is a repeat-identifying software that can provide a list of repeat family sequences to mask repeats in a genome with RepeatMasker. Repeatmasker is a program that screens DNA sequences for interspersed repeats and low complexity DNA sequences. the output of the program is a detailed annotation of the repeats that are present in the query sequence as well as a modified version of the query sequence in which all the annotated repeats have been masked. 

Things to consider with Repeatmodeler software is that it can take a long time with large genomes (>1Gb==>96hrs on a 16 cpu node). You also need to set the correct parameters in repeatmodeler so that you get repeats that are not only grouped by family, but are also annotated.

Repeatmodeler http://www.repeatmasker.org/RepeatModeler/  
RepeatMasker http://www.repeatmasker.org/RMDownload.html

The first step to run Repeatmodeler is that you need to build a Database. The Build Database step is quick (several seconds at most). you can either use an interactive node or a job file.

#### Job file: repeatmodeler_database.job
- Queue: medium
- PE: multi-thread
- Number of CPUs: 1
- Memory: 10G
- Module: `module load bio/repeatmodeler`
- Commands:

```
BuildDatabase -name Guam_rail /path/to_assembly/bHypOws1_hifiasm.bp.p_ctg.fasta
# usage:BuildDatabase -name {database_name} {genome_file-in_fasta_format}
```
##### Explanation:
```
-name: name to be given to the database
genome file in fasta format
```

##### Output files:
- several files with a specific structure to be populate and used by repeatmodeler command.

The second step is two actually run RepeatModeler. Again this can take several days depending on the genome size.

#### Job file: repeatmodeler_Guam_Rail.job
- Queue: high
- PE: multi-thread
- Number of CPUs: 36
- Memory: 10G
- Module: `module load bio/repeatmodeler`
- Commands:

```
# Usage: RepeatModeler -database {database_name} -pa {number of cores} -LTRStruct > out.log
RepeatModeler -database Guam_rail -threads 36 -engine ncbi -LTRStruct  > repeatmodeler_GR_out.log

```

##### Explanation:
```
-database: The prefix name of a XDF formatted sequence database containing the genomic sequence to use when building repeat models.
-pa: number of cpus
-engine: The name of the search engine we are using. I.e abblast/wublast or ncbi (rmblast version).
#Note the new version of repeatmodeler calls another program called LTRStruct.
-LTRStruct: enables the optional LTR structural finder.
```

##### Output files:
- consensi.fa.classified: complete database to be used in RepeatMasker.

The last step to get a repeat annotation is to run ReapeatMasker. 


#### Job file: repeatmasker_Guam_Rail.job
- Queue: high
- PE: multi-thread
- Number of CPUs: 30
- Memory: 10G
- Module: `module load bio/repeatmodeler`
- Commands:

```
# usage: RepeatMasker -pa 30 -gff -lib {consensi_classified} -dir {dir_name} {genome_in_fasta}

RepeatMasker -pa $NSLOTS -xsmall -gff -lib consensi.fa.classified -dir ../repeatmasker /path/to_assembly/bHypOws1_hifiasm.bp.p_ctg.fasta
```
##### Explanation:
```
-lib: repeatmodeler repbase database to be search
-pa: number of cpus
-xsmall: softmasking (instead of hardmasking with N)
-dir ../repmasker: writes the output to the directory repmasker
-gff: output format of the annotated repeats
```

##### Output files:
- bHypOws1_hifiasm.bp.p_ctg.fasta.tbl: summary information about the repetitive elements
- bHypOws1_hifiasm.bp.p_ctg..masked.fasta: masked assembly (in our case, softmasked)
- bHypOws1_hifiasm.bp.p_ctg.fasta.out: detailed information about the repetitive elements, including coordinates, repeat type and size.


### Run GeMoMa

Gene Model Mapper (GeMoMa) is a homology-based gene prediction program. GeMoMa uses the annotation of protein-coding genes in a reference genome to infer the annotation of protein-coding genes in a target genome. Thus, GeMoMa uses amino acid and intron position conservation to create gen models. In addition, GeMoMa allows to incorporate RNA-seq evidence for splice site prediction. (see more in [GeMoMa](http://www.jstacs.de/index.php/GeMoMa-Docs)).


In this section, we show how to run the main modules and analysis with GeMoMa using real data. We will start the genome annotation for our Clouded Leopard final genome. Since GeMoMa uses the annotation of coding genes from reference genome the first step is to download some reference genomes (taxa_assembly.fasta) with its genome annotation (taxa_annotation.gff). We have done this for you already to speed up the process. The reference genomes that we have are located here ```/data/genomics/workshops/smsc_2024/ref_genomes```. We have a total of 6 reference genomes for closely related taxa:

* Atlantisia rogersi (Island rail)
* Falco peregrinus (Peregrine falcon)
* Gallus gallus (red junglefowl )
* Strigops habroptila (Kākāpō)
* Zapornia atra (Henderson crake)
* Taeniopygia (Zebra finch)

If you wish to download other genomes you need to search on known genome databases or genome hubs. For instance, you can search on NCBI genomes and get the ftp url and download the files (gff and fasta) using wget. 

<details><summary>SOLUTION</summary>
<p>

```
cd /ref_genomes/
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/001/735/GCF_000001735.4_XXXX/GCF_00000XXXX_genomic.fna.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/001/735/GCF_000001735.4_XXXX/GCF_00000XXXX_genomic.gff.gz
cd ..
```
</p>
</details>


In General GeMoMa workflow performs 7 stepts: Extract RNA-seq evidence (ERE), DenoiseIntrons, Extractor, external search (tblastn or mmseqs), Gene Model Mapper (GeMoMa), GeMoMa Annotation Filter (GAF), and AnnnotationFinalizer. You can run this modules one by one or you can use the GeMoMa Pipeline, which is a multi-threaded tool that can uses all compute cores on one machine. 

#### Job file: gemoma_Guam_Rail.job
- Queue: high memory, medium time
- PE: multi-thread
- Number of CPUs: 40
- Memory: 12G (12G per CPU, 480G total)
- Module: `module load bio/gemoma/1.9`
- Commands:


```
set maxHeapSize 400000
GeMoMa -Xmx400G GeMoMaPipeline threads=$NSLOTS GeMoMa.Score=ReAlign AnnotationFinalizer.r=NO p=true o=true t=/path/to_assembly/bHypOws1_hifiasm.bp.p_ctg.fasta.gz outdir=output/ s=own i=G_gallus a=ref_genomes/Gallus_gallus/GCF_016699485.2_bGalGal1.mat.broiler.GRCg7b_genomic.gff.gz g=ref_genomes/Gallus_gallus/GCF_016699485.2_bGalGal1.mat.broiler.GRCg7b_genomic.fna.gz s=own i=A_rogersi a=ref_genomes/Atlantisia_rogersi/GCA_013401215.1_ASM1340121v1_genomic.gff.gz g=ref_genomes/Atlantisia_rogersi/GCA_013401215.1_ASM1340121v1_genomic.fna.gz s=own i=F_peregrinus a=ref_genomes/Falco_peregrinus/GCF_023634155.1_bFalPer1.pri_genomic.gff.gz g=ref_genomes/Falco_peregrinus/GCF_023634155.1_bFalPer1.pri_genomic.fna.gz s=own i=S_habroptila a=ref_genomes/Strigops_habroptila/GCF_004027225.2_bStrHab1.2.pri_genomic.gff.gz g=ref_genomes/Strigops_habroptila/GCF_004027225.2_bStrHab1.2.pri_genomic.fna.gz s=own i=Z_atra a=ref_genomes/Zapornia_atra/GCA_013400835.1_ASM1340083v1_genomic.gff.gz g=ref_genomes/Zapornia_atra/GCA_013400835.1_ASM1340083v1_genomic.fna.gz s=own i=Z_finch a=ref_genomes/Zebra_finch/GCF_003957565.2_bTaeGut1.4.pri_genomic.gff.gz g=ref_genomes/Zebra_finch/GCF_003957565.2_bTaeGut1.4.pri_genomic.fna.gz
```

##### Explanation:
```
threads: nnumber of CPUs
AnnotationFinalizer.r: to rename the predictions (NO/YES)
p: obtain predicted proteines (false/true)
o: keep and safe  individual predictions for each reference species (false/true)
t: target genome (fasta)
outdir: path and prefix of output directory
i: allows to provide an ID for each reference organism
g: path to individual reference genome
a: path to indvidual annotation genome
```

**Tips and Notes about GeMoMa PipeLine:**

* Depending on your default settings and the amount of data, you might have to increase the memory that can be used by setting the VM arguments for initial and maximal heap size, e.g., -Xms200G -Xmx400G.
* GeMoMaPipeline writes its version and all parameters that are no files at the beginning of the predicted annotation. This allows to check parameters at any time point.
* If GeMoMaPipline crashes with an Exception, the parameter restart can be used to restart the latest GeMoMaPipeline run, which was finished without results, with very similar parameters. This allows to avoid time-consuming steps like the search that were successful in the latest GeMoMaPipeline run.
* If you like to use mapped RNA-seq data, you have to use the parameters r and ERE.m:

```
GeMoMa -Xmx400G GeMoMaPipeline  threads=$NSLOTS AnnotationFinalizer.r=NO p=false o=true t=Guam_Rail.fna.gz outdir=output/ r=MAPPED ERE.m=<SAM/BAM> a=NCBI/GCF_XXXX_genomic.gff.gz g=NCBI/GCF_XXXX_genomic.fna.gz

```
* If you like to combine GeMoMa predictions with given external annotation external.gff, e.g., from ab-initio gene prediction, you can use the parameter e:

```
GeMoMa -Xmx400G GeMoMaPipeline threads=$NSLOTS AnnotationFinalizer.r=NO o=true t=clouded_leopard.fna.gz outdir=output/ 
p=true i=<REFERENCE_ID> a=NCBI/GCF_XXXX_genomic.gff.gz g=NCBI/GCF_XXXX_genomic.fna.gz ID=<EXTERNAL_ID> e=external.gff
```
* GeMoMa pipeline is extremely complex with many parameter for each of the module. Here is the full description of the paramenter for the pipeline script but also for each of the modules. [link to GeMoMa documentation](http://www.jstacs.de/index.php/GeMoMa-Docs)

