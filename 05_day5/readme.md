Today, we will explore the annotation of the genome, to see if some of the outliers we found might belong to candidate genes, the distribution of our SNPs in exons, introns, regulatory regions,etc. We will also look at the annotation of repeated and transposable elements and compare that to our CNV detection.

## Getting started 

First of all, like previous days please copy into your directory the folder 05_day5, open it and we will work from there all day (excpet for snpEff.

```
cp -r Share/physalia_adaptation/05_day5 .
cd 05_day5
```

Inhere, you should have everything we need for today. You can explore the different folders and files
``` ls 02_data``` in which you will find the vcf, as well as the annotated genome and transcriptome

Have a look at those files

``` ls 03_outliers``` in which I put the txt files that we exported on day 3 when analysing the association with temperature, and on day4 when analysing divergence between haploblocks or between sexes
I also put here the list of all SNPs present in the vcf. you can look at each file with the command ``` head 03_outliers/SNP_pos.txt``` for instance

## SNPeff : annotating our snps 

### On the server
This program will use the gff file and the position of each SNP to annotate the vcf.
If you work on a model species which already has a database, you are lucky! If not, we will learn how to build a database

#### Step 1: prepare the database
First we need to add a line into the config file, for this we will use the editor nano which is easy-to-use

```
#go inside the snpEff folder
cd snpEff 
#edit 
nano snpEff.config
```
You should see a file with many lines explaining databases etc...
just add a new line and write
```
#new genome
genome_mallotus_dummy.genome : capelin
```
To exit nano make, ctrl-X and "yes"

Now we will put the right files at the right place.
```
cp ~Share/ressources/genome_mallotus_dummy.gff data/genome_mallotus_dummy/genes.gff
cp ~Share/ressources/genome_mallotus_dummy.fasta data/genomes/genome_mallotus_dummy.fa
cp ~Share/ressources/genome_mallotus_dummy.fasta data/genomes/genome_mallotus_dummy.genome
```

And build the database:
```
java -jar snpEff.jar build -gff3 -v genome_mallotus_dummy
#have a look at the database:
java -jar snpEff.jar dump genome_mallotus_dummy | less
```
To exit "less" simply press "q"


#### Step 2: annotate the vcf
Now we can annotate our vcf and exit snpEff. As you are getting used to now, we use a raw vcf file in the folder 02_data and will write the output into the folder 04_snpEff in which we will ahve all subsequent files related to the snpEff analyses
```
java -Xmx4g -jar snpEff.jar genome_mallotus_dummy ../02_data/canada.vcf > ../04_snpEff/canada_annotated.vcf
cd ..
```

Let's look at the new vcf
```
less -S 04_snpEff/canada_annotated.vcf
```
As you see it kepts the vcf format with its specific header but this is not easy to use as it is now if we want to import SNP information in R.
We will use a few bash command and awk to spli the information by the symbol "|" and rather make different column separated by tab

```
cat 04_snpEff/canada_annotated.vcf | awk -F"|" '$1=$1' OFS="\t" | cut -f 1-9 > 04_snpEff/SNP_annotated_formatted.txt
#look at the first 25 lines
head -n 25 04_snpEff/SNP_annotated_formatted.txt
#look at the last lines
tail 04_snpEff/SNP_annotated_formatted.txt
```

### On your computer, in R studio
Please copy the whole folder 05_day5 to your local computer, and set in Rstudio your working directory as 05_day5

analyse the results of snpEff in R ---> coming soon

## Bedtools : find the intersection between SNPs and genes
Bedtools is a program that is great to find the intersection between two files. It usually works on a specific bedformat which have at least three columns (Chromosome, FromPosition, ToPosition) and 12 columns to the maximum.
It won't like having header. We will try to keeep a 4th column with SNP id.

I have prepared for your the annotation file of the genome in a bed-readable format, that you may want to see by doing
```
less -S 05_bed/genome_mallotus_dummy_annotation_simplified.bed
```

If you remember our outliers files they were not formatted as such... Because we do not cover all the genome, we will look for genes in a window of X kb around the SNP position. The size of this window should ideally be adjust depending on LD decay in your organism (which can be assessed by plotting LD against distance with the output of plink that we found yesterday... For today we will choos 10 kb but if you are curious you cna explore different size.
To prepare the files, we will use R. Note that you could also easily to that in command lines if you wish.

### In R (either on Rstudio in your computer or in a R terminal on the server)

```
#load file
outlier_temp_rda<-read.table("03_outliers/outlier_temp_rda.txt", header=T)
#have a quick look
head(outlier_temp_rda)
#what's its dimension?
dim(outlier_temp_rda)

#which size around the SNP
window<-10000
#add a vector with start position
outlier_temp_rda$start<-outlier_temp_rda$position-(window/2)
#oups it can't be ngative! replace negative by 0
outlier_temp_rda$start[outlier_temp_rda$start<0]<-0 
#add a vector with stop position
outlier_temp_rda$stop<-outlier_temp_rda$position+(window/2)
#have a look
head(outlier_temp_rda)

#which columns shoud we keep?
outlier_temp_rda_bed<-outlier_temp_rda[,c(2,5,6,1)]
#save your file
write.table(outlier_temp_rda_bed, "05_bed/outlier_temp_rda.bed", row.names=F, sep="\t", quote=F,col.names=F)
```
Now that we are good for the rda outliers, we also need to do so for all the SNPs if we wan tto do later gene enrichment analysis. We want to know the pool of genes in which outliers may have been drawn.

```
all_snps<-read.table("03_outliers/SNP_pos.txt", header=T)
all_snps$start<-all_snps$position-(window/2)
all_snps$start[all_snps$start<0]<-0 
all_snps$stop<-all_snps$position+(window/2)
head(all_snps)
all_snps_bed<-all_snps[,c(2,4,5,1)]
write.table(all_snps_bed, "05_bed/all_snps.bed", row.names=F, sep="\t", quote=F,col.names=F)
```

And you can do the same for the outliers from baypass, or the outliers of divergence on chr4 or chr5... (if you are late skip)

### on the server
Please copy back your formatted outliers and snp bedfiles into the 05_day5/05_bed folder on the server. 
We will now run bedtools. The command intersect will look for overlap between the file givne with -a, and the file given with -b, the argument -wb will print the information coming from the annotation file (gene names, gene ontology, uniprot ID, etc)
the command ">" redirect the output in the file of your choice


With the command wc -l we will count the lines to see how many transcripts intersect with our outliers

```
bedtools intersect -a 05_bed/outlier_temp_rda.bed -b 05_bed/genome_mallotus_dummy_annotation_simplified.bed -wb > 05_bed/outlier_temp_rda.intersect
cat 05_bed/outlier_temp_rda.intersect | wc -l
head 05_bed/outlier_temp_rda.intersect
```
For gene enrichment analyses we will need a simpler format, so we will cut the 8th column (transcript name) with the command cut and keep it for goseq
```
cat 05_bed/outlier_temp_rda.intersect | cut -f 8 > 06_go/outlier_temp_rda.transcript
head 06_go/outlier_temp_rda.transcript
```

We repeat the analyses for all SNPs

```
bedtools intersect -a 05_bed/all_snps.bed -b 05_bed/genome_mallotus_dummy_annotation_simplified.bed -wb > 05_bed/all_snps.intersect
cat 05_bed/all_snps.intersect | wc -l

cat 05_bed/all_snps.intersect | cut -f 8 > 06_go/all_snps.transcript
head 06_go/all_snps.transcript
```

You may now repeat the commands for other outliers files if you wish. (if you are late skip)

### Have a look at your results
So copying back your .intersect files (and .transcript) on your computer, you can easily open them with a text editor (notepad ++ or equivalent). You cna see the name of the genes, gene ontology. What do you think?
For information about gene ontology codes, you may want to check this website http://geneontology.org/docs/ontology-documentation/

For information about proteins, you can look at https://www.uniprot.org/

## Goseq Gene ontology enrichment analysis
#### [WARNING]#### Gene enrichment analysis are mostly designed for RNAseq analyses in which all genes are analysed and of much better use when you have whole-genome data. Here on RAD-seq data, in which we subsample the genome, and with so few outliers, we should not make this kind of analysis, or at least not overinterpret it. We decided to include it in the tutorial for teaching purpose since most of you will be working with whole genome data but please keep in mind this warning!

Note: This script was built with the heolp or Dr. E. Berdan

This part will only be in Rstudio on your computer.
Please install the library goseq ( https://bioconductor.org/packages/release/bioc/html/goseq.html)
Disclaimer: I admit that this library is a pain because you will see that it requires many little twists like putting the transcripts in row.names, etc that makes it difficult to prepare the data... Hopefully there will be simpler versions or maybe one of you knows an easier tool that he/she may recommend...


```
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")

BiocManager::install("goseq")
library(goseq)
```


#### Step 1: Prepare our correspondance transcript/GO
Again, as we are on a non-model organism (and a dummy genome), there is no database of our transcriptome and GO terms in the library, so we will build it.

We have prepared for you a simplified annotation of the transcripts, and tried to remove duplicates. In fact, when a genome gets annotated, or when contigs from a transcriptome are aligned to a genome, one may have mutliple matches, which in turn may inflate our calculations. (if one SNP hits 10 times the same gene but with a different transcript name..)

```
transcript_info = read.table("06_go/transcript_go_simplified.txt", header=T, stringsAsFactors=F, sep="\t")
head(transcript_info)
row.names(transcript_info)<-transcript_info$TranscriptName
```
As you see go terms are all side by side, which will not be super helpful to make a list transcripts/GO
Please install the library splitstackshape which is able to split a whole column, and the library data.table which is super helpful to transform wide datatbale into long datatable
```
library(splitstackshape)
#split the GO terms
go_split = cSplit(transcript_info, "GeneGo", sep = ";")
go_split$contig<- transcript_info$TranscriptName
head(go_split)
```

```
library(data.table)
#linearize the matrix
terms = colnames(select(go_split, contains("GeneGo")))
go_long = melt(go_split,measure.vars=terms, id.vars = "contig", na.rm=TRUE)
head(go_long)
go_ready = as.data.frame(go_long[,c(1, 3)])
head(go_ready)
```

#### Step 2: Enrichment test


```
#what are the transcript covered by our dataset?
all_transcripts<-read.table("05_bed/all_snps.transcript", header=F)
colnames(all_transcripts)[1]<-"TranscriptName"
dim(all_transcripts)
head(all_transcripts)
```
```
#add size info
all_transcripts<-left_join(all_transcripts,transcript_info[,c(1,3)])
dim(all_transcripts)
#make unique
all_transcripts_unique<- all_transcripts %>% distinct(TranscriptName,.keep_all = TRUE)
dim(all_transcripts_unique)
row.names(all_transcripts_unique)<-all_transcripts_unique$TranscriptName
head(all_transcripts_unique)

```
#transcripts in outliers
outliers_transcripts_temp_rda<-read.table("05_bed/outlier_temp_rda.transcript", header=F)
colnames(outliers_transcripts_temp_rda)<-"TranscriptName"
head(outliers_transcripts_temp_rda)
dim(outliers_transcripts_temp_rda)
```
```
all_transcripts_unique$outliers_temp_rda<- as.numeric(all_transcripts_unique$TranscriptName %in% outliers_transcripts_temp_rda$TranscriptName)
head(all_transcripts_unique)
```
```
measured_genes = as.vector(all_transcripts_unique$TranscriptName)
outliers_genes = as.vector(all_transcripts_unique$outliers_temp_rda)
length = as.vector(all_transcripts_unique$length)
```
```
pwf_outliers = nullp(outliers_genes, bias.data=length)
row.names(pwf_outliers)<-row.names(all_transcripts_unique)
head(pwf_outliers)
```
```
enrich_outliers = goseq(pwf_outliers,gene2cat=go_ready, use_genes_without_cat=TRUE)
enrich_outliers$over_represented_padjust<-p.adjust(enrich_outliers$over_represented_pvalue,method="BH")
head(enrich_outliers)
```
```
enrich_outliers[which(enrich_outliers$over_represented_padjust<0.10),]
```



