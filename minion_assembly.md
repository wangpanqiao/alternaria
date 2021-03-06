# Minion_Assemblies
==========

This document details the commands used to assemble and annotate minion genomes.

Note - all this work was performed in the directory:
/home/groups/harrisonlab/project_files/alternaria

The following is a summary of the work presented in this Readme.

The following processes were applied to Fusarium genomes prior to analysis:
Data qc
Genome assembly
Repeatmasking
Gene prediction
Functional annotation


# 0. Building of directory structure

#Building of directory structure


Data was basecalled again using Albacore 2.02 on the minion server:

```bash
screen -a
ssh nanopore@nanopore

# upgrade albacore
wget https://mirror.oxfordnanoportal.com/software/analysis/ont_albacore-2.1.10-cp34-cp34m-manylinux1_x86_64.whl
~/.local/bin/read_fast5_basecaller.py --version
pip3 install --user ont_albacore-2.1.10-cp34-cp34m-manylinux1_x86_64.whl --upgrade
~/.local/bin/read_fast5_basecaller.py --version

mkdir Aalt_21-12-17
cd Aalt_21-12-17

# Oxford nanopore 07/03/17
Organism=A.alternata
Date=05-02-18
FlowCell="FLO-MIN106"
Kit="SQK-RBK001"
RawDatDir=/data/seq_data/minion/2017/20171221_Alternaria-AA/Alternaria-AA/GA30000/reads
OutDir=/data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10
mkdir -p $OutDir

mkdir -p ~/Aalt_21-12-17/$Date
cd ~/Aalt_21-12-17/$Date
~/.local/bin/read_fast5_basecaller.py \
  --flowcell $FlowCell \
  --kit $Kit \
  --input $RawDatDir \
  --recursive \
  --worker_threads 24 \
  --save_path Alt_albacore_v2.10_demultiplexed \
  --output_format fastq,fast5 \
  --reads_per_fastq_batch 4000 \
  --barcoding
  cat Alt_albacore_v2.10_demultiplexed/workspace/pass/barcode01/*.fastq | gzip -cf > Alt_albacore_v2.10_barcode01.fastq.gz
  cat Alt_albacore_v2.10_demultiplexed/workspace/pass/barcode02/*.fastq | gzip -cf > Alt_albacore_v2.10_barcode02.fastq.gz
  OutDir=/data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10
  mkdir -p $OutDir
  chmod +w $OutDir
  cp Alt_albacore_v2.10_barcode0*.fastq.gz $OutDir/.
  chmod +rw $OutDir/Alt_albacore_v2.10_barcode0*.fastq.gz
  # scp Alt_albacore_v2.10_barcode*.fastq.gz armita@192.168.1.200:$OutDir/.
  tar -cz -f Alt_albacore_v2.10_demultiplexed.tar.gz Alt_albacore_v2.10_demultiplexed
  OutDir=/data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10
  mv Alt_albacore_v2.10_demultiplexed.tar.gz $OutDir/.
  chmod +rw $OutDir/Alt_albacore_v2.10_demultiplexed.tar.gz
```

Building a directory structure:

```bash
ProjDir=/home/groups/harrisonlab/project_files/alternaria
cd $ProjDir

Organism=A.alternata_ssp_tenuissima
Strain=1166
OutDir=raw_dna/minion/$Organism/$Strain
mkdir -p $OutDir
RawDat=$(ls /data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10/Alt_albacore_v2.10_barcode02.fastq.gz)
cd $OutDir
cp -s $RawDat .
cd $ProjDir

Organism=gaisen
Strain=650
OutDir=raw_dna/minion/$Organism/$Strain
mkdir -p $OutDir
RawDat=$(ls /data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10/Alt_albacore_v2.10_barcode01.fastq.gz)
cd $OutDir
cp -s $RawDat .
cd $ProjDir

```


<!--
### MiSeq data

```bash
	RawDatDir=/home/miseq_data/2017/RAW/170626_M04465_0043_000000000-B48RG/Data/Intensities/BaseCalls
	ProjectDir=/home/groups/harrisonlab/project_files/fusarium
	OutDir=$ProjectDir/raw_dna/paired/F.oxysporum_fsp_narcissi/FON_63
	mkdir -p $OutDir/F
	mkdir -p $OutDir/R
	cp $RawDatDir/FON63_S2_L001_R1_001.fastq.gz $OutDir/F/.
	cp $RawDatDir/FON63_S2_L001_R2_001.fastq.gz $OutDir/R/.
```


#### QC of MiSeq data

programs:
  fastqc
  fastq-mcf
  kmc

Data quality was visualised using fastqc:
```bash
	for RawData in $(ls raw_dna/paired/*/*/*/*.fastq.gz | grep 'FON_63'); do
		ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc
		echo $RawData;
		qsub $ProgDir/run_fastqc.sh $RawData
	done
```

```bash
	for StrainPath in $(ls -d raw_dna/paired/*/* | grep 'FON_63'); do
		ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/rna_qc
		IlluminaAdapters=/home/armita/git_repos/emr_repos/tools/seq_tools/ncbi_adapters.fa
		ReadsF=$(ls $StrainPath/F/*.fastq*)
		ReadsR=$(ls $StrainPath/R/*.fastq*)
		echo $ReadsF
		echo $ReadsR
		qsub $ProgDir/rna_qc_fastq-mcf.sh $ReadsF $ReadsR $IlluminaAdapters DNA
	done
``` -->

## Assembly

### Removal of adapters

Splitting reads and trimming adapters using porechop
```bash
	for RawReads in $(ls raw_dna/minion/*/*/*.fastq.gz); do
    Organism=$(echo $RawReads| rev | cut -f3 -d '/' | rev)
    Strain=$(echo $RawReads | rev | cut -f2 -d '/' | rev)
    echo "$Organism - $Strain"
  	OutDir=qc_dna/minion/$Organism/$Strain
  	ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc
  	qsub $ProgDir/sub_porechop.sh $RawReads $OutDir
  done
```


## Identify sequencing coverage

For Minion data:
```bash
for RawData in $(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/minion/*/*/*q.gz); do
echo $RawData;
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc;
GenomeSz=35
# OutDir=$(dirname $RawData)
OutDir=$(echo $RawData | cut -f11,12,13,14 -d '/')
mkdir -p $OutDir
qsub $ProgDir/sub_count_nuc.sh $GenomeSz $RawData $OutDir
done
```

```bash
  for StrainDir in $(ls -d qc_dna/minion/*/*); do
    Strain=$(basename $StrainDir)
    printf "$Strain\t"
    for File in $(ls $StrainDir/*.txt); do
      echo $(basename $File);
      cat $File | tail -n1 | rev | cut -f2 -d ' ' | rev;
    done | grep -v '.txt' | awk '{ SUM += $1} END { print SUM }'
  done
```

MinION coverage

```
650	39.48
1166	44.07
```

### Read correction using Canu

```bash
for TrimReads in $(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/minion/*/*/*q.gz); do
Organism=$(echo $TrimReads | rev | cut -f3 -d '/' | rev)
Strain=$(echo $TrimReads | rev | cut -f2 -d '/' | rev)
OutDir=assembly/canu-1.6/$Organism/"$Strain"
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/canu
qsub $ProgDir/sub_canu_correction.sh $TrimReads 35m $Strain $OutDir
done
```

### Assembbly using SMARTdenovo

```bash
for CorrectedReads in $(ls assembly/canu-1.6/*/*/*.trimmedReads.fasta.gz); do
Organism=$(echo $CorrectedReads | rev | cut -f3 -d '/' | rev)
Strain=$(echo $CorrectedReads | rev | cut -f2 -d '/' | rev)
Prefix="$Strain"_smartdenovo
OutDir=assembly/SMARTdenovo/$Organism/"$Strain"
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/SMARTdenovo
qsub $ProgDir/sub_SMARTdenovo.sh $CorrectedReads $Prefix $OutDir
done
```


Quast and busco were run to assess the effects of racon on assembly quality:

```bash
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
for Assembly in $(ls assembly/SMARTdenovo/*/*/*.dmo.lay.utg); do
  Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
  OutDir=$(dirname $Assembly)
  qsub $ProgDir/sub_quast.sh $Assembly $OutDir
done
```


```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/*.dmo.lay.utg); do
Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=gene_pred/busco/$Organism/$Strain/assembly
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```


