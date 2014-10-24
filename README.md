# Bionode Example with Dat and Gasket
This is a basic example of using [Dat](http://dat-data.com), [Gasket](https://github.com/datproject/gasket) and [Bionode](http://bionode.io) to do reproducible bioinformatics.

## WORK IN PROGRESS
This example is not yet fully functional. We are fixing some bugs.

## Install
On your command line, run the following:
```bash
git clone git@github.com:bionode/bionode-example-dat-gasket.git
cd bionode-example-dat-gasket
npm install
```

## Run

There are 4 pipelines defined in `package.json` that you can run in order.

### Create a new empty dat repo

```bash
npm run init
```

### Get all [Eukaryota](http://en.wikipedia.org/wiki/Eukaryote) genomes [metadata](http://en.wikipedia.org/wiki/Metadata) from [NCBI](http://www.ncbi.nlm.nih.gov) into Dat
```bash
npm run fetch
```
#### What is happening?
The previous command will run a pipeline named ```fetch-eukaryota-genomes-metadata``` that is stored inside the [package.json](https://github.com/bionode/bionode-example-dat-gasket/blob/master/package.json) file of this git repository.

The following is a description of that pipeline:
```javascript
// Initialize a Dat repository (stored locally in the .dat folder)
"dat init --no-prompt",
// Query the NCBI API for the metadata and returns it in Newline Delimited JSON format
"bionode ncbi search genome eukaryota",
// Store the data in the dat repository
"dat import --json"
```

You can look at the data that got stored by doing ```dat listen``` (then go to localhost:6461 in your browser) or ```dat cat | head``` while in the same folder where the ```.dat``` folder is located.

### Search NCBI for our raw data

```bash
npm run search
```
#### What is happening?
Like the previous gasket command, this runs another pipeline inside the [package.json](https://github.com/bionode/bionode-example-dat-gasket/blob/master/package.json) file.

Here's the description of that pipeline:

```javascript
// Output all the data in the .dat repository
"dat cat",
// Collect only the JSON object that matches Guillardia
"grep Guillardia",
// Extract the value of the assemblyid property
"tool-stream extractProperty assemblyid",
// Download that genome assembly, it is the reference genome
// http://en.wikipedia.org/wiki/Genome_project#Genome_assembly
// http://en.wikipedia.org/wiki/Reference_genome
"bionode ncbi download assembly",
// Wait for the download to complete
"tool-stream collectMatch status completed ",
// Get the NCBI unique ID for this assembly again
"tool-stream extractProperty uid",
// Query the NCBI API to get the UID of the project related to this assembly
"bionode ncbi link assembly bioproject ",
// Extract the UID of the project
"tool-stream extractProperty destUID ",
// Query NCBI to get the UIDs of the related Sequence Read Archive (SRA) files
// http://www.ncbi.nlm.nih.gov/books/NBK47539/
"bionode ncbi link bioproject sra ",
// Extract the SRAs UIDs
"tool-stream extractProperty destUID"
// **Cheating** To have this example run in minutes and not hours/day,
// we will only download the smallest dataset by using its unique ID that
// we figured out in advance. In a normal situation, the following command
// would not be here.
"grep 35526"
```

### [Align](http://en.wikipedia.org/wiki/Sequence_alignment) the [*Guillardia theta*](http://en.wikipedia.org/wiki/Guillardia) genomic sequences

```bash
// Download the SRA file
"bionode ncbi download sra",
// Extract a FASTQ file containing the DNA sequences from the SRA file
"bionode sra fastq-dump",
// Get the path of the FASTQ file
"tool-stream extractProperty destFile",
// Map the DNA sequences to the reference genome
// http://en.wikipedia.org/wiki/Sequence_alignment
// http://en.wikipedia.org/wiki/List_of_sequence_alignment_software#Short-Read_Sequence_Alignment
"bionode bwa mem **/*fna.gz",
// Wait for the previous step to finish and produce a Sequence Alignment/Map (SAM) file
"tool-stream collectMatch status finished",
// Get the path of the SAM file
"tool-stream extractProperty sam",
// Convert the SAM file to a binary format (BAM)
"bionode sam"

Once you have a BAM file, you can view which [scaffold](http://en.wikipedia.org/wiki/Contig#Sequence_contigs) of the reference genomes has more [reads](http://www.k.u-tokyo.ac.jp/pros-e/person/shinichi_morishita/genome-assembly.jpg) mapped to it
```bash
samtools idxstats 35526/SRR070675.bam | sort -k3 -n
```
Then, you can look for variants ([SNPs](http://en.wikipedia.org/wiki/Single-nucleotide_polymorphism)) in that scaffold (JH993052.1)
```bash
gunzip 503988/GCA_000315625.1_Guith1_genomic.fna.gz
samtools mpileup -uf 503988/GCA_000315625.1_Guith1_genomic.fna 35526/SRR070675.bam | bcftools view -v snps - | grep JH993052.1
```
There are several variants, we can start to have a look at the last one by using the [Text Alignment Viewer (tview)](http://samtools.sourceforge.net/tview.shtml)
```bash
samtools tview -p JH993052.1:31185 35526/SRR070675.bam 503988/GCA_000315625.1_Guith1_genomic.fna
# Press ? for help
```
