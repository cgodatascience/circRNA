[![DOI](https://zenodo.org/badge/131415787.svg)](https://zenodo.org/badge/latestdoi/131415787)

# Classification of RNA-Seq samples using circRNA expression
A collection of R-scripts to automate the download of RNA-Seq samples, quality control, preprocessing, detection of circRNAs and classification using machine learning from the detected circRNAs.

![pipeline](https://carmengmz.github.io/circRNA/circRNA-pipeline.png)

## User manual

### Downloading the SRAs

In first place we must select a set of sequenced RNA-seq samples suitable for the detection of circRNAs. In [Selection of the RNA-Seq library](https://github.com/carmengmz/circRNA/wiki/Selection-of-RNA-Seq-suitable-for-circRNA-detection) can be find some tips on how to make the selection.

If the detected circRNAs will be used for classification, the samples must belong to different groups. For example, sequenced samples from normal patients vs sequenced samples from patients with some disease or condition. In our example we will select sequenced samples of blood exosomes from normal patiens and sequenced samples of blood exosomes from patiens with coronary heart disease.

We have performed the search in the [Gene Expression Omnibus repository (GEO)](https://www.ncbi.nlm.nih.gov/gds) of the NCBI (National Center for Biotechnology Information) using the following criteria:
- <b>Organism</b>: Homo sapiens
- <b>Study type</b>: Expression profiling by throughput sequencing
- <b>Query</b>: "blood exosomes"

From the search results we have selected two projects containing sequenced samples from human blood exosomes, which were also used to characterize circRNA in its associated publication.
- [RNA-seq reveals abundant circRNA, lncRNA and mRNA in blood exosomes of patients with coronary heart disease](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99985)
- [RNA-seq reveals abundant circRNA, lncRNA and mRNA in blood exosomes of normal persons](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE100206)

From every result (follow the links) we retrieve the id of the BioProyect: PRJNA390278 and 	PRJNA390988.

Once we have the BioProyect id, we will retrieve the ids of the SRA Runs in: https://www.ncbi.nlm.nih.gov/Traces/study/ (under the column Run) and we will build a text file separated by tabs called <b>phenodata.txt</b> with two columns: <b>File</b> and <b>Group</b>. In our [example](https://github.com/carmengmz/circRNA/tree/master/example) we will work with six samples of each group:

```
File        Group
SRR5679904  coronary
SRR5679905  coronary
SRR5679906  coronary
SRR5679907  coronary
SRR5679908  coronary
SRR5679909  coronary
SRR5712482  normal
SRR5712483  normal
SRR5712484  normal
SRR5712485  normal
SRR5712486  normal
SRR5712487  normal
```

Now we can run the script [Download.R](https://github.com/carmengmz/circRNA/blob/master/src/Download.R) to download the <b>*.sra</b> files. The <b>phenodata.txt</b> file must be in the working directory. In a Unix like S.O. command line we will type:
```
> Rscript Download.R
```
As a result, at the end of the script, in the working directory we will have the downloaded sra files:

```zsh
phenodata.txt   SRR5679907.sra  SRR5712483.sra    SRR5712487.sra     
SRR5679904.sra  SRR5679908.sra  SRR5712484.sra   
SRR5679905.sra  SRR5679909.sra  SRR5712485.sra          
SRR5679906.sra  SRR5712482.sra  SRR5712486.sra 
```

### Converting to FASTQ

Once the SRAs are downloaded, the next step is to convert the files to the FASTQ format, separating the readings into two different files in the case of paired-end reads. To do this we are going to use the <b>fastq-dump</b> tool included in the [NCBI's SRA Toolkit](https://trace.ncbi.nlm.nih.gov/Traces/sra/sra.cgi?view=software) available for several operating systems. The automation of this process has been implemented in the [Trans.R](https://github.com/carmengmz/circRNA/blob/master/src/Trans.R) script. The command to convert the files to the FASTQ format with <b>fastq-dump</b> is:
```
fastq-dump --split-3 <file.sra>
```
The <b>--split-3</b> option generates a single FASTQ file if we have single-end reads or two FASTQ files separating the readings paired in case of paired-end reads. In this case the file names will have the suffixes _1.FASTQ and _2.FASTQ. In addition, we can obtain a third file FASTQ without suffix and of a much smaller size, which will contain "orphan" readings. We will discard this file.

The <b>fastq-dump</b> tool must be able to be executed directly in the working directory, for example, adding the path to the executable to the <b>.bashrc</b> file on a Unix system or to the global PATH variable on a Windows system. The <b>*.sra</b> files and the <b>phenodata.txt</b> file must be too in the working directory. In a Unix like S.O. command line we will type:

```
> Rscript Trans.R
```
As a result, at the end of the script, in the working directory we will have the FASTQ files. SRA files can now be deleted to preserve disk space. In our example we have paired-end reads and this is the content of the working directory:

```
phenodata.txt       SRR5679907_1.fastq  SRR5712482_2.fastq  SRR5712486_1.fastq
SRR5679904_1.fastq  SRR5679907_2.fastq  SRR5712483_1.fastq  SRR5712486_2.fastq
SRR5679904_2.fastq  SRR5679908_1.fastq  SRR5712483_2.fastq  SRR5712487_1.fastq
SRR5679905_1.fastq  SRR5679908_2.fastq  SRR5712484_1.fastq  SRR5712487_2.fastq
SRR5679905_2.fastq  SRR5679909_1.fastq  SRR5712484_2.fastq
SRR5679906_1.fastq  SRR5679909_2.fastq  SRR5712485_1.fastq
SRR5679906_2.fastq  SRR5712482_1.fastq  SRR5712485_2.fastq
```

### Quality control and cleaning of raw data

The integral evaluation of the quality and the preprocessing of the raw data are the first and most critical steps for all subsequent analyzes and the correct interpretation of the results. We will use:
- [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/): quality control before and after cleaning the raw data (MultiQC has not yet implemented the integration with Fastp).
- [Fastp](https://github.com/OpenGene/fastp): elimination of adapters, filtering of the readings and correction of the bases.
- [MultiQC](http://multiqc.info): summarize FASTQC quality reports  

In [Cleaning and preprocessing RNA-Seq](https://github.com/carmengmz/circRNA/wiki/Cleaning-and-preprocessing-RNA-Seq) are explained the reasons for the choice of these tools. 

All the tools (<b>FASTQC</b>, <b>Fastp</b> and <b>MultiQC</b>) must be able to be executed directly in the working directory for example, adding the path to the executables to the <b>.bashrc</b> file in an Unix like SO. <b>Fastp</b> is avalaible for Linux and macOs but not for Windows. The script [Clean.R](https://github.com/carmengmz/circRNA/blob/master/src/Clean.R) automate the task of quality control and cleaning:

1. In first place, it will make quality control of raw FASTQ with the <b>FASTQC</b> tool. Then it will summarize quality reports with <b>MultiQC</b> by each group defined in <b>phenodata.txt</b>. The reports will be stored in folders:  <b>&lt;group&gt;_qc_raw</b> (one folder by each group). The MultiQC report is a "*.html" file with MQC_ prefix inside these folders.
  
2. Then it will use <b>Fastp</b> tool to clean raw data. As a result we will have the same FASTQ files but with suffixes <b>_clean.FASTQ</b> (for single-end reads) or <b>_clean_1.FASTQ</b> and <b>_clean_2.FASTQ</b> (for paired-end reads). <b>Fastp</b> also generates a quality control report for each file (or pair of files if they are paired-end reads). These reports will be stored in the <b>fastp</b> folder.

3. And to finish it will make quality control of clean FASTQ with the <b>FASTQC</b> tool. Then it will sumarize quality reports with <b>MultiQC</b> by each group defined in <b>phenodata.txt</b>. The reports will be stored in folders: <b>&lt;group&gt;_qc_clean</b> (one folder by each group). The MultiQC report is a html file with MQC_ prefix inside these folders.

The raw <b>*.fastq</b> files and the <b>phenodata.txt</b> file must be in the working directory. In a Unix like S.O. command line we will run the [Clean.R](https://github.com/carmengmz/circRNA/blob/master/src/Clean.R) script with:

```
> Rscript Clean.R
```
As a result, at the end of the script, in the working directory we will have these files: the clean FASTQ files and the folders with the reports. In our example this is the content of the working directory (raw FASTQ files can now be deleted to preserve disk space):

```
fastp                       SRR5679906_clean_1.fastq    SRR5712483_clean_1.fastq
coronary_qc_raw             SRR5679906_clean_2.fastq    SRR5712483_clean_2.fastq
coronary_qc_clean           SRR5679907_clean_1.fastq    SRR5712484_clean_1.fastq
normal_qc_raw               SRR5679907_clean_2.fastq    SRR5712484_clean_2.fastq
normal_qc_clean             SRR5679908_clean_1.fastq    SRR5712485_clean_1.fastq
phenodata.txt               SRR5679908_clean_2.fastq    SRR5712485_clean_2.fastq
SRR5679904_clean_1.fastq    SRR5679909_clean_1.fastq    SRR5712486_clean_1.fastq
SRR5679904_clean_2.fastq    SRR5679909_clean_2.fastq    SRR5712486_clean_2.fastq
SRR5679905_clean_1.fastq    SRR5712482_clean_1.fastq    SRR5712487_clean_1.fastq
SRR5679905_clean_2.fastq    SRR5712482_clean_2.fastq    SRR5712487_clean_2.fastq
```

The generated reports are available in the [example](https://github.com/carmengmz/circRNA/tree/master/example) folder.

### Detection of circRNA
In recent years, multiple tools and pipelines have been developed for the identification of circRNAs. In parallel we can find published studies comparing the different detection tools. 

In this project we will use the CIRI2 tool to identify the circRNAs present in our samples. To do that, we first need to align the readings to a reference genome using the BWA aligner. BWA is a tool for mapping divergent sequences against a large reference genome, such as the human genome. In order to perform this task you will need to download the reference genome in FASTA format and its annotations in GTF format. We will use the latest available version of the reference genome GRCh38. If you need to use other reference genome, you can edit the <b>reference</b> variable in [Findcrna.R](https://github.com/carmengmz/circRNA/blob/master/src/Findcrna.R) script. 

For example, you can download and uncompress reference genome from Ensembl using the following UNIX commands. For homo sapiens the file labeled toplevel combines all chromosomes. 

```
wget ftp://ftp.ensembl.org/pub/release-77/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.toplevel.fa.gz 
gunzip Homo_sapiens.GRCh38.dna.toplevel.fa.gz 
mv Homo_sapiens.GRCh38.dna.toplevel.fa hg38.fa
```
And Homo sapiens gene model annotation can be downloaded as follow:
```
wget ftp://ftp.ensembl.org/pub/release-77/gtf/homo_sapiens/Homo_sapiens.GRCh38.77.gtf.gz 
gunzip Homo_sapiens.GRCh38.77.gtf.gz 
mv Homo_sapiens.GRCh38.77.gtf hg38.gtf
```
It is important to download both files (the FASTA reference sequence and the GTF data with the annotations) from the same provider. 
 
The the first step to using BWA for aligning our sequences is to build a reference genome index in fasta format:
```
bwa index -a bwtsw hg38.fa
```
Now, we align our readings to the reference genome (for more details, see the CIRI2 manual):
```
bwa mem –T 19 -t 10 hg38.fa reads.fq > aln-se.sam (for single-end reads)
bwa mem –T 19 -t 10 hg38.fa read1.fq read2.fq > aln-pe.sam (for paired-end reads)
```
Finally, we passed the sam file generated in the previous command to the CIRI2 tool:
```
perl CIRI2.pl -T 10 -I aln-pe.sam -O  outfile -F hg38.fa -A hg38.gtf > ciri.log
```
Here we are using 10 threads (<b>-t 10</b> in BWA command and <b>-T 10</b> in CIRI2 command), you can change that modifing the value of the <b>num_thread</b> variable in the [Findcrna.R](https://github.com/carmengmz/circRNA/blob/master/src/Findcrna.R) script.

As a result we will get a text file <b>outfile</b> with the circRNAs identified in our reads.

To automate the detection of circRNAs in all samples, the [Findcrna.R](https://github.com/carmengmz/circRNA/blob/master/src/Findcrna.R) script has been implemented. The dependencies are the following (must be able to be executed directly in the working directory):

- [BWA-MEM Aligner](http://bio-bwa.sourceforge.net/)
- [Perl](https://www.perl.org/)
- [CIRI2](https://sourceforge.net/projects/ciri/files/CIRI2/)

Also, the following files must be in working directory:
- the FASTA reference sequence named <b>hg38.fa</b> and the GTF data with the annotations <b>hg38.gtf</b>. This names can be changed editing the <b>ref_fasta</b> and <b>ref_gtf</b> variables in [Findcrna.R](https://github.com/carmengmz/circRNA/blob/master/src/Findcrna.R) script.
- the <b>phenodata.txt</b> file.
- the clean FASTQ sequences.   
    -- Expected names for singled-end data: <b>&lt;File&gt;_clean.fastq</b>       
    -- Expected names for paired-end data: <b>&lt;File&gt;_clean_1.fastq</b> and <b>&lt;File&gt;_clean_2.fastq</b>     
    being <b>File</b> the name of each sample specified in <b>phenodata.txt</b>

In a Unix like S.O. command line we will run the [Findcrna.R](https://github.com/carmengmz/circRNA/blob/master/src/Findcrna.R) script whith:

```
> Rscript Findcrna.R
```

As a result we will get a text file for each sample named <b>&lt;File&gt;-outfile</b> with the detected circRNAs. The outfiles of our example can be found in the [example](https://github.com/carmengmz/circRNA/tree/master/example) forder. Also we will get the sam files with the alignment of each sample to the reference genome, the log files of the aligment and the log files of CIRI2.

This would be now the content of our working directory (clean FASTQ files have been deleted to preserve disk space):
```
phenodata.txt       SRR5679907-outfile  SRR5712482-aln.sam   SRR5712486-outfile    
SRR5679904-outfile  SRR5679907-aln.sam  SRR5712483-outfile   SRR5712486-aln.sam
SRR5679904-aln.sam  SRR5679908-outfile  SRR5712483-aln.sam   SRR5712487-outfile
SRR5679905-outfile  SRR5679908-aln.sam  SRR5712484-outfile   SRR5712487-aln.sam
SRR5679905-aln.sam  SRR5679909-outfile  SRR5712484-aln.sam          
SRR5679906-outfile  SRR5679909-aln.sam  SRR5712485-outfile 
SRR5679906-aln.sam  SRR5712482-outfile  SRR5712485-aln.sam  
```

### Annotating circRNA to generate table with counts

Once obtained, using the CIRI2 tool, the output files (<identifier>-outfile) with the circRNAs detected in each sample, we will proceed to the creation of a single table with the readings counts of the circRNAs of each sample. We will use the following fields:
  
<table>
  <tr> 
    <td> Field in the output of CIRI2 </td><td> Description </td>
  </tr>
  <tr>
    <td> circRNA_ID </td> <td>  Identifier for the circRNA in the chromosome format: start | final.
    <br/>For example, chr1: 1223244 | 1223968 </td>
  </tr>
  <tr>
    <td> chr </td> <td> Chromosome </td>
  </tr>
  <tr>
    <td> circRNA_start </td> <td> Initial coordinate </td>
  </tr>
  <tr>
    <td> circRNA_end </td> <td> Final coordinate </td>
  </tr>
  <tr>
    <td> #junction_readas </td> <td> Number of reads </td>
  </tr>
  <tr>
    <td> strand </td> <td> Strand (+/-) </td>
  </tr>
</table>

To construct the table of readings, a margin of 10 positions was allowed at the beginning and at the end of the coordinates to consider that a detection belonged to the same circRNA. 

The automation of the annotation process to build the table with the reading counts has been carried out in the [Annotate.R](https://github.com/carmengmz/circRNA/blob/master/src/Annotate.R) script. 

The range of difference positions allowed at the start and end can be set in the variable <b>range</b>. The script can be executed in any operating system that supports the environment R. 

In a Unix/Linux environment we would do it with the following command:
```
> Rscript Annotate.R
```
As a result we will get the counts table in a file called [circ_annotations.rds](https://github.com/carmengmz/circRNA/tree/master/example/circ_annotations.rds). The annotations file for our example can be found in the [example](https://github.com/carmengmz/circRNA/tree/master/example) folder.

### Machine Learning Classification using the normalized circRNA counts table

Classification using Machine Learning has been automated in the R Markdown [Classify.Rmd](https://github.com/carmengmz/circRNA/blob/master/src/Classify.Rmd) script. This script can be executed on any operating system with a graphical environment in which Rstudio can be installed (https://www.rstudio.com/)

The script needs to be tuned. Here we are going to do so for our classification example, using read counts of circRNAs detected and quantified from peripherical blood exosomes, from patients with Coronary Heart Disease and healthy people.

In first place, we will load the read counts from [circ_annotations.rds](https://github.com/carmengmz/circRNA/tree/master/example/circ_annotations.rds) and group info from [phenodata.txt](https://github.com/carmengmz/circRNA/tree/master/example/phenodata.txt)

```
library(readr)
phenodata <- read_delim("phenodata.txt", "\t", escape_double = FALSE, trim_ws = TRUE)
table <- readRDS("circ_anotations.rds")
```

Then, we will set the base and contrast groups with the same names that can be found in [phenodata.txt](https://github.com/carmengmz/circRNA/tree/master/example/phenodata.txt):

```
contrast = "coronary"
base = "normal"
```

Now, we are going to filter circRNAs whose reading counts are below 10 (in absolute value) in at least 70% of the samples (you can change these values inside the script)

```
filter_value <-10
perc_smaples <- 0.7

counts <- table[,phenodata$File[phenodata$Group %in% c(base,contrast)]]
counts <- counts[ rowSums(counts >= filter_value) >= dim(counts)[2]*perc_samples,]
group <- phenodata$Group[phenodata$Group %in% c(base,contrast)]
```

And we will use the stabilizing variance transformation (VST) to eliminate the dependence between the mean and the variance using DESeq2 package. 

```
library(DESeq2)
vst <- vst(as.matrix(counts), fitType="parametric")
```

Next, we will generate train and test sets, since the model must be trained in a subset of samples (train) and then validated in a different subset of samples (test), with 20% of samples in test set (you can change this value)

```
test_perc <- 0.2
dat <- getPartition(as.factor(group), vst, test_perc)

xtrain <- dat$xtrain
ytrain <- factor(dat$ytrain$condition)

xtest <- dat$xtest
ytest <- factor(dat$ytest$condition)
```

We will continue selecting, with a Random Forest algorithm, in the training set, the circRNAs considered most important. Whith these circRNAs we will train three different classification models: Support Vector Machines, Random Forests and Extreme Learning Neural Networks. 

```
library(randomForest)
forest.imp = randomForest(class ~. , data = data.frame(t(xtrain), class = ytrain), 
		ntree = 1000, keep.forest = FALSE, importance = TRUE)
	
att.scores = as.data.frame(importance(forest.imp, type = 1))
```
The number of predictors (circRNAs) using for classification in our example will be k=10, you must tune this number guided by the MDS graph (tip: the lower number that makes data grouping in MDS): 

```
k=10
selected.circs <- rownames(att.scores)[order(att.scores, decreasing = TRUE)][1:k]
selected.circs

xtrain.cut = xtrain[selected.circs,]
```

Training a Support Vector Machine: we get AUC=1 in train set

```{r}
library(e1071)

hyper_cost = 2^(0:9)
hyper_kernel = "linear"
cross_folds = 5

svm_tune <- tune.svm(class ~ ., data=data.frame(class=ytrain, t(xtrain.cut)),  
                          cost = hyper_cost, tunecontrol=tune.control(cross=cross_folds))
svm_tune
plot(svm_tune)
------------------------

Parameter tuning of ‘svm’:

- sampling method: 5-fold cross validation 

- best parameters:
 cost
    1

- best performance: 0 
```

And also AUC=1 in test set:

```
library(caret)
pred.svm <- predict(svm_tune$best.model,t(xtest))
confusionMatrix(pred.svm, ytest)
------------------------

Confusion Matrix and Statistics

          Reference
Prediction coronary normal
  coronary        2      0
  normal          0      2
                                     
               Accuracy : 1          
                 95% CI : (0.3976, 1)
    No Information Rate : 0.5        
    P-Value [Acc > NIR] : 0.0625     
                                     
                  Kappa : 1          
 Mcnemar's Test P-Value : NA         
                                     
            Sensitivity : 1.0        
            Specificity : 1.0        
         Pos Pred Value : 1.0        
         Neg Pred Value : 1.0        
             Prevalence : 0.5        
         Detection Rate : 0.5        
   Detection Prevalence : 0.5        
      Balanced Accuracy : 1.0        
                                     
       'Positive' Class : coronary   
                                     
```
Same results have been obtained with Random Forest and Extreme machine Learning Classification models. A tuned script with the example can be found in  [Coronary-Classify.Rmd](https://github.com/carmengmz/circRNA/blob/master/example/Coronary-Classify.Rmd) and the output report can be viewed on-line in [Coronary-Classify.html](https://carmengmz.github.io/circRNA/example/Coronary-Classify.html)

In conclusion, we have been able to discriminate between samples of patients with coronary heart disease and healthy people with 100% accuracy. These same results have been obtained for colorectal and hepatocellular cancer. For pancreatic cancer we get an AUC=0.8333.

This result shows that circRNAs present in exosomes of human peripheral blood are expressed so differently for healthy people and people with various diseases, that allows classification by machine learning with 100% accuracy.

The reports for hepatocellular, colorectal and pacreatic cancer can be found in the [experiment](https://github.com/carmengmz/circRNA/blob/master/experiment) folder and online in: https://carmengmz.github.io/circRNA/experiment/