Error correction using racon:

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/*.dmo.lay.utg); do
Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
ReadsFq=$(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/minion/*/$Strain/*q.gz)
# ReadsFq=$(ls qc_dna/minion/*/$Strain/*q.gz)
Iterations=10
OutDir=$(dirname $Assembly)"/racon2_$Iterations"
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/racon
qsub $ProgDir/sub_racon.sh $Assembly $ReadsFq $Iterations $OutDir
done
```

```bash
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
for Assembly in $(ls assembly/SMARTdenovo/A.*/*/racon2_10/*.fasta | grep 'round_10'); do
OutDir=$(dirname $Assembly)
echo "" > tmp.txt
ProgDir=~/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
$ProgDir/remove_contaminants.py --keep_mitochondria --inp $Assembly --out $OutDir/racon_min_500bp_renamed.fasta --coord_file tmp.txt > $OutDir/log.txt
done
```

Quast and busco were run to assess the effects of racon on assembly quality:

```bash
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
for Assembly in $(ls assembly/SMARTdenovo/A.*/*/racon2_10/racon_min_500bp_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
done
```


```bash
# for Assembly in $(ls assembly/SMARTdenovo/F.*/*/racon*/*.fasta | grep 'FON_63' | grep 'racon_min_500bp_renamed'); do
for Assembly in $(ls assembly/SMARTdenovo/A.*/*/racon2_10/*.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=gene_pred/busco/$Organism/$Strain/assembly
# OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```
```bash
printf "Filename\tComplete\tDuplicated\tFragmented\tMissing\tTotal\n"
for File in $(ls gene_pred/busco/A*/*/assembly/*/short_summary_*.txt); do
FileName=$(basename $File)
Complete=$(cat $File | grep "(C)" | cut -f2)
Duplicated=$(cat $File | grep "(D)" | cut -f2)
Fragmented=$(cat $File | grep "(F)" | cut -f2)
Missing=$(cat $File | grep "(M)" | cut -f2)
Total=$(cat $File | grep "Total" | cut -f2)
printf "$FileName\t$Complete\t$Duplicated\t$Fragmented\t$Missing\t$Total\n"
done
```


# Assembly correction using nanopolish

Fast5 files are very large and need to be stored as gzipped tarballs. These needed temporarily unpacking but must be deleted after nanpolish has finished running.

Raw reads were moved onto the cluster scratch space for this step and unpacked:

```bash
for Tar in $(ls /data/scratch/nanopore_tmp_data/Alternaria/albacore_v2.1.10/Alt_albacore_v2.10_demultiplexed.tar.gz); do
  ScratchDir=/data/scratch/armita/alternaria/albacore_v2.1.10
  mkdir -p $ScratchDir
  tar -zxvf $Tar -C $ScratchDir
done
```


```bash
for Assembly in $(ls assembly/SMARTdenovo/*/1166/racon2_10/racon_min_500bp_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
# Step 1 extract reads as a .fq file which contain info on the location of the fast5 files
# Note - the full path from home must be used
ReadDir=raw_dna/nanopolish/$Organism/$Strain
mkdir -p $ReadDir
ReadsFq=$(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/raw_dna/minion/*/$Strain/*.fastq.gz)
ScratchDir=/data/scratch/armita/alternaria
Fast5Dir=$ScratchDir/albacore_v2.1.10/Alt_albacore_v2.10_demultiplexed/workspace/pass/barcode01
# nanopolish index -d $Fast5Dir $ReadsFq
OutDir=$(dirname $Assembly)/nanopolish
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/nanopolish
# submit alignments for nanoppolish
qsub $ProgDir/sub_minimap2_nanopolish.sh $Assembly $ReadsFq $OutDir/nanopolish
done

for Assembly in $(ls assembly/SMARTdenovo/*/650/racon2_10/racon_min_500bp_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
# Step 1 extract reads as a .fq file which contain info on the location of the fast5 files
# Note - the full path from home must be used
ReadDir=raw_dna/nanopolish/$Organism/$Strain
mkdir -p $ReadDir
ReadsFq=$(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/raw_dna/minion/*/$Strain/*.fastq.gz)
ScratchDir=/data/scratch/armita/alternaria
Fast5Dir=$ScratchDir/albacore_v2.1.10/Alt_albacore_v2.10_demultiplexed/workspace/pass/barcode02
# nanopolish index -d $Fast5Dir $ReadsFq
OutDir=$(dirname $Assembly)
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/nanopolish
# submit alignments for nanoppolish
qsub $ProgDir/sub_minimap2_nanopolish.sh $Assembly $ReadsFq $OutDir/nanopolish
done
```

 Split the assembly into 50Kb fragments an submit each to the cluster for
 nanopolish correction

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/racon2_10/racon_min_500bp_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
OutDir=$(dirname $Assembly)/nanopolish
RawReads=$(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/raw_dna/minion/*/$Strain/*.fastq.gz)
AlignedReads=$(ls $OutDir/reads.sorted.bam)

NanoPolishDir=/home/armita/prog/nanopolish/nanopolish/scripts
python $NanoPolishDir/nanopolish_makerange.py $Assembly --segment-length 50000 > $OutDir/nanopolish_range.txt

Ploidy=1
echo "nanopolish log:" > $OutDir/nanopolish_log.txt
ls -lh $OutDir/*/*.fa | grep -v ' 0 ' | cut -f8 -d '/' | sed 's/_consensus.fa//g' > $OutDir/files_present.txt
for Region in $(cat $OutDir/nanopolish_range.txt | grep -vwf "$OutDir/files_present.txt" | head -n1); do
# for Region in $(cat $OutDir/nanopolish_range.txt); do
Jobs=$(qstat | grep 'sub_nanopo' | grep 'qw' | wc -l)
while [ $Jobs -gt 1 ]; do
sleep 1m
printf "."
Jobs=$(qstat | grep 'sub_nanopo' | grep 'qw' | wc -l)
done		
printf "\n"
echo $Region
echo $Region >> $OutDir/nanopolish_log.txt
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/nanopolish
qsub $ProgDir/sub_nanopolish_variants.sh $Assembly $RawReads $AlignedReads $Ploidy $Region $OutDir/$Region
done
done
```

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/racon2_10/racon_min_500bp_renamed.fasta | grep '1166'); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
OutDir=assembly/SMARTdenovo/$Organism/$Strain/nanopolish
mkdir -p $OutDir
# cat "" > $OutDir/"$Strain"_nanoplish.fa
NanoPolishDir=/home/armita/prog/nanopolish/nanopolish/scripts
InDir=$(dirname $Assembly)
python $NanoPolishDir/nanopolish_merge.py $InDir/nanopolish/*/*.fa > $OutDir/"$Strain"_nanoplish.fa

echo "" > tmp.txt
ProgDir=~/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
$ProgDir/remove_contaminants.py --keep_mitochondria --inp $OutDir/"$Strain"_nanoplish.fa --out $OutDir/"$Strain"_nanoplish_min_500bp_renamed.fasta --coord_file tmp.txt > $OutDir/log.txt
done
```

Quast and busco were run to assess the effects of nanopolish on assembly quality:

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/nanopolish/*_nanoplish_min_500bp_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
# Quast
OutDir=$(dirname $Assembly)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
# Busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=gene_pred/busco/$Organism/$Strain/assembly
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```


### Pilon assembly correction

Assemblies were polished using Pilon

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/nanopolish/*_nanoplish_min_500bp_renamed.fasta); do
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
IlluminaDir=$(ls -d ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/$Strain)
TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz | head -n1)
TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz | head -n1)
OutDir=$(dirname $Assembly)/../pilon
Iterations=10
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/pilon
qsub $ProgDir/sub_pilon.sh $Assembly $TrimF1_Read $TrimR1_Read $OutDir $Iterations
done
```

Contigs were renamed
```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/pilon/*.fasta | grep 'pilon_10'); do
echo $Assembly
echo "" > tmp.txt
OutDir=$(dirname $Assembly)
ProgDir=~/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
$ProgDir/remove_contaminants.py --keep_mitochondria --inp $Assembly --out $OutDir/pilon_min_500bp_renamed.fasta --coord_file tmp.txt > $OutDir/log.txt
done
```

Quast and busco were run to assess the effects of pilon on assembly quality:

```bash

for Assembly in $(ls assembly/SMARTdenovo/*/*/pilon/*.fasta); do
  Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
  echo "$Organism - $Strain"
  # OutDir=$(dirname $Assembly)
  # ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
  # qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  Jobs=$(qstat | grep 'sub_busco' | grep 'qw'| wc -l)
  while [ $Jobs -gt 1 ]; do
  sleep 1m
  printf "."
  Jobs=$(qstat | grep 'sub_busco' | grep 'qw'| wc -l)
  done
  printf "\n"
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
  BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
  # OutDir=gene_pred/busco/$Organism/$Strain/assembly
  OutDir=$(dirname $Assembly)
  qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

```bash
printf "Filename\tComplete\tDuplicated\tFragmented\tMissing\tTotal\n"
for File in $(ls assembly/SMARTdenovo/*/*/pilon/*/short_summary_*.txt); do  
FileName=$(basename $File)
Complete=$(cat $File | grep "(C)" | cut -f2)
Duplicated=$(cat $File | grep "(D)" | cut -f2)
Fragmented=$(cat $File | grep "(F)" | cut -f2)
Missing=$(cat $File | grep "(M)" | cut -f2)
Total=$(cat $File | grep "Total" | cut -f2)
printf "$FileName\t$Complete\t$Duplicated\t$Fragmented\t$Missing\t$Total\n"
done
```

```
short_summary_pilon_1.txt	1301	1	4	10	1315
short_summary_pilon_2.txt	1302	1	3	10	1315
short_summary_pilon_3.txt	1302	1	3	10	1315
short_summary_pilon_4.txt	1302	1	3	10	1315
short_summary_pilon_5.txt	1302	1	3	10	1315
short_summary_pilon_6.txt	1302	1	3	10	1315
short_summary_pilon_7.txt	1302	1	3	10	1315
short_summary_pilon_8.txt	1302	1	3	10	1315
short_summary_pilon_9.txt	1302	1	3	10	1315
short_summary_pilon_10.txt	1302	1	3	10	1315
short_summary_pilon_min_500bp_renamed.txt	1302	1	3	10	131
short_summary_pilon_1.txt	1290	2	8	17	1315
short_summary_pilon_2.txt	1294	2	5	16	1315
short_summary_pilon_3.txt	1298	2	5	12	1315
short_summary_pilon_4.txt	1298	2	5	12	1315
short_summary_pilon_5.txt	1298	2	5	12	1315
short_summary_pilon_6.txt	1298	2	5	12	1315
short_summary_pilon_7.txt	1298	2	5	12	1315
short_summary_pilon_8.txt	1298	2	5	12	1315
short_summary_pilon_9.txt	1298	2	5	12	1315
short_summary_pilon_10.txt	1298	2	5	12	1315
short_summary_pilon_min_500bp_renamed.txt	1298	2	5	12	1315
```

# Hybrid Assembly

## Spades Assembly

```bash
for TrimReads in $(ls ../../../../../home/groups/harrisonlab/project_files/alternaria/raw_dna/minion/*/*/*.fastq.gz); do
Organism=$(echo $TrimReads | rev | cut -f3 -d '/' | rev)
Strain=$(echo $TrimReads | rev | cut -f2 -d '/' | rev)
IlluminaDir=$(ls -d ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/$Strain)
TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz | head -n1)
TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz | head -n1)
OutDir=assembly/spades_minion/$Organism/"$Strain"
echo $TrimF1_Read
echo $TrimR1_Read
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/spades
qsub $ProgDir/sub_spades_minion.sh $TrimReads $TrimF1_Read $TrimR1_Read $OutDir
done
```

Contigs shorter than 500bp were removed from the assembly

```bash
  for Contigs in $(ls assembly/spades_minion/*/*/contigs.fasta); do
    AssemblyDir=$(dirname $Contigs)
    mkdir $AssemblyDir/filtered_contigs
    FilterDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/abyss
    $FilterDir/filter_abyss_contigs.py $Contigs 500 > $AssemblyDir/filtered_contigs/contigs_min_500bp.fasta
  done
```


Quast and BUSCO

```bash
for Assembly in $(ls assembly/spades_minion/*/*/filtered_contigs/contigs_min_500bp.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
OutDir=$(dirname $Assembly)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
# BuscoDB="Fungal"
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```


```bash
  for File in $(ls assembly/spades_minion/A.*/*/filtered_contigs/run_contigs_min_500bp/short_summary_*.txt | grep -v 'arborescens'); do
  Strain=$(echo $File| rev | cut -d '/' -f5 | rev)
  Organism=$(echo $File | rev | cut -d '/' -f6 | rev)
  Prefix=$(basename $File)
  Complete=$(cat $File | grep "(C)" | cut -f2)
  Fragmented=$(cat $File | grep "(F)" | cut -f2)
  Missing=$(cat $File | grep "(M)" | cut -f2)
  Total=$(cat $File | grep "Total" | cut -f2)
  echo -e "$Organism\t$Strain\t$Prefix\t$Complete\t$Fragmented\t$Missing\t$Total"
  done
```

```
spades_minion	A.alternata_ssp_tenuissima	short_summary_contigs_min_500bp.txt	1302	3	10	1315
spades_minion	A.gaisen	short_summary_contigs_min_500bp.txt	1302	3	10	1315
```

## Merging pacbio and hybrid assemblies


Merging was noted to improve the 1166 apple pathotype assembly but not the 650 pear pathotype assembly

```bash
for MinionAssembly in $(ls assembly/SMARTdenovo/*/*/pilon/pilon_min_500bp_renamed.fasta | grep '1166'); do
Organism=$(echo $MinionAssembly | rev | cut -f4 -d '/' | rev)
Strain=$(echo $MinionAssembly | rev | cut -f3 -d '/' | rev)
HybridAssembly=$(ls assembly/spades_*/${Organism}/${Strain}*/filtered_contigs/contigs_min_500bp.fasta)
OutDir=assembly/merged_SMARTdenovo_spades/$Organism/${Strain}
AnchorLength=500000
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/quickmerge
qsub $ProgDir/sub_quickmerge.sh $MinionAssembly $HybridAssembly $OutDir $AnchorLength
done
```

Quast and BUSCO

```bash
for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/merged.fasta | grep '1166'); do
Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
OutDir=$(dirname $Assembly)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

```bash
  for File in $(ls assembly/merged_SMARTdenovo_spades/*/*/*/short_summary_*.txt); do
  Strain=$(echo $File| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $File | rev | cut -d '/' -f4 | rev)
  Prefix=$(basename $File)
  Complete=$(cat $File | grep "(C)" | cut -f2)
  Fragmented=$(cat $File | grep "(F)" | cut -f2)
  Missing=$(cat $File | grep "(M)" | cut -f2)
  Total=$(cat $File | grep "Total" | cut -f2)
  echo -e "$Organism\t$Strain\t$Prefix\t$Complete\t$Fragmented\t$Missing\t$Total"
  done
```

This merged assembly was polished using Pilon

```bash
for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/merged.fasta | grep '1166' | grep -v -e '_100kb' -e 'hybrid_first'); do
Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
echo "$Organism - $Strain"
IlluminaDir=$(ls -d ../../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/$Strain)
TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz | head -n1)
TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz | head -n1)
OutDir=$(dirname $Assembly)/pilon
Iterations=5
# OutDir=$(dirname $Assembly)/pilon2
# Iterations=10
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/pilon
qsub $ProgDir/sub_pilon.sh $Assembly $TrimF1_Read $TrimR1_Read $OutDir $Iterations
done
```

Contigs were renamed
```bash
for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon/*.fasta | grep 'pilon_5' | grep '1166'); do
# for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon2/*.fasta | grep 'pilon_10'); do
echo $Assembly
echo "" > tmp.txt
OutDir=$(dirname $Assembly)
ProgDir=~/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
$ProgDir/remove_contaminants.py --keep_mitochondria --inp $Assembly --out $OutDir/pilon_min_500bp_renamed.fasta --coord_file tmp.txt > $OutDir/log.txt
done
```


BUSCO

```bash

for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon/*.fasta | grep '1166'); do
# for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon2/*.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
Jobs=$(qstat | grep 'sub_busco' | grep 'qw'| wc -l)
while [ $Jobs -gt 1 ]; do
sleep 1m
printf "."
Jobs=$(qstat | grep 'sub_busco' | grep 'qw'| wc -l)
done
printf "\n"
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

```bash
  for File in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon/*/short_summary_*.txt | grep '650'); do
  Strain=$(echo $File| rev | cut -d '/' -f4 | rev)
  Organism=$(echo $File | rev | cut -d '/' -f5 | rev)
  Prefix=$(basename $File)
  Complete=$(cat $File | grep "(C)" | cut -f2)
  Fragmented=$(cat $File | grep "(F)" | cut -f2)
  Missing=$(cat $File | grep "(M)" | cut -f2)
  Total=$(cat $File | grep "Total" | cut -f2)
  echo -e "$Organism\t$Strain\t$Prefix\t$Complete\t$Fragmented\t$Missing\t$Total"
  done
```

```
A.alternata_ssp_tenuissima	1166	short_summary_pilon_1.txt	1302	3	10	1315
A.alternata_ssp_tenuissima	1166	short_summary_pilon_2.txt	1302	3	10	1315
A.alternata_ssp_tenuissima	1166	short_summary_pilon_3.txt	1302	3	10	1315
A.alternata_ssp_tenuissima	1166	short_summary_pilon_4.txt	1302	3	10	1315
A.alternata_ssp_tenuissima	1166	short_summary_pilon_5.txt	1302	3	10	1315
A.alternata_ssp_tenuissima	1166	short_summary_pilon_min_500bp_renamed.txt	1302	3	10 1315

A.gaisen	650	short_summary_pilon_1.txt	1260	5	50	1315
A.gaisen	650	short_summary_pilon_2.txt	1260	5	50	1315
A.gaisen	650	short_summary_pilon_3.txt	1260	5	50	1315
A.gaisen	650	short_summary_pilon_4.txt	1260	5	50	1315
A.gaisen	650	short_summary_pilon_5.txt	1260	5	50	1315
A.gaisen	650	short_summary_pilon_min_500bp_renamed.txt	1260	5	50	1315
```

The results of merging showed worse results than when the assembly was performed
using MinION only data for A. gaisen isolate 650. As such, the hybrid assembly
was used for the apple pathotype genome and the SMARTdenovo assembly used for the
pear pathotype genome.



## Identifying Mitochondrial genes in assemblies

Using a blast based approach of Mt genes:

```bash
for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/1166/pilon/pilon_min_500bp_renamed.fasta assembly/SMARTdenovo/A.gaisen/650/pilon/pilon_min_500bp_renamed.fasta); do
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
echo $Assembly
Query=$(ls ../../../../home/groups/harrisonlab/project_files/alternaria/analysis/blast_homology/Mt_dna/Mt_genes_A.alternata_Liao_2017.fasta)
OutDir=analysis/Mt_genes/$Organism/$Strain
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
qsub $ProgDir/run_blast2csv.sh $Query protein $Assembly $OutDir
done
```

Using an exclusion database with deconseq:


```bash
  for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/1166/pilon/pilon_min_500bp_renamed.fasta assembly/SMARTdenovo/A.gaisen/650/pilon/pilon_min_500bp_renamed.fasta); do
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
    echo "$Organism - $Strain"
    for Exclude_db in "Aalt_mtDNA"; do
      AssemblyDir=$(dirname $Assembly)
      OutDir=$AssemblyDir/../deconseq_$Exclude_db
      ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
      qsub $ProgDir/sub_deconseq_no_retain.sh $Assembly $Exclude_db $OutDir
    done
  done
```

Results were summarised using the commands:

```bash
for Exclude_db in "Aalt_mtDNA"; do
echo $Exclude_db
for File in $(ls assembly/*/*/*/*/log.txt | grep "$Exclude_db"); do
  echo $File
Name=$(echo $File | rev | cut -f3 -d '/' | rev);
Good=$(cat $File |cut -f2 | head -n1 | tail -n1);
Bad=$(cat $File |cut -f2 | head -n3 | tail -n1);
printf "$Name\t$Good\t$Bad\n";
done
done
```

```
Aalt_mtDNA
1166	22	1
650	23	1
650	27	1
```

Quast was run on the removed mtDNA:

```bash
for Assembly in $(ls assembly/*/A*/*/deconseq_Aalt_mtDNA/*_cont.fa); do
Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
OutDir=$(dirname $Assembly)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
done
```

# Repeat Masking

Repeat masking was performed on the hybrid assembly.

```bash
  # for Assembly in $(ls assembly/merged_SMARTdenovo_spades/*/*/pilon2/pilon_min_500bp_renamed.fasta); do
  for Assembly in $(ls assembly/merged_SMARTdenovo_spades/A.alternata_ssp_tenuissima/1166/deconseq_Aalt_mtDNA/contigs_min_500bp_filtered_renamed.fasta  assembly/SMARTdenovo/A.gaisen/650/deconseq_Aalt_mtDNA/contigs_min_500bp_filtered_renamed.fasta | grep '650'); do
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    echo "$Organism - $Strain"
    OutDir=repeat_masked/$Organism/"$Strain"/filtered_contigs
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/repeat_masking
    qsub $ProgDir/rep_modeling.sh $Assembly $OutDir
    qsub $ProgDir/transposonPSI.sh $Assembly $OutDir
  done
```

The TransposonPSI masked bases were used to mask additional bases from the
repeatmasker / repeatmodeller softmasked and hardmasked files.

```bash

for File in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked.fa); do
OutDir=$(dirname $File)
TPSI=$(ls $OutDir/*_contigs_unmasked.fa.TPSI.allHits.chains.gff3)
OutFile=$(echo $File | sed 's/_contigs_softmasked.fa/_contigs_softmasked_repeatmasker_TPSI_appended.fa/g')
echo "$OutFile"
bedtools maskfasta -soft -fi $File -bed $TPSI -fo $OutFile
echo "Number of masked bases:"
cat $OutFile | grep -v '>' | tr -d '\n' | awk '{print $0, gsub("[a-z]", ".")}' | cut -f2 -d ' '
done
# The number of N's in hardmasked sequence are not counted as some may be present within the assembly and were therefore not repeatmasked.
for File in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_hardmasked.fa); do
OutDir=$(dirname $File)
TPSI=$(ls $OutDir/*_contigs_unmasked.fa.TPSI.allHits.chains.gff3)
OutFile=$(echo $File | sed 's/_contigs_hardmasked.fa/_contigs_hardmasked_repeatmasker_TPSI_appended.fa/g')
echo "$OutFile"
bedtools maskfasta -fi $File -bed $TPSI -fo $OutFile
done
```
```
repeat_masked/A.alternata_ssp_tenuissima/1166/filtered_contigs/1166_contigs_softmasked_repeatmasker_TPSI_appended.fa
Number of masked bases:
1268326
repeat_masked/A.gaisen/650/filtered_contigs/650_contigs_softmasked_repeatmasker_TPSI_appended.fa
Number of masked bases:
749954
```

```bash
for File in $(ls repeat_masked/A.*/*/filtered_contigs/*_contigs_unmasked.fa.TPSI.allHits.chains.bestPerLocus.gff3); do
Strain=$(echo $File| rev | cut -d '/' -f3 | rev)
Organism=$(echo $File | rev | cut -d '/' -f4 | rev)
# echo "$Organism - $Strain"
DDE_1=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'DDE_1' | sed "s/^\s*//g" | cut -f1 -d ' ')
Gypsy=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'gypsy' | sed "s/^\s*//g" | cut -f1 -d ' ')
HAT=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'hAT' | sed "s/^\s*//g" | cut -f1 -d ' ')
TY1_Copia=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'TY1_Copia' | sed "s/^\s*//g" | cut -f1 -d ' ')
Mariner=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep -w 'mariner' | sed "s/^\s*//g" | cut -f1 -d ' ')
Cacta=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'cacta' | sed "s/^\s*//g" | cut -f1 -d ' ')
LINE=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'LINE' | sed "s/^\s*//g" | cut -f1 -d ' ')
MuDR_A_B=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'MuDR_A_B' | sed "s/^\s*//g" | cut -f1 -d ' ')
HelitronORF=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'helitronORF' | sed "s/^\s*//g" | cut -f1 -d ' ')
Mariner_ant1=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'mariner_ant1' | sed "s/^\s*//g" | cut -f1 -d ' ')
ISC1316=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'ISC1316' | sed "s/^\s*//g" | cut -f1 -d ' ')
Crypton=$(cat $File | cut -f9 | cut -f2 -d';' | sort| uniq -c | grep 'Crypton' | sed "s/^\s*//g" | cut -f1 -d ' ')
printf "$Organism\t$Strain\t$DDE_1\t$Gypsy\t$HAT\t$TY1_Copia\t$Mariner\t$Cacta\t$LINE\t$MuDR_A_B\t$HelitronORF\t$Mariner_ant1\t$ISC1316\t$Crypton\n"
done
```

Quast and BUSCO

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
OutDir=$(dirname $Assembly)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
OutDir=$(dirname $Assembly)
qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```


```bash
  for File in $(ls repeat_masked/*/*/filtered_contigs/run_*_contigs_unmasked/short_summary_*.txt); do
  Strain=$(echo $File| rev | cut -d '/' -f4 | rev)
  Organism=$(echo $File | rev | cut -d '/' -f5 | rev)
  Complete=$(cat $File | grep "(C)" | cut -f2)
  Fragmented=$(cat $File | grep "(F)" | cut -f2)
  Missing=$(cat $File | grep "(M)" | cut -f2)
  Total=$(cat $File | grep "Total" | cut -f2)
  echo -e "$Organism\t$Strain\t$Complete\t$Fragmented\t$Missing\t$Total"
  done
```

```
A.alternata_ssp_tenuissima	1166	1302	3	10	1315
A.gaisen	650	1298	5	12	1315
```

```bash
  for File in $(ls repeat_masked/*/*/filtered_contigs/report.tsv); do
  Strain=$(echo $File| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $File | rev | cut -d '/' -f4 | rev)
  Contigs=$(cat $File | grep "contigs (>= 0 bp)" | cut -f2)
  Length=$(cat $File | grep "Total length (>= 0 bp)" | cut -f2)
  Largest=$(cat $File | grep "Largest contig" | cut -f2)
  N50=$(cat $File | grep "N50" | cut -f2)
  echo -e "$Organism\t$Strain\t$Contigs\t$Length\t$Largest\t$N50"
  done
```

```
A.alternata_ssp_tenuissima	1166	22	35704180	3902980	2583941
A.gaisen	650	27	34346950	6257968	2110033
```

KAT kmer spectra analysis

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  IlluminaDir=$(ls -d ../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/$Strain)
  cat $IlluminaDir/F/*.fastq.gz > $IlluminaDir/F/F_trim_appended.fq.gz
  cat $IlluminaDir/F/*.fastq.gz > $IlluminaDir/R/R_trim_appended.fq.gz
  ReadsF=$(ls $IlluminaDir/F/F_trim_appended.fq.gz)
  ReadsR=$(ls $IlluminaDir/R/R_trim_appended.fq.gz)
  OutDir=$(dirname $Assembly)/kat
  Prefix="repeat_masked"
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/kat
  qsub $ProgDir/sub_kat.sh $Assembly $ReadsF $ReadsR $OutDir $Prefix
done
```

after KAT jobs have finished running, then remove appended trimmed reads
```bash
  rm ../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/*/*/F_trim_appended.fq.gz
  rm ../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/*/*/R_trim_appended.fq.gz
```

# Identify Telomere repeats:

Telomeric repeats were identified in assemblies

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
OutDir=analysis/telomere/$Organism/$Strain
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/telomeres
$ProgDir/annotate_telomeres.py --fasta $Assembly --out $OutDir/telomere_hits
# Motif="TTAGGG"
# for Strand in '+' '-'; do
# ProgDir=/home/armita/git_repos/emr_repos/scripts/popgen/codon
# python $ProgDir/identify_telomere_repeats.py $Assembly $Motif $Strand $OutDir/${Strain}_telomere_${Motif}_${Strand}.bed
# $ProgDir/how_many_repeats_regions.py $OutDir/${Strain}_telomere_${Motif}_${Strand}.bed
# done
done
cat $OutDir/telomere_hits.txt | sort -nr -k5 | less
```

# Gene Prediction

#### Aligning


```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa | grep '650'); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
for FileF in $(ls ../../../../home/groups/harrisonlab/project_files/alternaria/qc_rna/paired/*/*/F/*.fastq.gz); do
Jobs=$(qstat | grep 'sub_sta' | grep 'qw'| wc -l)
while [ $Jobs -gt 1 ]; do
sleep 1m
printf "."
Jobs=$(qstat | grep 'sub_sta' | grep 'qw'| wc -l)
done
printf "\n"
FileR=$(echo $FileF | sed 's&/F/&/R/&g' | sed 's/F.fastq.gz/R.fastq.gz/g')
echo $FileF
echo $FileR
Prefix=$(echo $FileF | rev | cut -f3 -d '/' | rev)
# Timepoint=$(echo $FileF | rev | cut -f2 -d '/' | rev)
Timepoint="treatment"
#echo "$Timepoint"
OutDir=alignment/star/$Organism/$Strain/$Timepoint/$Prefix
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/RNAseq
qsub $ProgDir/sub_star.sh $Assembly $FileF $FileR $OutDir
done
done
```

Accepted hits .bam file were concatenated and indexed for use for gene model training:

```bash
  for OutDir in $(ls -d alignment/star/*/* | grep '650'); do
    Strain=$(echo $OutDir | rev | cut -d '/' -f1 | rev)
    Organism=$(echo $OutDir | rev | cut -d '/' -f2 | rev)
    echo "$Organism - $Strain"
    # For all alignments
    BamFiles=$(ls $OutDir/treatment/*/*.sortedByCoord.out.bam | tr -d '\n' | sed 's/.bam/.bam /g')
    mkdir -p $OutDir/concatenated
    samtools merge -@ 12 -f $OutDir/concatenated/concatenated.bam $BamFiles
  done
```


#### Braker prediction

Before braker predictiction was performed, I double checked that I had the
genemark key in my user area and copied it over from the genemark install
directory:

```bash
	ls ~/.gm_key
	cp /home/armita/prog/genemark/gm_key_64 ~/.gm_key
```

```bash
  for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
    Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    OutDir=gene_pred/braker/$Organism/"$Strain"_braker
    AcceptedHits=$(ls alignment/star/$Organism/$Strain/concatenated/concatenated.bam)
    GeneModelName="$Organism"_"$Strain"_braker
    rm -r /home/armita/prog/augustus-3.1/config/species/"$Organism"_"$Strain"_braker
    ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/braker1
    qsub $ProgDir/sub_braker_fungi.sh $Assembly $OutDir $AcceptedHits $GeneModelName
  done
```


## Supplimenting Braker gene models with CodingQuary genes

Additional genes were added to Braker gene predictions, using CodingQuary in
pathogen mode to predict additional regions.

Firstly, aligned RNAseq data was assembled into transcripts using Cufflinks.

Note - cufflinks doesn't always predict direction of a transcript and
therefore features can not be restricted by strand when they are intersected.

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
OutDir=gene_pred/cufflinks/$Organism/$Strain/concatenated
mkdir -p $OutDir
AcceptedHits=$(ls alignment/star/$Organism/$Strain/concatenated/concatenated.bam)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/RNAseq
qsub $ProgDir/sub_cufflinks.sh $AcceptedHits $OutDir
done
```

Secondly, genes were predicted using CodingQuary:

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
Jobs=$(qstat | grep 'sub_cuff' | wc -l)
while [ $Jobs -gt '0' ]; do
sleep 10
printf "."
Jobs=$(qstat | grep 'sub_cuff' | wc -l)
done
printf "\n"
OutDir=gene_pred/codingquary/$Organism/${Strain}
CufflinksGTF=$(ls gene_pred/cufflinks/$Organism/$Strain/concatenated/transcripts.gtf)
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
qsub $ProgDir/sub_CodingQuary.sh $Assembly $CufflinksGTF $OutDir
done
```

The next step had problems with the masked pacbio genome. Bioperl could not read in
the fasta sequences. This was overcome by wrapping the unmasked genome and using this
fasta file.

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
NewName=$(echo $Assembly | sed 's/_unmasked.fa/_unmasked_wrapped.fa/g')
cat $Assembly | fold > $NewName
done
```

Then, additional transcripts were added to Braker gene models, when CodingQuary
genes were predicted in regions of the genome, not containing Braker gene
models:

```bash
for BrakerGff in $(ls gene_pred/braker/*/*_braker/*/augustus.gff3 | grep '650'); do
Strain=$(echo $BrakerGff| rev | cut -d '/' -f3 | rev | sed 's/_braker_new//g' | sed 's/_braker_pacbio//g' | sed 's/_braker//g')
Organism=$(echo $BrakerGff | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
# Assembly=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa)
Assembly=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_unmasked_wrapped.fa)
CodingQuaryGff=$(ls gene_pred/codingquary/$Organism/$Strain/out/PredictedPass.gff3)
PGNGff=$(ls gene_pred/codingquary/$Organism/$Strain/out/PGN_predictedPass.gff3)
AddDir=gene_pred/codingquary/$Organism/$Strain/additional
FinalDir=gene_pred/final/$Organism/$Strain/final
AddGenesList=$AddDir/additional_genes.txt
AddGenesGff=$AddDir/additional_genes.gff
FinalGff=$AddDir/combined_genes.gff
mkdir -p $AddDir
mkdir -p $FinalDir

for x in $CodingQuaryGff $PGNGff; do
  bedtools intersect -v -a $x -b $BrakerGff | grep 'gene'| cut -f2 -d'=' | cut -f1 -d';'
done > $AddGenesList

for y in $CodingQuaryGff $PGNGff; do
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation
  $ProgDir/gene_list_to_gff.pl $AddGenesList $y CodingQuarry_v2.0 ID CodingQuary
done > $AddGenesGff
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
# -
# This section is edited
$ProgDir/add_CodingQuary_features.pl $AddGenesGff $Assembly > $AddDir/add_genes_CodingQuary_unspliced.gff3
$ProgDir/correct_CodingQuary_splicing.py --inp_gff $AddDir/add_genes_CodingQuary_unspliced.gff3 > $FinalDir/final_genes_CodingQuary.gff3
# -
$ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_CodingQuary.gff3 $FinalDir/final_genes_CodingQuary
cp $BrakerGff $FinalDir/final_genes_Braker.gff3
$ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_Braker.gff3 $FinalDir/final_genes_Braker
cat $FinalDir/final_genes_Braker.pep.fasta $FinalDir/final_genes_CodingQuary.pep.fasta | sed -r 's/\*/X/g' > $FinalDir/final_genes_combined.pep.fasta
cat $FinalDir/final_genes_Braker.cdna.fasta $FinalDir/final_genes_CodingQuary.cdna.fasta > $FinalDir/final_genes_combined.cdna.fasta
cat $FinalDir/final_genes_Braker.gene.fasta $FinalDir/final_genes_CodingQuary.gene.fasta > $FinalDir/final_genes_combined.gene.fasta
cat $FinalDir/final_genes_Braker.upstream3000.fasta $FinalDir/final_genes_CodingQuary.upstream3000.fasta > $FinalDir/final_genes_combined.upstream3000.fasta

GffBraker=$FinalDir/final_genes_CodingQuary.gff3
GffQuary=$FinalDir/final_genes_Braker.gff3
GffAppended=$FinalDir/final_genes_appended.gff3
cat $GffBraker $GffQuary > $GffAppended
done
```

The final number of genes per isolate was observed using:
```bash
  for DirPath in $(ls -d gene_pred/final/*/*/final); do
    Strain=$(echo $DirPath| rev | cut -d '/' -f2 | rev)
    Organism=$(echo $DirPath | rev | cut -d '/' -f3 | rev)
    echo "$Organism - $Strain"
    cat $DirPath/final_genes_Braker.pep.fasta | grep '>' | wc -l;
    cat $DirPath/final_genes_CodingQuary.pep.fasta | grep '>' | wc -l;
    cat $DirPath/final_genes_combined.pep.fasta | grep '>' | wc -l;
    echo "";
  done
```

The number of genes predicted by Braker, supplimented by CodingQuary and in the
final combined dataset was shown:

```
A.alternata_ssp_tenuissima - 1166
12761
904
13665

A.gaisen - 650
12351
875
13226
```


In preperation for submission to ncbi, gene models were renamed and duplicate gene features were identified and removed.
 * no duplicate genes were identified



```bash
for GffAppended in $(ls gene_pred/final/*/*/final/final_genes_appended.gff3 | grep '650'); do
Strain=$(echo $GffAppended | rev | cut -d '/' -f3 | rev)
Organism=$(echo $GffAppended | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
FinalDir=gene_pred/final/$Organism/$Strain/final
GffFiltered=$FinalDir/filtered_duplicates.gff
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
$ProgDir/remove_dup_features.py --inp_gff $GffAppended --out_gff $GffFiltered
GffRenamed=$FinalDir/final_genes_appended_renamed.gff3
LogFile=$FinalDir/final_genes_appended_renamed.log
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
$ProgDir/gff_rename_genes.py --inp_gff $GffFiltered --conversion_log $LogFile > $GffRenamed
rm $GffFiltered
# Assembly=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_softmasked_repeatmasker_TPSI_appended.fa)
Assembly=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_unmasked_wrapped.fa)
$ProgDir/gff2fasta.pl $Assembly $GffRenamed gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed
# The proteins fasta file contains * instead of Xs for stop codons, these should
# be changed
sed -i 's/\*/X/g' gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.pep.fasta
done
```

```
A.alternata_ssp_tenuissima - 1166
Identifiied the following duplicated transcripts:
CUFF_8299_1_996.t2
CUFF_10096_1_77.t2
A.gaisen - 650
Identifiied the following duplicated transcripts:
CUFF_7300_1_36.t2
CUFF_739_1_82.t2
CUFF_5702_1_539.t2
CUFF_1632_1_77.t2
CUFF_9393_1_13.t2
CUFF_989_1_1564.t2
```

```bash
for File in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
  echo $File | rev | cut -f3 -d '/' |  rev
  cat $File | grep '>' | wc -l
done
```

```
1166
13663
650
13220
```

## Checking gene prediction accruacy using BUSCO

```bash
 for Assembly in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.gene.fasta); do
   Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
   Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
   echo "$Organism - $Strain"
   ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
   BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/ascomycota_odb9)
   OutDir=gene_pred/busco/$Organism/$Strain/transcript
   qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
 done
```

```bash
 for File in $(ls gene_pred/busco/*/*/transcript/*/short_summary_*.txt); do
 Strain=$(echo $File| rev | cut -d '/' -f4 | rev)
 Organism=$(echo $File | rev | cut -d '/' -f5 | rev)
 Complete=$(cat $File | grep "(C)" | cut -f2)
 Single=$(cat $File | grep "(S)" | cut -f2)
 Fragmented=$(cat $File | grep "(F)" | cut -f2)
 Missing=$(cat $File | grep "(M)" | cut -f2)
 Total=$(cat $File | grep "Total" | cut -f2)
 echo -e "$Organism\t$Strain\t$Complete\t$Single\t$Fragmented\t$Missing\t$Total"
 done
```


#Functional annotation

## A) Interproscan

Interproscan was used to give gene models functional annotations.
Annotation was run using the commands below:

Note: This is a long-running script. As such, these commands were run using
'screen' to allow jobs to be submitted and monitored in the background.
This allows the session to be disconnected and reconnected over time.

Screen ouput detailing the progress of submission of interporscan jobs
was redirected to a temporary output file named interproscan_submission.log .

```bash
	screen -a
	ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/interproscan
	for Genes in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
	echo $Genes
	$ProgDir/sub_interproscan.sh $Genes
	done 2>&1 | tee -a interproscan_submisison.log
```

Following interproscan annotation split files were combined using the following
commands:

```bash
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/interproscan
for Proteins in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
Strain=$(echo $Proteins | rev | cut -d '/' -f3 | rev)
Organism=$(echo $Proteins | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
echo $Strain
InterProRaw=gene_pred/interproscan/$Organism/$Strain/raw
$ProgDir/append_interpro.sh $Proteins $InterProRaw
done
```


## B) SwissProt


```bash
for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
OutDir=gene_pred/swissprot/$Organism/$Strain
SwissDbDir=../../../../home/groups/harrisonlab/uniprot/swissprot
SwissDbName=uniprot_sprot
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/swissprot
qsub $ProgDir/sub_swissprot.sh $Proteome $OutDir $SwissDbDir $SwissDbName
done
```

## Small secreted proteins

Putative effectors identified within Augustus gene models using a number
of approaches:

 * A) From Braker gene models - Signal peptide & small cystein rich protein
 <!-- * B) From Augustus gene models - Hmm evidence of WY domains  
 * C) From Augustus gene models - Hmm evidence of RxLR effectors
 * D) From Augustus gene models - Hmm evidence of CRN effectors  
 * E) From ORF fragments - Signal peptide & RxLR motif  
 * F) From ORF fragments - Hmm evidence of WY domains  
 * G) From ORF fragments - Hmm evidence of RxLR effectors -->


### A) From Augustus gene models - Identifying secreted proteins

Required programs:
 * SigP
 * biopython
 * TMHMM


Proteins that were predicted to contain signal peptides were identified using
the following commands:

```bash
for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
SplitfileDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/signal_peptides
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/signal_peptides
Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
SplitDir=gene_pred/braker_split/$Organism/$Strain
mkdir -p $SplitDir
BaseName="$Organism""_$Strain"_braker_preds
$SplitfileDir/splitfile_500.py --inp_fasta $Proteome --out_dir $SplitDir --out_base $BaseName
for File in $(ls $SplitDir/*_braker_preds_*); do
Jobs=$(qstat | grep 'pred_sigP' | wc -l)
while [ $Jobs -gt '20' ]; do
sleep 10
printf "."
Jobs=$(qstat | grep 'pred_sigP' | wc -l)
done
printf "\n"
echo $File
qsub $ProgDir/pred_sigP.sh $File signalp-4.1
done
done
```

The batch files of predicted secreted proteins needed to be combined into a
single file for each strain. This was done with the following commands:
```bash
  for SplitDir in $(ls -d gene_pred/braker_split/*/* | grep '650'); do
    Strain=$(echo $SplitDir | cut -d '/' -f4)
    Organism=$(echo $SplitDir | cut -d '/' -f3)
    InStringAA=''
    InStringNeg=''
    InStringTab=''
    InStringTxt=''
    SigpDir=braker_signalp-4.1
    echo "$Organism - $Strain"
    for GRP in $(ls -l $SplitDir/*_braker_preds_*.fa | rev | cut -d '_' -f1 | rev | sort -n); do  
      InStringAA="$InStringAA gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_braker_preds_$GRP""_sp.aa";  
      InStringNeg="$InStringNeg gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_braker_preds_$GRP""_sp_neg.aa";  
      InStringTab="$InStringTab gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_braker_preds_$GRP""_sp.tab";
      InStringTxt="$InStringTxt gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_braker_preds_$GRP""_sp.txt";  
    done
    cat $InStringAA > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_aug_sp.aa
    cat $InStringNeg > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_aug_neg_sp.aa
    tail -n +2 -q $InStringTab > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_aug_sp.tab
    cat $InStringTxt > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_aug_sp.txt
  done
```

Some proteins that are incorporated into the cell membrane require secretion.
Therefore proteins with a transmembrane domain are not likely to represent
cytoplasmic or apoplastic effectors.

Proteins containing a transmembrane domain were identified:

```bash
  for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/transmembrane_helices
    qsub $ProgDir/submit_TMHMM.sh $Proteome
  done
```

Those proteins with transmembrane domains were removed from lists of Signal
peptide containing proteins

```bash
for File in $(ls gene_pred/trans_mem/*/*/*_TM_genes_neg.txt); do
Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
# echo "$Organism - $Strain"
NonTmHeaders=$(echo "$File" | sed 's/neg.txt/neg_headers.txt/g')
cat $File | cut -f1 > $NonTmHeaders
SigP=$(ls gene_pred/braker_signalp-4.1/$Organism/$Strain/"$Strain"_aug_sp.aa)
OutDir=$(dirname $SigP)
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/ORF_finder
$ProgDir/extract_from_fasta.py --fasta $SigP --headers $NonTmHeaders > $OutDir/"$Strain"_final_sp_no_trans_mem.aa
# echo "Number of SigP proteins:"
TotalProts=$(cat $SigP | grep '>' | wc -l)
# echo "Number without transmembrane domains:"
SecProt=$(cat $OutDir/"$Strain"_final_sp_no_trans_mem.aa | grep '>' | wc -l)
# echo "Number of gene models:"
SecGene=$(cat $OutDir/"$Strain"_final_sp_no_trans_mem.aa | grep '>' | cut -f1 -d't' | sort | uniq |wc -l)
# A text file was also made containing headers of proteins testing +ve
PosFile=$(ls gene_pred/trans_mem/$Organism/$Strain/"$Strain"_TM_genes_pos.txt)
TmHeaders=$(echo $PosFile | sed 's/.txt/_headers.txt/g')
cat $PosFile | cut -f1 > $TmHeaders
printf "$Organism\t$Strain\t$TotalProts\t$SecProt\t$SecGene\n"
done
```

```
A.alternata_ssp_tenuissima	1166	1511	1253	1251
A.gaisen	650	1461	1210	1208
```

### C) From Augustus gene models - Effector identification using EffectorP

Required programs:
 * EffectorP.py

```bash
  for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    BaseName="$Organism"_"$Strain"_EffectorP
    OutDir=analysis/effectorP/$Organism/$Strain
    ProgDir=~/git_repos/emr_repos/tools/seq_tools/feature_annotation/fungal_effectors
    qsub $ProgDir/pred_effectorP.sh $Proteome $BaseName $OutDir
  done
```

Those genes that were predicted as secreted and tested positive by effectorP
were identified:

Note - this doesnt exclude proteins with TM domains or GPI anchors

```bash
  for File in $(ls analysis/effectorP/*/*/*_EffectorP.txt); do
    Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
    echo "$Organism - $Strain"
    Headers=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_headers.txt/g')
    cat $File | grep 'Effector' | grep -v 'Effector probability:' | cut -f1 > $Headers
    printf "EffectorP headers:\t"
    cat $Headers | wc -l
    Secretome=$(ls gene_pred/braker_signalp-4.1/$Organism/$Strain/"$Strain"_final_sp_no_trans_mem.aa)
    OutFile=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted.aa/g')
    ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_from_fasta.py --fasta $Secretome --headers $Headers > $OutFile
    OutFileHeaders=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted_headers.txt/g')
    cat $OutFile | grep '>' | tr -d '>' > $OutFileHeaders
    printf "Secreted effectorP headers:\t"
    cat $OutFileHeaders | wc -l
    Gff=$(ls gene_pred/final/$Organism/$Strain/*/final_genes_appended_renamed.gff3)
    EffectorP_Gff=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted.gff/g')
    ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_gff_for_sigP_hits.pl $OutFileHeaders $Gff effectorP ID > $EffectorP_Gff
  done
```

```
A.alternata_ssp_tenuissima - 1166
EffectorP headers:	1970
Secreted effectorP headers:	248
A.gaisen - 650
EffectorP headers:	1941
Secreted effectorP headers:	246
```

## SSCP

Small secreted cysteine rich proteins were identified within secretomes. These
proteins may be identified by EffectorP, but this approach allows direct control
over what constitutes a SSCP.

```bash
for Secretome in $(ls gene_pred/braker_signalp-4.1/*/*/*_final_sp_no_trans_mem.aa | grep '650'); do
Strain=$(echo $Secretome| rev | cut -f2 -d '/' | rev)
Organism=$(echo $Secretome | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
OutDir=analysis/sscp/$Organism/$Strain
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/sscp
$ProgDir/sscp_filter.py --inp_fasta $Secretome --max_length 300 --threshold 3 --out_fasta $OutDir/"$Strain"_sscp_all_results.fa
cat $OutDir/"$Strain"_sscp_all_results.fa | grep 'Yes' > $OutDir/"$Strain"_sscp.fa
printf "number of SSC-rich genes:\t"
cat $OutDir/"$Strain"_sscp.fa | grep '>' | tr -d '>' | cut -f1 -d '.' | sort | uniq | wc -l
done
```

```
A.alternata_ssp_tenuissima - 1166
% cysteine content threshold set to:	3
maximum length set to:	300
No. short-cysteine rich proteins in input fasta:	202
number of SSC-rich genes:	202
A.gaisen - 650
% cysteine content threshold set to:	3
maximum length set to:	300
No. short-cysteine rich proteins in input fasta:	192
number of SSC-rich genes:	192
```

## CAZY proteins

Carbohydrte active enzymes were idnetified using CAZYfollowing recomendations
at http://csbl.bmb.uga.edu/dbCAN/download/readme.txt :

```bash
for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep -e '1166' -e '650' | grep '650'); do
Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
OutDir=gene_pred/CAZY/$Organism/$Strain
mkdir -p $OutDir
Prefix="$Strain"_CAZY
CazyHmm=../../../../home/groups/harrisonlab/dbCAN/dbCAN-fam-HMMs.txt
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/HMMER
qsub $ProgDir/sub_hmmscan.sh $CazyHmm $Proteome $Prefix $OutDir
done
```

The Hmm parser was used to filter hits by an E-value of E1x10-5 or E1x10-e3 if they had a hit over a length of X %.

Those proteins with a signal peptide were extracted from the list and gff files
representing these proteins made.

```bash
for File in $(ls gene_pred/CAZY/*/*/*CAZY.out.dm); do
Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
OutDir=$(dirname $File)
# echo "$Organism - $Strain"
ProgDir=/home/groups/harrisonlab/dbCAN
$ProgDir/hmmscan-parser.sh $OutDir/"$Strain"_CAZY.out.dm > $OutDir/"$Strain"_CAZY.out.dm.ps
CazyHeaders=$(echo $File | sed 's/.out.dm/_headers.txt/g')
cat $OutDir/"$Strain"_CAZY.out.dm.ps | cut -f3 | sort | uniq > $CazyHeaders
# echo "number of CAZY proteins identified:"
TotalProts=$(cat $CazyHeaders | wc -l)
# Gff=$(ls gene_pred/codingquary/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
Gff=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
CazyGff=$OutDir/"$Strain"_CAZY.gff
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/ORF_finder
$ProgDir/extract_gff_for_sigP_hits.pl $CazyHeaders $Gff CAZyme ID > $CazyGff
# echo "number of CAZY genes identified:"
TotalGenes=$(cat $CazyGff | grep -w 'gene' | wc -l)

SecretedProts=$(ls gene_pred/braker_signalp-4.1/$Organism/$Strain/"$Strain"_final_sp_no_trans_mem.aa)
SecretedHeaders=$(echo $SecretedProts | sed 's/.aa/_headers.txt/g')
cat $SecretedProts | grep '>' | tr -d '>' > $SecretedHeaders
CazyGffSecreted=$OutDir/"$Strain"_CAZY_secreted.gff
$ProgDir/extract_gff_for_sigP_hits.pl $SecretedHeaders $CazyGff Secreted_CAZyme ID > $CazyGffSecreted
# echo "number of Secreted CAZY proteins identified:"
cat $CazyGffSecreted | grep -w 'mRNA' | cut -f9 | tr -d 'ID=' | cut -f1 -d ';' > $OutDir/"$Strain"_CAZY_secreted_headers.txt
SecProts=$(cat $OutDir/"$Strain"_CAZY_secreted_headers.txt | wc -l)
# echo "number of Secreted CAZY genes identified:"
SecGenes=$(cat $CazyGffSecreted | grep -w 'gene' | wc -l)
# cat $OutDir/"$Strain"_CAZY_secreted_headers.txt | cut -f1 -d '.' | sort | uniq | wc -l
printf "$Organism\t$Strain\t$TotalProts\t$TotalGenes\t$SecProts\t$SecGenes\n"
done
```

```
A.alternata_ssp_tenuissima	1166	777	777	389	389
A.gaisen	650	781	781	383	383
```

Note - the CAZY genes identified may need further filtering based on e value and
cuttoff length - see below:

Cols in yourfile.out.dm.ps:
1. Family HMM
2. HMM length
3. Query ID
4. Query length
5. E-value (how similar to the family HMM)
6. HMM start
7. HMM end
8. Query start
9. Query end
10. Coverage

* For fungi, use E-value < 1e-17 and coverage > 0.45

* The best threshold varies for different CAZyme classes (please see http://www.ncbi.nlm.nih.gov/pmc/articles/PMC4132414/ for details). Basically to annotate GH proteins, one should use a very relax coverage cutoff or the sensitivity will be low (Supplementary Tables S4 and S9); (ii) to annotate CE families a very stringent E-value cutoff and coverage cutoff should be used; otherwise the precision will be very low due to a very high false positive rate (Supplementary Tables S5 and S10)


### Summary of CAZY families by organism


```bash
for CAZY in $(ls gene_pred/CAZY/*/*/*_CAZY.out.dm.ps | grep '650'); do
Strain=$(echo $CAZY | rev | cut -f2 -d '/' | rev)
Organism=$(echo $CAZY | rev | cut -f3 -d '/' | rev)
OutDir=$(dirname $CAZY)
echo "$Organism - $Strain"
Secreted=$(ls gene_pred/braker_signalp-4.1/$Organism/$Strain/*_final_sp_no_trans_mem_headers.txt)
Gff=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/CAZY
$ProgDir/summarise_CAZY.py --cazy $CAZY --inp_secreted $Secreted --inp_gff $Gff --summarise_family --trim_gene_id 2 --kubicek_2014
done
```

```
A.alternata_ssp_tenuissima - 1166
B-Galactosidases - 4
A-Galactosidases - 3
Polygalacturonase - 8
A-Arabinosidases - 14
Xylanases - 14
Polygalacturonate lyases - 21
B-Glucuronidases - 2
B-Glycosidases - 12
Cellulases - 30
Xyloglucanases - 1
other - 279


A.gaisen - 650
B-Galactosidases - 4
A-Galactosidases - 2
Polygalacturonase - 12
A-Arabinosidases - 14
Xylanases - 14
Polygalacturonate lyases - 21
B-Glucuronidases - 4
B-Glycosidases - 11
Cellulases - 30
Xyloglucanases - 1
other - 269
```

## D) Secondary metabolites (Antismash and SMURF)

Antismash was run to identify clusters of secondary metabolite genes within
the genome. Antismash was run using the weserver at:
http://antismash.secondarymetabolites.org


Results of web-annotation of gene clusters within the assembly were downloaded to
the following directories:

```bash
  for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    OutDir=gene_pred/secondary_metabolites/antismash/$Organism/$Strain
    mkdir -p $OutDir
  done
```

```bash
  for Zip in $(ls gene_pred/secondary_metabolites/antismash/*/*/*.zip | grep '650'); do
    OutDir=$(dirname $Zip)
    unzip -d $OutDir $Zip
  done
```

```bash
for AntiSmash in $(ls gene_pred/secondary_metabolites/antismash/*/*/*/*.final.gbk | grep '650'); do
    Organism=$(echo $AntiSmash | rev | cut -f4 -d '/' | rev)
    Strain=$(echo $AntiSmash | rev | cut -f3 -d '/' | rev)
    echo "$Organism - $Strain"
    OutDir=gene_pred/secondary_metabolites/antismash/$Organism/$Strain
    Prefix=$OutDir/${Strain}_antismash
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/secondary_metabolites
    $ProgDir/antismash2gff.py --inp_antismash $AntiSmash --out_prefix $Prefix

    # Identify secondary metabolites within predicted clusters
    printf "Number of secondary metabolite detected:\t"
    cat "$Prefix"_secmet_clusters.gff | wc -l
    GeneGff=gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3
    bedtools intersect -u -a $GeneGff -b "$Prefix"_secmet_clusters.gff > "$Prefix"_secmet_genes.gff
    cat "$Prefix"_secmet_genes.gff | grep -w 'mRNA' | cut -f9 | cut -f2 -d '=' | cut -f1 -d ';' > "$Prefix"_antismash_secmet_genes.txt
    bedtools intersect -wo -a $GeneGff -b "$Prefix"_secmet_clusters.gff | grep 'mRNA' | cut -f9,10,12,18 | sed "s/ID=//g" | perl -p -i -e "s/;Parent=g\w+//g" | perl -p -i -e "s/;Notes=.*//g" > "$Prefix"_secmet_genes.tsv
    printf "Number of predicted proteins in secondary metabolite clusters:\t"
    cat "$Prefix"_secmet_genes.tsv | wc -l
    printf "Number of predicted genes in secondary metabolite clusters:\t"
    cat "$Prefix"_secmet_genes.gff | grep -w 'gene' | wc -l

      # Identify cluster finder additional non-secondary metabolite clusters
      printf "Number of cluster finder non-SecMet clusters detected:\t"
      cat "$Prefix"_clusterfinder_clusters.gff | wc -l
      GeneGff=gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3
      bedtools intersect -u -a $GeneGff -b "$Prefix"_clusterfinder_clusters.gff > "$Prefix"_clusterfinder_genes.gff
      cat "$Prefix"_clusterfinder_genes.gff | grep -w 'mRNA' | cut -f9 | cut -f2 -d '=' | cut -f1 -d ';' > "$Prefix"_clusterfinder_genes.txt

      printf "Number of predicted proteins in cluster finder non-SecMet clusters:\t"
      cat "$Prefix"_clusterfinder_genes.txt | wc -l
      printf "Number of predicted genes in cluster finder non-SecMet clusters:\t"
      cat "$Prefix"_clusterfinder_genes.gff | grep -w 'gene' | wc -l
  done
```


These clusters represented the following genes. Note that these numbers just
show the number of intersected genes with gff clusters and are not confirmed by
function

```
A.alternata_ssp_tenuissima - 1166
Number of secondary metabolite detected:	34
Number of predicted proteins in secondary metabolite clusters:	946
Number of predicted genes in secondary metabolite clusters:	888
Number of cluster finder non-SecMet clusters detected:	102
Number of predicted proteins in cluster finder non-SecMet clusters:	2280
Number of predicted genes in cluster finder non-SecMet clusters:	2266
A.gaisen - 650
Number of secondary metabolite detected:	30
Number of predicted proteins in secondary metabolite clusters:	743
Number of predicted genes in secondary metabolite clusters:	741
Number of cluster finder non-SecMet clusters detected:	97
Number of predicted proteins in cluster finder non-SecMet clusters:	2250
Number of predicted genes in cluster finder non-SecMet clusters:	2243
```

SMURF was also run to identify secondary metabolite gene clusters.
<!--
Genes needed to be parsed into a specific tsv format prior to submission on the
SMURF webserver.

```bash
  for Gff in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.gff3); do
    Strain=$(echo $Gff | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Gff | rev | cut -f4 -d '/' | rev)
    OutDir=analysis/secondary_metabolites/smurf/$Organism/$Strain
    mkdir -p $OutDir
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/secondary_metabolites
    $ProgDir/gff2smurf.py --gff $Gff > $OutDir/"$Strain"_genes_smurf.tsv
  done
```

SMURF output was received by email and downloaded to the cluster in the output
directory above.

Output files were parsed into gff format:

```bash
  for OutDir in $(ls -d analysis/secondary_metabolites/smurf/*/*); do
    Strain=$(echo $OutDir | rev | cut -f1 -d '/' | rev)
    Organism=$(echo $OutDir | rev | cut -f2 -d '/' | rev)
    GeneGff=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
    SmurfClusters=$(ls $OutDir/Secondary-Metabolite-Clusters.txt)
    SmurfBackbone=$(ls $OutDir/Backbone-genes.txt)
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/secondary_metabolites
    $ProgDir/smurf2gff.py --smurf_clusters $SmurfClusters --smurf_backbone $SmurfBackbone > $OutDir/Smurf_clusters.gff
   bedtools intersect -wo -a $GeneGff -b $OutDir/Smurf_clusters.gff | grep 'mRNA' | cut -f9,10,12,18 | sed "s/ID=//g" | perl -p -i -e "s/;Parent=g\w+//g" | perl -p -i -e "s/;Notes=.*//g" > $OutDir/"$Strain"_smurf_secmet_genes.tsv
  done
``` -->

# Genes with transcription factor annotations:


A list of PFAM domains, superfamily annotations used as part of the DBD database
and a further set of interproscan annotations listed by Shelest et al 2017 were made
http://www.transcriptionfactor.org/index.cgi?Domain+domain:all
https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5415576/

```bash
for Interpro in $(ls gene_pred/interproscan/*/*/*_interproscan.tsv | grep '650'); do
Organism=$(echo $Interpro | rev | cut -f3 -d '/' | rev)
Strain=$(echo $Interpro | rev | cut -f2 -d '/' | rev)
# echo "$Organism - $Strain"
OutDir=analysis/transcription_factors/$Organism/$Strain
mkdir -p $OutDir
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/transcription_factors
$ProgDir/interpro2TFs.py --InterPro $Interpro > $OutDir/"$Strain"_TF_domains.tsv
# echo "total number of transcription factors"
cat $OutDir/"$Strain"_TF_domains.tsv | cut -f1 | sort | uniq > $OutDir/"$Strain"_TF_gene_headers.txt
NumTF=$(cat $OutDir/"$Strain"_TF_gene_headers.txt | wc -l)
printf "$Organism\t$Strain\t$NumTF\n"
done
```

```
A.alternata_ssp_tenuissima      1166    690
A.gaisen        650     651
```


# Genomic analysis
The first analysis was based upon BLAST searches for genes known to be involved in toxin production


## Genes with homology to PHIbase
Predicted gene models were searched against the PHIbase database using tBLASTx.

```bash
qlogin -pe smp 12
cd /data/scratch/armita/alternaria
dbFasta=$(ls /home/groups/harrisonlab/phibase/v4.4/phi_accessions.fa)
dbType="prot"
for QueryFasta in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.cds.fasta | grep '650'); do
Organism=$(echo $QueryFasta | rev | cut -f4 -d '/' | rev)
Strain=$(echo $QueryFasta | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
Prefix="${Strain}_phi_accessions"
Eval="1e-30"
OutDir=analysis/blast_homology/$Organism/$Strain
mkdir -p $OutDir
makeblastdb -in $dbFasta -input_type fasta -dbtype $dbType -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
blastx -num_threads 6 -db $OutDir/$Prefix.db -query $QueryFasta -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
cat $OutDir/${Prefix}_hits.txt | grep 'effector' | cut -f1,2 | sort | uniq > $OutDir/${Prefix}_hits_headers.txt
done
```

## Presence and genes with homology to Alternaria toxins

The first analysis was based upon BLAST searches for genes known to be involved in toxin production


```bash
qlogin -pe smp 4
cd /data/scratch/armita/alternaria
dbFasta=$(ls /home/groups/harrisonlab/project_files/alternaria/analysis/blast_homology/CDC_genes/A.alternata_CDC_genes.fa)
dbType="nucl"
for QueryFasta in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.cds.fasta); do
Organism=$(echo $QueryFasta | rev | cut -f4 -d '/' | rev)
Strain=$(echo $QueryFasta | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
Prefix="${Strain}_CDC_genes"
Eval="1e-100"
OutDir=analysis/blast_homology/$Organism/$Strain
mkdir -p $OutDir
makeblastdb -in $dbFasta -input_type fasta -dbtype $dbType -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
tblastx -num_threads 24 -db $OutDir/$Prefix.db -query $QueryFasta -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
cat $OutDir/${Prefix}_hits.txt | cut -f1,2 | sort | uniq > $OutDir/${Prefix}_hits_headers.txt
# cat $OutDir/${Prefix}_hits.txt | grep 'effector' | cut -f1,2 | sort | uniq > $OutDir/${Prefix}_hits_headers.txt
done
```


Blast searches were also performed in comparison to the genome sequence

```bash
  for Subject in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    Query=../../../../home/groups/harrisonlab/project_files/alternaria/analysis/blast_homology/CDC_genes/A.alternata_CDC_genes.fa
    qsub $ProgDir/blast_pipe.sh $Query dna $Subject
  done
```

```bash
  HitsList=""
  StrainList=""
  for BlastHits in $(ls analysis/blast_homology/*/*/*_A.alternata_CDC_genes.fa_homologs.csv); do
    sed -i "s/\t\t/\t/g" $BlastHits
    sed -i "s/Grp1\t//g" $BlastHits
    Strain=$(echo $BlastHits | rev | cut -f2 -d '/' | rev)
    HitsList="$HitsList $BlastHits"
    StrainList="$StrainList $Strain"
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    $ProgDir/blast_parse.py --blast_csv $HitsList --headers $StrainList --identity 0.7 --evalue 1e-30
  done
```

```
Query ID's	1166	650
AB015351_-_AKT1_gene	0	2
AB015352_-_AKT2_gene	0	1
AB034586_-_ACTT1_gene	0	1
AB035491_-_AKTR_gene	0	2
AB035492_-_AKT3_gene	0	2
AB070711_-_AFT1-1_gene	0	2
AB070712_-_AFTR-1_gene	0	2
AB070713_-_AFT3-1_gene	0	2
AB119280_-_AFTS1_gene	1	0
AB176941_ACTT3_gene	0	2
AB176941_ACTTR_gene	0	2
AB179766_-_AFT3-2_gene	0	2
AB179766_-_AFT9-1_gene	0	2
AB179766_-_AFT10-1_gene	0	2
AB179766_-_AFT11-1_gene	0	2
AB179766_-_AFT12-1_gene	0	2
AB179766_-_AFTR-2_gene	0	2
AB432914_ACTT2_gene	0	1
AB444613_ACTT5_gene	0	1
AB444614_ACTT6_gene	0	2
AB465676_-_ALT1_gene	0	0
AB525198_-_AMT1_gene	2	0
AB525198_-_AMT2_gene	3	1
AB525198_-_AMT3_gene	2	0
AB525198_-_AMT4_gene	2	0
AB525198_-_AMT5_gene	2	0
AB525198_-_AMT6_gene	2	0
AB525198_-_AMT7_gene	2	0
AB525198_-_AMT8_gene	2	0
AB525198_-_AMT9_gene	2	0
AB525198_-_AMT10_gene	1	0
AB525198_-_AMT11_gene	0	0
AB525198_-_AMT12_gene	2	0
AB525198_-_AMT13_gene	1	0
AB525198_-_AMT14_gene	1	1
AB525198_-_AMT15_gene	0	0
AB525198_-_AMT16_gene	3	2
AB525198_-_AMTR1_gene	2	2
AB688098_-_ACRTS1_gene	0	0
AB725683_-_ACRTS2_gene	0	0
```

Once blast searches had completed, the BLAST hits were converted to GFF
annotations:

```bash
  for BlastHits in $(ls analysis/blast_homology/*/*/*_A.alternata_CDC_genes.fa_homologs.csv); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    Strain=$(echo $BlastHits | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $BlastHits | rev | cut -f3 -d '/' | rev)
    HitsGff=analysis/blast_homology/$Organism/$Strain/"$Strain"_A.alternata_CDC_genes.fa_homologs.gff
    Column2=toxin_homolog
    NumHits=2
    $ProgDir/blast2gff.pl $Column2 $NumHits $BlastHits > $HitsGff
    cat $HitsGff | grep 'AKT' > analysis/blast_homology/$Organism/$Strain/"$Strain"_A.alternata_CDC_genes.fa_homologs_AKT.gff
    cat $HitsGff | grep 'AMT' > analysis/blast_homology/$Organism/$Strain/"$Strain"_A.alternata_CDC_genes.fa_homologs_AMT.gff
  done
```


Extracted gff files of the BLAST hit locations were intersected with gene models
to identify if genes were predicted for these homologs:

```bash
for HitsGff in $(ls analysis/blast_homology/*/*/*_A.alternata_CDC_genes.fa_homologs.gff | grep '1166'); do
Strain=$(echo $HitsGff | rev | cut -f2 -d '/' | rev)
Organism=$(echo $HitsGff | rev | cut -f3 -d '/' | rev)
Proteins=$(ls gene_pred/final/$Organism/$Strain/*/final_genes_appended_renamed.gff3)
OutDir=analysis/blast_homology/$Organism/$Strain/"$Strain"_A.alternata_CDC_genes
IntersectBlast=$OutDir/"$Strain"_A.alternata_CDC_genes_Intersect.gff
NoIntersectBlast=$OutDir/"$Strain"_A.alternata_CDC_genes_NoIntersect.gff
mkdir -p $OutDir
echo "$Organism - $Strain"
echo "The number of BLAST hits in the gff file were:"
cat $HitsGff | wc -l
echo "The number of blast hits intersected were:"
bedtools intersect -wao -a $HitsGff -b $Proteins > $IntersectBlast
cat $IntersectBlast | grep -w 'gene' | wc -l
echo "The number of blast hits not intersecting gene models were:"
bedtools intersect -v -a $HitsGff -b $Proteins > $NoIntersectBlast
# rm $NoIntersectBlast
cat $IntersectBlast | grep -w -E '0$' | wc -l
done
```

```
A.alternata_ssp_tenuissima - 1166
The number of BLAST hits in the gff file were:
34
The number of blast hits intersected were:
32
The number of blast hits not intersecting gene models were:
8
A.gaisen - 650
The number of BLAST hits in the gff file were:
37
The number of blast hits intersected were:
33
The number of blast hits not intersecting gene models were:
10
```


## Presence and genes with homology to Nc clock genes

The analysis was based upon BLAST searches for genes known to be involved in N. crassa clock function



```bash
  cp -s /home/groups/harrisonlab/project_files/alternaria/analysis/blast_homology/Nc_clock_genes.fasta analysis/blast_homology/Nc_clock_genes.fasta
  for Subject in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    Query=analysis/blast_homology/Nc_clock_genes.fasta
    qsub $ProgDir/blast_pipe.sh $Query dna $Subject
  done
```

<!-- ```bash
  HitsList=""
  StrainList=""
  for BlastHits in $(ls analysis/blast_homology/*/*/*_Nc_clock_genes.fasta_homologs.csv); do
    sed -i "s/\t\t/\t/g" $BlastHits
    sed -i "s/Grp1\t//g" $BlastHits
    Strain=$(echo $BlastHits | rev | cut -f2 -d '/' | rev)
    HitsList="$HitsList $BlastHits"
    StrainList="$StrainList $Strain"
  done
  ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
  $ProgDir/blast_parse.py --blast_csv $HitsList --headers $StrainList --identity 0.7 --evalue 1e-30
``` -->


Once blast searches had completed, the BLAST hits were converted to GFF
annotations:

```bash
  for BlastHits in $(ls analysis/blast_homology/*/*/*_Nc_clock_genes.fasta_homologs.csv); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    Strain=$(echo $BlastHits | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $BlastHits | rev | cut -f3 -d '/' | rev)
    HitsGff=analysis/blast_homology/$Organism/$Strain/"$Strain"_Nc_clock_gene_homologs.gff
    Column2=clock_homolog
    NumHits=1
    $ProgDir/blast2gff.pl $Column2 $NumHits $BlastHits > $HitsGff
  done
```


Extracted gff files of the BLAST hit locations were intersected with gene models
to identify if genes were predicted for these homologs:

```bash
for HitsGff in $(ls analysis/blast_homology/*/*/*_Nc_clock_gene_homologs.gff); do
Strain=$(echo $HitsGff | rev | cut -f2 -d '/' | rev)
Organism=$(echo $HitsGff | rev | cut -f3 -d '/' | rev)
Proteins=$(ls gene_pred/final/$Organism/$Strain/*/final_genes_appended_renamed.gff3)
OutDir=analysis/blast_homology/$Organism/$Strain/"$Strain"_Nc_clock_gene
IntersectBlast=$OutDir/"$Strain"_Nc_clock_gene_Intersect.gff
NoIntersectBlast=$OutDir/"$Strain"_Nc_clock_gene_NoIntersect.gff
mkdir -p $OutDir
echo "$Organism - $Strain"
echo "The number of BLAST hits in the gff file were:"
cat $HitsGff | wc -l
echo "The number of blast hits intersected were:"
bedtools intersect -wao -a $HitsGff -b $Proteins > $IntersectBlast
cat $IntersectBlast | grep -w 'gene' | wc -l
echo "The number of blast hits not intersecting gene models were:"
bedtools intersect -v -a $HitsGff -b $Proteins > $NoIntersectBlast
# rm $NoIntersectBlast
cat $IntersectBlast | grep -w -E '0$' | wc -l
# Gene models cds and gene sequences were extracted
cat $IntersectBlast | grep -w 'mRNA' | cut -f18 | sed 's/ID=//g' | cut -f1 -d ';' > $OutDir/"$Strain"_Nc_clock_gene_Intersect_genes.txt
cat $OutDir/"$Strain"_Nc_clock_gene_Intersect_genes.txt | sed "s/\.t.*//g" > $OutDir/"$Strain"_Nc_clock_gene_Intersect_genes_mod.txt
Cds=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.cds.fasta)
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/ORF_finder
$ProgDir/extract_from_fasta.py --fasta $Cds --headers $OutDir/"$Strain"_Nc_clock_gene_Intersect_genes.txt > $OutDir/"$Strain"_Nc_clock_gene_Intersect_cds.fa
Gene=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gene.fasta)
$ProgDir/extract_from_fasta.py --fasta $Gene --headers $OutDir/"$Strain"_Nc_clock_gene_Intersect_genes_mod.txt > $OutDir/"$Strain"_Nc_clock_gene_Intersect_gene.fa
done
```

# Build Annotation Tables


```bash
for GeneGff in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.gff3 | grep -e '1166' -e '650' | grep '650'); do
  Strain=$(echo $GeneGff | rev | cut -f3 -d '/' | rev)
  Organism=$(echo $GeneGff | rev | cut -f4 -d '/' | rev)
  Assembly=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_unmasked.fa)
  Antismash=$(ls gene_pred/secondary_metabolites/antismash/$Organism/$Strain/*_antismash_secmet_genes.tsv)
  # Smurf=$(ls analysis/secondary_metabolites/smurf/$Organism/$Strain/*_smurf_secmet_genes.tsv)
  TFs=$(ls analysis/transcription_factors/$Organism/$Strain/"$Strain"_TF_domains.tsv)
  SigP=$(ls gene_pred/braker_signalp-4.1/$Organism/$Strain/"$Strain"_aug_sp.aa)
  TM_out=$(ls gene_pred/trans_mem/$Organism/$Strain/"$Strain"_TM_genes_pos.txt)
  # GPI_out=$(ls gene_pred/trans_mem/$Organism/$Strain/GPIsom/GPI_pos.fa)
  EffP_list=$(ls analysis/effectorP/$Organism/$Strain/"$Organism"_"$Strain"_EffectorP_headers.txt)
  CAZY_list=$(ls gene_pred/CAZY/$Organism/$Strain/"$Strain"_CAZY.out.dm.ps)
  PhiHits=$(ls analysis/blast_homology/$Organism/$Strain/"$Strain"_phi_accessions_hits_headers.txt)
  ToxinHits=$(ls analysis/blast_homology/$Organism/$Strain/"$Strain"_CDC_genes_hits_headers.txt)
  InterPro=$(ls gene_pred/interproscan/$Organism/$Strain/*_interproscan.tsv)
  SwissProt=$(ls gene_pred/swissprot/$Organism/$Strain/swissprot_vMar2018_tophit_parsed.tbl)
  # Orthology=$(ls analysis/orthology/orthomcl/At_Aa_Ag_all_isolates/formatted/Results_Apr10/Orthogroups.txt)
  Orthology=$(ls analysis/orthology/orthomcl/At_Aa_Ag_all_isolates/formatted/Results_May31/Orthogroups.txt)
  if [[ $Strain == '648' ]]; then
    OrthoStrainID='At_1'
    echo $OrthoStrainID
  elif [[ $Strain == '1082' ]]; then
    OrthoStrainID='At_2'
    echo $OrthoStrainID
  elif [[ $Strain == '1164' ]]; then
    OrthoStrainID='At_3'
    echo $OrthoStrainID
  elif [[ $Strain == '24350' ]]; then
    OrthoStrainID='At_4'
    echo $OrthoStrainID
  elif [[ $Strain == '635' ]]; then
    OrthoStrainID='At_5'
    echo $OrthoStrainID
  elif [[ $Strain == '743' ]]; then
    OrthoStrainID='At_6'
    echo $OrthoStrainID
  elif [[ $Strain == '1166' ]]; then
    OrthoStrainID='At_7'
    echo $OrthoStrainID
  elif [[ $Strain == '1177' ]]; then
    OrthoStrainID='At_8'
    echo $OrthoStrainID
  elif [[ $Strain == '675' ]]; then
    OrthoStrainID='Aa_1'
    echo $OrthoStrainID
  elif [[ $Strain == '97.0013' ]]; then
    OrthoStrainID='Aa_2'
    echo $OrthoStrainID
  elif [[ $Strain == '97.0016' ]]; then
    OrthoStrainID='Aa_3'
    echo $OrthoStrainID
  elif [[ $Strain == '650' ]]; then
    OrthoStrainID='Ag_1'
    echo $OrthoStrainID
  fi
  OrthoStrainAll='At_1 At_2 At_3 At_4 At_5 At_6 At_6 At_7 At_8 Aa_1 Aa_2 Aa_3 Ag_1'
  OutDir=gene_pred/annotation/$Organism/$Strain
  mkdir -p $OutDir
  ProgDir=/home/armita/git_repos/emr_repos/scripts/alternaria/pathogen/annotation
  # $ProgDir/build_annot_table_Alt2.py --genes_gff $GeneGff --SigP $SigP --TM_list $TM_out --EffP_list $EffP_list --CAZY_list $CAZY_list --TFs $TFs --PhiHits $PhiHits --ToxinHits $ToxinHits --Antismash $Antismash --Smurf $Smurf --InterPro $InterPro --Swissprot $SwissProt --orthogroups $Orthology --strain_id $OrthoStrainID --OrthoMCL_all $OrthoStrainAll > $OutDir/"$Strain"_annotation_ncbi.tsv
  # SMURF results were exluceded due to them being considered untrustworthy
  $ProgDir/build_annot_table_Alt2.py --genes_gff $GeneGff --SigP $SigP --TM_list $TM_out --EffP_list $EffP_list --CAZY_list $CAZY_list --TFs $TFs --PhiHits $PhiHits --ToxinHits $ToxinHits --Antismash $Antismash --InterPro $InterPro --Swissprot $SwissProt --orthogroups $Orthology --strain_id $OrthoStrainID --OrthoMCL_all $OrthoStrainAll > $OutDir/"$Strain"_annotation_ncbi.tsv
done
```


LS regions of the apple and pear pathotype were investigated:


```bash
# 1166
Isolate='1166'
AnnotTab=$(ls gene_pred/annotation/A.*/$Isolate/${Isolate}_annotation_ncbi.tsv)
GenesCDC=$(echo $AnnotTab | sed 's/_annotation_ncbi.tsv/_CDC_genes.tsv/g')
cat $AnnotTab | grep -e 'contig_14' -e 'contig_15' -e 'contig_18' -e 'contig_19' -e 'contig_20' -e 'contig_21' > $GenesCDC
TotalGenes=$(cat $GenesCDC | wc -l)
Secreted=$(cat $GenesCDC | cut -f1,8 | grep 'Yes' | wc -l)
EffectorP=$(cat $GenesCDC | cut -f1,8,9 | grep "Yes.Yes" | wc -l)
CAZyme=$(cat $GenesCDC | cut -f1,8,10 | grep 'CAZY' | grep 'Yes' | wc -l)
SecMet=$(cat $GenesCDC | grep 'AS_' | wc -l)
printf "$Isolate\t$TotalGenes\t$Secreted\t$EffectorP\t$CAZyme\t$SecMet\n"

# 650
Isolate='650'
AnnotTab=$(ls gene_pred/annotation/A.*/$Isolate/${Isolate}_annotation_ncbi.tsv)
GenesCDC=$(echo $AnnotTab | sed 's/_annotation_ncbi.tsv/_CDC_genes.tsv/g')
cat $AnnotTab | grep -e 'contig_14' -e 'contig_16' -e 'contig_23' -e 'contig_24' > $GenesCDC
TotalGenes=$(cat $GenesCDC | wc -l)
Secreted=$(cat $GenesCDC | cut -f1,8 | grep 'Yes' | wc -l)
EffectorP=$(cat $GenesCDC | cut -f1,8,9 | grep "Yes.Yes" | wc -l)
CAZyme=$(cat $GenesCDC | cut -f1,8,10 | grep 'CAZY' | grep 'Yes' | wc -l)
SecMet=$(cat $GenesCDC | grep 'AS_' | wc -l)
printf "$Isolate\t$TotalGenes\t$Secreted\t$EffectorP\t$CAZyme\t$SecMet\n"
```

```
1166	624	32	12	6	153
650	502	41	13	8	154
```

# Genomic analysis


## Alignment vs minion genomes

Illumina sequence data from each of the isolates was aligned against the minion
assemblies.


### Read alignment

```bash
for Reference in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa | grep '650'); do
RefStrain=$(echo $Reference | rev | cut -f3 -d '/' | rev)
for StrainPath in $(ls -d ../../../../home/groups/harrisonlab/project_files/alternaria/qc_dna/paired/*/* | grep -v 'arborescens'); do
Jobs=$(qstat | grep 'sub_bowtie' | grep 'qw'| wc -l)
while [ $Jobs -gt 1 ]; do
sleep 1m
printf "."
Jobs=$(qstat | grep 'sub_bowtie' | grep 'qw'| wc -l)
done
printf "\n"
echo $StrainPath
Strain=$(echo $StrainPath | rev | cut -f1 -d '/' | rev)
Organism=$(echo $StrainPath | rev | cut -f2 -d '/' | rev)
echo "$Organism - $Strain"
F_Read=$(ls $StrainPath/F/*_trim.fq.gz | head -n1 | tail -n1)
R_Read=$(ls $StrainPath/R/*_trim.fq.gz | head -n1 | tail -n1)
echo $F_Read
echo $R_Read
# OutDir=analysis/genome_alignment/bowtie/$Organism/$Strain/vs_${RefStrain}
OutDir=analysis/genome_alignment/bowtie/$Organism/$Strain/vs_${RefStrain}_v2
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/genome_alignment
qsub $ProgDir/bowtie/sub_bowtie.sh $Reference $F_Read $R_Read $OutDir $Strain
done
done
```

### Read coverage

Identify read coverage over each bp

```bash
  for Bam in $(ls analysis/genome_alignment/bowtie/*/*/vs_*/*_aligned_sorted.bam); do
    Target=$(echo $Bam | rev | cut -f2 -d '/' | rev)
    Strain=$(echo $Bam | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Bam | rev | cut -f4 -d '/' | rev)
    echo "$Organism - $Strain - $Target"
    OutDir=$(dirname $Bam)
    samtools depth -aa $Bam > $OutDir/${Organism}_${Strain}_${Target}_depth.tsv
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/genome_alignment/coverage_analysis
    $ProgDir/cov_by_window.py --cov $OutDir/${Organism}_${Strain}_${Target}_depth.tsv > $OutDir/${Organism}_${Strain}_${Target}_depth_10kb.tsv
    sed -i "s/$/\t$Strain/g" $OutDir/${Organism}_${Strain}_${Target}_depth_10kb.tsv
  done
  for Cov in $(ls analysis/genome_alignment/bowtie/*/*/vs_*/*_vs_*_depth.tsv); do
    echo ${Cov} | cut -f4,5,6 -d '/' --output-delimiter " - "
    cat $Cov | cut -f3 | sort -n | awk ' { a[i++]=$1; } END { print a[int(i/2)]; }'
  done > analysis/genome_alignment/bowtie/read_coverage.txt
  for Strain in 1166 650; do
    for Cov in $(ls ../../../../../../data/scratch/armita/alternaria/analysis/genome_alignment/bowtie/*/${Strain}/vs_${Strain}/*_vs_*_depth.tsv); do
      echo ${Cov} | cut -f14,15,16 -d '/' --output-delimiter " - "
      cat $Cov | cut -f3 | sort -n | awk ' { a[i++]=$1; } END { print a[int(i/2)]; }'
    done
  done >> analysis/genome_alignment/bowtie/read_coverage.txt
  # for Target in "vs_650" "vs_1166"; do
  #   OutDir=analysis/genome_alignment/bowtie/grouped_${Target}
  #   mkdir -p $OutDir
  #   cat analysis/genome_alignment/bowtie/*/*/*/*_*_${Target}_depth_10kb.tsv > $OutDir/${Target}_grouped_depth.tsv
  # done
```

This allowed assessment of read depth against the reference genomes.

The analysis was repeated for each isolate vs its own assembly to give an control value for the isolate:

```bash
for Reference in $(ls repeat_masked/*/*/ncbi_edits_repmask/*_contigs_unmasked.fa | grep -v -e '1166' -e '650'); do
Strain=$(echo $Reference | rev | cut -f3 -d '/'| rev)
StrainPath=$(ls -d qc_dna/paired/*/$Strain)
Organism=$(echo $StrainPath | rev | cut -f2 -d '/' | rev)
F_Read=$(ls $StrainPath/F/*_trim.fq.gz)
R_Read=$(ls $StrainPath/R/*_trim.fq.gz)
for FileF in $(ls qc_rna/paired/*/*/F/*.fastq.gz); do
Jobs=$(qstat | grep 'sub_bwa' | grep 'qw'| wc -l)
# while [ $Jobs -gt 1 ]; do
# sleep 1m
# printf "."
# Jobs=$(qstat | grep 'sub_bwa' | grep 'qw'| wc -l)
# done
echo $F_Read
echo $R_Read
Prefix="${Organism}_${Strain}"
OutDir=analysis/genome_alignment/bowtie/$Organism/$Strain/vs_${Strain}
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/genome_alignment
qsub $ProgDir/bowtie/sub_bowtie.sh $Reference $F_Read $R_Read $OutDir $Strain
done
done
```

```bash
for Reference in $(ls ../../../../../../data/scratch/armita/alternaria/repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
Strain=$(echo $Reference | rev | cut -f3 -d '/'| rev)
Organism=$(echo $Reference | rev | cut -f4 -d '/' | rev)
Reads=$(ls qc_dna/minion/*/$Strain/*_trim.fastq.gz)
Prefix="${Organism}_${Strain}"
OutDir=analysis/genome_alignment/minimap/$Organism/$Strain/vs_${Strain}
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/genome_alignment
qsub $ProgDir/minimap/sub_minimap2.sh $Reference $Reads $OutDir
done
```

```bash
for Sam in $(ls analysis/genome_alignment/minimap/*/*/vs_*/*_aligned_sorted.bam); do
  Target=$(echo $Sam | rev | cut -f2 -d '/' | rev)
  Strain=$(echo $Sam | rev | cut -f3 -d '/' | rev)
  Organism=$(echo $Sam | rev | cut -f4 -d '/' | rev)
  echo "$Organism - $Strain - $Target"
  OutDir=$(dirname $Sam)
  samtools depth -aa $Sam > $OutDir/${Organism}_${Strain}_${Target}_depth.tsv
done

for Strain in 1166 650; do
  for Cov in $(ls analysis/genome_alignment/minimap/*/*/vs_*/*_depth.tsv); do
    echo ${Cov} | cut -f4,5,6 -d '/' --output-delimiter " - "
    cat $Cov | cut -f3 | sort -n | awk ' { a[i++]=$1; } END { print a[int(i/2)]; }'
  done
done > analysis/genome_alignment/minimap/read_coverage.txt
```



### Plot read coverage


```R
library(readr)
setwd("~/Downloads/Aalt/coverage2")

appended_df <- read_delim("~/Downloads/Aalt/coverage2/vs_1166_grouped_depth.tsv", "\t", escape_double = FALSE, col_names = FALSE, col_types = cols(X4 = col_factor(levels = c("675", "97.0013", "97.0016", "650", "648", "24350", "1082", "1164", "635", "743", "1166", "1177"))), trim_ws = TRUE)

myFun <- function(x) {
  c(min = min(x), max = max(x),
    mean = mean(x), median = median(x),
    std = sd(x))
}

colnames(appended_df) <- c("contig","position", "depth", "strain")

appended_df$treatment <- paste(appended_df$strain , appended_df$contig)
tapply(appended_df$depth, appended_df$treatment, myFun)

df2 <- cbind(do.call(rbind, tapply(appended_df$depth, appended_df$treatment, myFun)))
write.csv(df2, '1166_contig_coverage.csv')

appended_df$depth <- ifelse(appended_df$depth > 100, 100, appended_df$depth)

# install.packages("ggplot2")
library(ggplot2)
require(scales)

for (i in 1:22){
contig = paste("contig", i, sep = "_")
p0 <- ggplot(data=appended_df[appended_df$contig == contig, ], aes(x=`position`, y=`depth`, group=1)) +
geom_line() +
labs(x = "Position (bp)", y = "Coverage") +
scale_y_continuous(breaks=seq(0,100,25), limits=c(0,100)) +
facet_wrap(~strain, nrow = 12, ncol = 1, strip.position = "left")
outfile = paste("1166_contig", i, "cov.jpg", sep = "_")
ggsave(outfile , plot = p0, device = 'jpg', path = NULL,
scale = 1, width = 500, height = 500, units = 'mm',
dpi = 150, limitsize = TRUE)
}

```


```R
library(readr)
setwd("~/Downloads/Aalt/coverage")

df_650 <- read_delim("~/Downloads/Aalt/coverage/vs_650_grouped_depth.tsv", "\t", escape_double = FALSE, col_names = FALSE, col_types = cols(X4 = col_factor(levels = c("675", "97.0013", "97.0016", "650", "648", "24350", "1082", "1164", "635", "743", "1166", "1177"))), trim_ws = TRUE)

myFun <- function(x) {
  c(min = min(x), max = max(x),
    mean = mean(x), median = median(x),
    std = sd(x))
}

colnames(df_650) <- c("contig","position", "depth", "strain")

df_650$treatment <- paste(df_650$strain , df_650$contig)
tapply(df_650$depth, df_650$treatment, myFun)

df2 <- cbind(do.call(rbind, tapply(df_650$depth, df_650$treatment, myFun)))
write.csv(df2, '650_contig_coverage.csv')

df_650$depth <- ifelse(df_650$depth > 100, 100, df_650$depth)

# install.packages("ggplot2")
library(ggplot2)
require(scales)

for (i in 1:27){
contig = paste("contig", i, sep = "_")
p0 <- ggplot(data=df_650[df_650$contig == contig, ], aes(x=`position`, y=`depth`, group=1)) +
geom_line() +
labs(x = "Position (bp)", y = "Coverage") +
scale_y_continuous(breaks=seq(0,100,25), limits=c(0,100)) +
facet_wrap(~strain, nrow = 12, ncol = 1, strip.position = "left")
outfile = paste("650_contig", i, "cov.jpg", sep = "_")
ggsave(outfile , plot = p0, device = 'jpg', path = NULL,
scale = 1, width = 500, height = 500, units = 'mm',
dpi = 150, limitsize = TRUE)
}
```

## 4.0 Characterising CDCs

### 4.1.b Repetative content

The % repeatmasking was identified for each contig:

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
  Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
  Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
  OutDir=analysis/CDC_contigs/$Organism/$Strain
  mkdir -p $OutDir
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/repeat_masking
  $ProgDir/count_Ns.py --inp_fasta $Assembly --out_txt $OutDir/N_content.txt
done
```

```
# 1166
contig_1	3902980	0	0.00
contig_2	3781932	0	0.00
contig_3	3000390	0	0.00
contig_4	2851745	0	0.00
contig_5	2693844	0	0.00
contig_6	2583941	0	0.00
contig_7	2502671	0	0.00
contig_8	2455819	0	0.00
contig_9	2451092	0	0.00
contig_10	2402550	0	0.00
contig_11	2194253	0	0.00
contig_12	1303685	0	0.00
contig_13	767820	0	0.00
contig_14	549494	0	0.00
contig_15	435297	0	0.00
contig_16	433285	0	0.00
contig_17	391795	0	0.00
contig_18	368761	0	0.00
contig_19	225742	0	0.00
contig_20	154051	0	0.00
contig_21	143777	0	0.00
contig_22	109256	0	0.00

# 650
contig_1	6257968	0	0.00
contig_2	2925786	0	0.00
contig_3	2776589	0	0.00
contig_4	2321443	0	0.00
contig_5	2116911	0	0.00
contig_6	2110033	0	0.00
contig_7	1975041	0	0.00
contig_8	1822907	0	0.00
contig_9	1811374	0	0.00
contig_10	1696734	0	0.00
contig_11	1617057	0	0.00
contig_12	1416541	0	0.00
contig_13	1110254	0	0.00
contig_14	629968	0	0.00
contig_15	554254	0	0.00
contig_16	547262	0	0.00
contig_17	522351	0	0.00
contig_18	463030	0	0.00
contig_19	353531	0	0.00
contig_20	350965	0	0.00
contig_21	313768	0	0.00
contig_22	288922	0	0.00
contig_23	206183	0	0.00
contig_24	89891	0	0.00
contig_25	25201	0	0.00
contig_26	23394	0	0.00
contig_27	19592	0	0.00
```

### 4.1.c Gene density and % content

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '650'); do
  Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
  Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
  OutDir=analysis/CDC_contigs/$Organism/$Strain
  mkdir -p $OutDir
  Genes=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation
  $ProgDir/feature_density.py --inp_fasta $Assembly --inp_gff $Genes --features gene
done
```
```
# 1166
contig, contig_lgth, feat_count, feat_density, feat_lgth, perc_feat
contig_1	3902980	1563	400.46	2412918	61.82
contig_2	3781932	1413	373.62	2203815	58.27
contig_3	3000390	1154	384.62	1733294	57.77
contig_4	2851745	1098	385.03	1685701	59.11
contig_5	2693844	1055	391.63	1663091	61.74
contig_6	2583941	986	381.59	1540588	59.62
contig_7	2502671	954	381.19	1498182	59.86
contig_8	2455819	960	390.91	1481084	60.31
contig_9	2451092	930	379.42	1359530	55.47
contig_10	2402550	872	362.95	1436796	59.8
contig_11	2194253	818	372.79	1268004	57.79
contig_12	1303685	504	386.6	753202	57.77
contig_13	767820	282	367.27	408936	53.26
contig_14	549494	184	334.85	237154	43.16
contig_15	435297	166	381.35	162962	37.44
contig_16	433285	165	380.81	236067	54.48
contig_17	391795	169	431.35	227914	58.17
contig_18	368761	109	295.58	154186	41.81
contig_19	225742	69	305.66	81190	35.97
contig_20	154051	35	227.2	57127	37.08
contig_21	143777	48	333.85	62379	43.39
contig_22	109256	42	384.42	63072	57.73
# 650
contig_1	6257968	2460	393.1	3775821	60.34
contig_2	2925786	1147	392.03	1689167	57.73
contig_3	2776589	1088	391.85	1682151	60.58
contig_4	2321443	890	383.38	1356784	58.45
contig_5	2116911	836	394.92	1311434	61.95
contig_6	2110033	795	376.77	1229056	58.25
contig_7	1975041	732	370.63	1184613	59.98
contig_8	1822907	722	396.07	1017676	55.83
contig_9	1811374	723	399.14	1076845	59.45
contig_10	1696734	655	386.04	941644	55.5
contig_11	1617057	601	371.66	919168	56.84
contig_12	1416541	544	384.03	845267	59.67
contig_13	1110254	435	391.8	654108	58.92
contig_14	629968	203	322.24	241021	38.26
contig_15	554254	201	362.65	306701	55.34
contig_16	547262	201	367.28	268674	49.09
contig_17	522351	189	361.83	286435	54.84
contig_18	463030	169	364.99	252899	54.62
contig_19	353531	136	384.69	175247	49.57
contig_20	350965	130	370.41	183371	52.25
contig_21	313768	114	363.33	164127	52.31
contig_22	288922	105	363.42	143655	49.72
contig_23	206183	64	310.4	88464	42.91
contig_24	89891	25	278.11	36680	40.8
contig_25	25201	3	119.04	3216	12.76
contig_27	19592	1	51.04	389	1.99
```

## Meiotic Machinery

The predicted proteome for Saccharomyces cerevisiae strain S288C was downloaded from the Saccharomyces Genome Database (www.yeastgenome.org; Cherry et al. (1998)) and from this, 86 genes with a known meiotic function, as identified in the supplementary material of Halary et al. (2011), were extracted. This list included 29 genes identified as core meiotic genes within the eukaryotes (Malik et al., 2008),  and fifteen genes that have functional descriptions as meiosis-specific proteins on the Saccharomyces Genome Database (Cherry et al., 2012).


```bash
qlogin -pe smp 16
cd /data/scratch/armita/alternaria
# QueeryGenes=$(ls analysis/meiotic_machinery/86_meiotic_proteins.fasta)
Prots=$(ls analysis/meiotic_machinery/S288C_reference_genome_R64-2-1_20150113/orf_trans_all_R64-2-1_20150113.fasta)
SaccharomycesProts=analysis/meiotic_machinery/S288C_reference_genome_R64-2-1_20150113/orf_trans_all_R64-2-1_20150113_parsed.fasta
cat $Prots | tr -d '*' | cut -f1,2 -d ' ' | sed 's/ /_/g' > $SaccharomycesProts

QueeryFasta=$(ls analysis/meiotic_machinery/86_meiotic_proteins.fasta)
QueeryHeaders=analysis/meiotic_machinery/86_meiotic_proteins.txt
cat $QueeryFasta | grep '>' | tr -d '>' > $QueeryHeaders

Proteome_1166=$(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep '1166')
Proteome_650=$(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep '650')
Proteome_675=$(ls /home/groups/harrisonlab/project_files/alternaria/gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta | grep '675')
for Proteome in $(ls $Proteome_1166 $Proteome_650 $Proteome_675); do
  Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
  Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
  OutDir=analysis/meiotic_machinery/blast/$Organism/$Strain
  mkdir -p $OutDir
  echo "$Organism - $Strain"
  mkdir -p $OutDir
  Eval="1e-1"
  Prefix="saccharomyces_vs_${Strain}"
  # makeblastdb -in $Proteome -input_type fasta -dbtype "prot" -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
  # blastp -num_threads 16 -db $OutDir/$Prefix.db -query $SaccharomycesProts -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
  Eval="1e-1"
  Prefix="${Strain}_vs_saccharomyces"
  # makeblastdb -in $SaccharomycesProts -input_type fasta -dbtype "prot" -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
  # blastp -num_threads 16 -db $OutDir/$Prefix.db -query $Proteome -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
  ProgDir=/home/armita/git_repos/emr_repos/scripts/alternaria/pathogen/blast
  $ProgDir/analyse_recipricol_blast.py --subset $QueeryHeaders --hits_a_vs_b $OutDir/saccharomyces_vs_${Strain}_hits.txt --hits_b_vs_a $OutDir/${Strain}_vs_saccharomyces_hits.txt > $OutDir/${Strain}_reciprical_hits.tsv
done
```

```bash
ProgDir=/home/armita/git_repos/emr_repos/scripts/alternaria/pathogen/blast
$ProgDir/analyse_recipricol_blast.py --subset analysis/meiotic_machinery/86_meiotic_proteins.txt --hits_a_vs_b analysis/meiotic_machinery/blast/A.gaisen/650/saccharomyces_vs_650_hits.txt --hits_b_vs_a analysis/meiotic_machinery/blast/A.gaisen/650/650_vs_saccharomyces_hits.txt | less -S

```

## Mating type genes:

```bash
  mkdir -p analysis/blast/MAT_genes
  for Subject in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    Query=analysis/blast/MAT_genes/MAT_genes.fasta
    qsub $ProgDir/blast_pipe.sh $Query dna $Subject
  done
```

```bash
  HitsList=""
  StrainList=""
  for BlastHits in $(ls analysis/blast_homology/*/*/*_Nc_clock_gene.fa_homologs.csv); do
    sed -i "s/\t\t/\t/g" $BlastHits
    sed -i "s/Grp1\t//g" $BlastHits
    Strain=$(echo $BlastHits | rev | cut -f2 -d '/' | rev)
    HitsList="$HitsList $BlastHits"
    StrainList="$StrainList $Strain"
    ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/blast
    $ProgDir/blast_parse.py --blast_csv $HitsList --headers $StrainList --identity 0.7 --evalue 1e-30
  done
```


```bash
qlogin -pe smp 12
cd /data/scratch/armita/alternaria
QueryFasta=$(ls /data/scratch/armita/alternaria/analysis/blast/MAT_genes/MAT_genes.fasta)
dbType="nucl"
for dbFasta in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
Organism=$(echo $dbFasta | rev | cut -f4 -d '/' | rev)
Strain=$(echo $dbFasta | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
Prefix="${Strain}_MAT"
Eval="1e-30"
OutDir=analysis/blast_homology/$Organism/$Strain
mkdir -p $OutDir
makeblastdb -in $dbFasta -input_type fasta -dbtype $dbType -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
blastn -num_threads 12 -db $OutDir/$Prefix.db -query $QueryFasta -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
done
```

```bash
for File in $(ls analysis/blast_homology/*/*/*_MAT_hits.txt); do
  Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
  Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
  for Hit in $(cat $File | cut -f1 | cut -f1 -d '_'); do
    printf "$Organism\t$Strain\t$Hit\n"
  done
done
```

```
A.alternata_ssp_tenuissima  1166	MAT1-2-1
A.gaisen	650	MAT1-1-1
```

# Toxin genes

```bash
qlogin -pe smp 4
cd /home/groups/harrisonlab/project_files/alternaria
QueryFasta=$(ls analysis/blast_homology/CDC_genes/A.alternata_CDC_genes.fa)
dbType="nucl"
for dbFasta in $(ls /data/scratch/armita/alternaria/repeat_masked/*/*/filtered_contigs/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
Strain=$(echo $dbFasta | rev | cut -f3 -d '/' | rev)
Organism=$(echo $dbFasta | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
Prefix="${Strain}_toxgenes"
Eval="1e-30"
OutDir=analysis/blast_homology/$Organism/$Strain
mkdir -p $OutDir
makeblastdb -in $dbFasta -input_type fasta -dbtype $dbType -title $Prefix.db -parse_seqids -out $OutDir/$Prefix.db
blastn -num_threads 4 -db $OutDir/$Prefix.db -query $QueryFasta -outfmt 6 -num_alignments 1 -out $OutDir/${Prefix}_hits.txt -evalue $Eval
done
```

```bash
for File in $(ls analysis/blast_homology/*/*/*_toxgenes_hits.txt); do
  Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
  Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
  for Hit in $(cat $File | cut -f1 | cut -f1 -d ' '); do
    printf "$Organism\t$Strain\t$Hit\n"
  done
done > analysis/blast_homology/CDC_genes/ref_genome_summarised_hits.tsv
```


```bash
cat gene_pred/annotation/A.alternata_ssp_tenuissima/1166/1166_annotation_ncbi2.tsv | grep 'Cluster' | grep -e 'contig_15' -e 'contig_18' -e 'contig_20' -e 'contig_21' | cut -f12 | sort | uniq -c
cat gene_pred/annotation/A.alternata_ssp_tenuissima/1166/1166_annotation_ncbi2.tsv | grep 'Cluster' | grep -e 'contig_14' -e 'contig_19' | cut -f12 | sort | uniq -c

cat gene_pred/annotation/A.gaisen/650/650_annotation_ncbi.tsv | grep 'Cluster' | grep -e 'contig_14' -e 'contig_16' -e 'contig_23' -e 'contig_24' | cut -f2,12 | sort | uniq -c
```


# Investigate GC content and compartmentalisation

```bash
  for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    GeneGff=$(ls gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.gff3)
    ProjDir=$PWD
    OutDir=analysis/GC-content/$Organism/$Strain
    mkdir -p $OutDir
    cd $OutDir
    OcculterCut -f $ProjDir/$Assembly -a $ProjDir/$GeneGff
    PlotProg=$(ls /home/armita/prog/occultercut/OcculterCut_v1.1/plot.plt)
    gnuplot $PlotProg
    mv plot.eps ${Strain}_GC-plot.eps
    cd $ProjDir
  done
```

<!--
On my local computer:

```r
compositionGC <- read.table("~/Downloads/Aalt/GC-content/tmp/compositionGC.txt", quote="\"", comment.char="")
library(ggplot2)
ggplot(data=compositionGC, aes(compositionGC$V2)) + geom_histogram()

``` -->


# Detecting RIP


```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa); do
  Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
  Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
  echo "$Organism - $Strain"
  RepeatGff=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_transposonmasked.gff)
  ProjDir=$PWD
  OutDir=analysis/RIP_index/$Organism/$Strain
  mkdir -p $OutDir
  cd $OutDir
  cat $ProjDir/$RepeatGff | sed 's/similarity/repeat_region/g' | sed 's/Target "/Target=/g' | sed 's/" / /g' | sed "s/$/;/g" > tmp.gff
  # ripcal -c -seq $ProjDir/$Assembly -gff tmp.gff
  ripcal -c -index -seq $ProjDir/$Assembly
  cd $ProjDir
done
```

Alignments and output files were copied to my local machine. Alignments were
imported into geneious where four were selected for reanalysis based upon them
having more than 50 sequences that with regions common to all sequences.

The 50 longest sequences were selected for these four loci and realigned using
clustalw

Sequences were then uploaded back onto the cluster

```bash
scp -r /Users/armita/Downloads/Aalt/RIP/A.alternata_ssp_tenuissima/selected_loci cluster:/data/scratch/armita/alternaria/analysis/RIP/A.alternata_ssp_tenuissima/1166/.
```

```bash
for Alignment in $(ls analysis/RIP/*/*/selected_loci/*.fasta | grep '1166' | grep '105_top50'); do
  Organism=$(echo $Alignment | rev | cut -f3 -d '/' | rev)
  Strain=$(echo $Alignment | rev | cut -f2 -d '/' | rev)
  Prefix=$(basename ${Alignment%.fasta})
  echo "$Organism - $Strain"
  OutDir=$(dirname $Alignment)/${Prefix}
  mkdir -p $OutDir
  ProjDir=$PWD
  cd $OutDir
  ripcal -c -seq $ProjDir/$Alignment
  cd $ProjDir
done
```

```bash
for Assembly in $(ls repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa | grep '1166'); do
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
RepeatGff=$(ls repeat_masked/$Organism/$Strain/filtered_contigs/*_contigs_transposonmasked.gff | grep '1166')
for Alignment in $(ls analysis/RIP/$Organism/$Strain/selected_loci/*.fasta | grep 'rnd-3_family-105'); do
Prefix=$(basename ${Alignment%.fasta})
Prefix=$(echo $Prefix | sed 's/_alignment//g' | sed 's/_top50//g')
OutDir=$(dirname $Alignment)
echo $Prefix
ProgDir=/home/armita/git_repos/emr_repos/tools/pathogen/fungal_RIP/ripcurl
$ProgDir/get_random.py --genome $Assembly --gff $RepeatGff --alignment $Alignment > $OutDir/${Prefix}_RIP_index.tsv
done
done
```

```bash
scp cluster:/data/scratch/armita/alternaria/analysis/RIP/A.alternata_ssp_tenuissima/1166/selected_loci/*_RIP_index.tsv A.alternata_ssp_tenuissima/selected_loci/.
```


```r
library('ggplot2')
install.packages('ggsignif')
library('ggsignif')
setwd('~/Downloads/Aalt/RIP/A.alternata_ssp_tenuissima/selected_loci/')

filename <- "rnd-2_family-2_RIP_index.tsv"
# filename <- "rnd-3_family-105_RIP_index.tsv"
# filename <- "rnd-3_family-313_RIP_index.tsv"
# filename <- "rnd-4_family-739_RIP_index.tsv"

prefix <- tools::file_path_sans_ext(filename)
df1 <- read.delim(filename, header=FALSE)
colnames(df1) <- c('sample', 'TA', 'AT', 'ratio')

outfile = paste(prefix, 'txt', sep='.')
sink(outfile)
t.test(data = df1, ratio~sample)
sink()

#write(t, file = outfile)
p1 <- ggplot(df1, aes(x=sample, y=ratio)) +
  geom_boxplot(outlier.colour=NA)
p1 <- p1 + ylim(0, 2)
p1 <- p1 + xlab('')
p1 <- p1 + ylab('TpA/ApT Ratio')
p1 <- p1 + ggtitle('rnd-2 family-2')
p1 <- p1 + scale_x_discrete(labels = c('',''))
# p <- p + geom_signif(comparisons = list(c('control', 'sample')),
              # map_signif_level=TRUE)


filename <- "rnd-3_family-105_RIP_index.tsv"

prefix <- tools::file_path_sans_ext(filename)
df1 <- read.delim(filename, header=FALSE)
colnames(df1) <- c('sample', 'TA', 'AT', 'ratio')

outfile = paste(prefix, 'txt', sep='.')
sink(outfile)
t.test(data = df1, ratio~sample)
sink()

p2 <- ggplot(df1, aes(x=sample, y=ratio)) +
  geom_boxplot(outlier.colour=NA)
p2 <- p2 + ylim(0, 2)
p2 <- p2 + xlab('')
p2 <- p2 + ylab('TpA/ApT Ratio')
p2 <- p2 + ggtitle('rnd-3 family-105')
p2 <- p2 + scale_x_discrete(labels = c('control','transposon sequence'))


filename <- "rnd-3_family-313_RIP_index.tsv"

prefix <- tools::file_path_sans_ext(filename)
df1 <- read.delim(filename, header=FALSE)
colnames(df1) <- c('sample', 'TA', 'AT', 'ratio')

outfile = paste(prefix, 'txt', sep='.')
sink(outfile)
t.test(data = df1, ratio~sample)
sink()

p3 <- ggplot(df1, aes(x=sample, y=ratio)) +
  geom_boxplot(outlier.colour=NA)
p3 <- p3 + ylim(0, 2)
p3 <- p3 + xlab('')
p3 <- p3 + ylab('')
p3 <- p3 + ggtitle('rnd-3 family-313')
p3 <- p3 + scale_x_discrete(labels = c('',''))


filename <- "rnd-4_family-739_RIP_index.tsv"

prefix <- tools::file_path_sans_ext(filename)
df1 <- read.delim(filename, header=FALSE)
colnames(df1) <- c('sample', 'TA', 'AT', 'ratio')

outfile = paste(prefix, 'txt', sep='.')
sink(outfile)
t.test(data = df1, ratio~sample)
sink()

p4 <- ggplot(df1, aes(x=sample, y=ratio)) +
  geom_boxplot(outlier.colour=NA)
p4 <- p4 + ylim(0, 2)
p4 <- p4 + xlab('')
p4 <- p4 + ylab('')
p4 <- p4 + ggtitle('rnd-4 family-739')
p4 <- p4 + scale_x_discrete(labels = c('control','transposon sequence'))


# Multiple plot function
#
# ggplot objects can be passed in ..., or to plotlist (as a list of ggplot objects)
# - cols:   Number of columns in layout
# - layout: A matrix specifying the layout. If present, 'cols' is ignored.
#
# If the layout is something like matrix(c(1,2,3,3), nrow=2, byrow=TRUE),
# then plot 1 will go in the upper left, 2 will go in the upper right, and
# 3 will go all the way across the bottom.
#
multiplot <- function(..., plotlist=NULL, file, cols=1, layout=NULL) {
  library(grid)

  # Make a list from the ... arguments and plotlist
  plots <- c(list(...), plotlist)

  numPlots = length(plots)

  # If layout is NULL, then use 'cols' to determine layout
  if (is.null(layout)) {
    # Make the panel
    # ncol: Number of columns of plots
    # nrow: Number of rows needed, calculated from # of cols
    layout <- matrix(seq(1, cols * ceiling(numPlots/cols)),
                    ncol = cols, nrow = ceiling(numPlots/cols))
  }

 if (numPlots==1) {
    print(plots[[1]])

  } else {
    # Set up the page
    grid.newpage()
    pushViewport(viewport(layout = grid.layout(nrow(layout), ncol(layout))))

    # Make each plot, in the correct location
    for (i in 1:numPlots) {
      # Get the i,j matrix positions of the regions that contain this subplot
      matchidx <- as.data.frame(which(layout == i, arr.ind = TRUE))

      print(plots[[i]], vp = viewport(layout.pos.row = matchidx$row,
                                      layout.pos.col = matchidx$col))
    }
  }
}

p <- multiplot(p1, p2, p3, p4, cols=2)

outfile = paste("multiplot", 'pdf', sep='.')
ggsave(outfile, p, width = 10, height = 10)


```

```
	Welch Two Sample t-test

data:  ratio by sample
t = -1.5357, df = 72.403, p-value = 0.129
alternative hypothesis: true difference in means is not equal to 0
95 percent confidence interval:
 -0.39525503  0.05125503
sample estimates:
mean in group control  mean in group sample
               0.7788                0.9508
```


```r
setwd('~/Downloads/Aalt/RIP/A.alternata_ssp_tenuissima/selected_loci/')
df1 <- read.delim("rnd-4_family-739_RIP_index.tsv", header=FALSE)
colnames(df1) <- c('sample', 'TA', 'AT', 'ratio')

t.test(ratio~sample)

library('ggplot2')
p <- ggplot(df1, aes(x=sample, y=ratio)) +
  geom_boxplot()
```

```
	Welch Two Sample t-test

data:  ratio by sample
t = -1.5357, df = 72.403, p-value = 0.129
alternative hypothesis: true difference in means is not equal to 0
95 percent confidence interval:
 -0.39525503  0.05125503
sample estimates:
mean in group control  mean in group sample
               0.7788                0.9508
```

# Sharing data

```bash
for File in $(ls /data/scratch/armita/alternaria/repeat_masked/*/*/filtered_contigs/*_contigs_unmasked.fa | grep -e '1166' -e '650'); do
Organism=$(echo $File | cut -f7 -d '/')
Strain=$(echo $File | cut -f8 -d '/')
echo "$Organism - $Strain"
OutDir=assembly/Armitage_genomes/$Organism/$Strain
mkdir -p $OutDir
cp $File $OutDir/.
cp ${File%_unmasked.fa}_softmasked_repeatmasker_TPSI_appended.fa $OutDir/.
cp ${File%_unmasked.fa}_hardmasked_repeatmasker_TPSI_appended.fa $OutDir/.
cp /data/scratch/armita/alternaria/gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.* $OutDir/.
cp /data/scratch/armita/alternaria/gene_pred/interproscan/$Organism/$Strain/${Strain}_interproscan.tsv $OutDir/.
cp /data/scratch/armita/alternaria/gene_pred/swissprot/$Organism/$Strain/swissprot_vMar2018_tophit_parsed.tbl $OutDir/${Strain}_swissprot_tophit.tbl
cp /data/scratch/armita/alternaria/gene_pred/annotation/$Organism/$Strain/${Strain}_annotation_ncbi.tsv $OutDir/.
done
```
