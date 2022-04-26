# Benchmarking OpenOmics on AWS
Step by step commands to benchmark OpenOmics framework on AWS

1.	Log in to your AWS account
2.	Launch a virtual machine with EC2
  * Choose an Amazon Machine Image (AMI): Select any 64-bit (x86) AMI  (say, Ubuntu Server 20.04 LTS) from “Quick Start”.
  * Choose an Instance Type.
  * Configure Instance: We recommended using dedicated tenancy and hyperthreading enabled.
  * Add Storage: You can add storage based on the workload requirements
  * Configure security group 
  * Review and launch the instance (ensure you have/create a key to ssh login in next step)
3.	Use SSH to login to the machine after the instance is up and running
  * $ ssh -i <key.pem> username@Public-DNS
4.	The logged in AWS instance machine is now ready to use – you can download OpenOmics workloads and related datasets to be executed on this instance.


# Step by step instructions to benchmark baseline (bwa-mem) and OpenOmics BWA-MEM (bwa-mem2) on c5.24xlarge and m6i.16xlarge instances of AWS
## Step 1: Download datasets
Download reference genome: Homo_sapiens_assembly38.fasta
```sh
wget https://storage.googleapis.com/genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta
```

Download dataset #1: ERR194147 (Paired-End) from 
s3://sra-pub-run-odp/sra/ERR194147/ERR194147

Download dataset #2: ERR1955529 (Paired-End) from 
s3://sra-pub-run-odp/sra/ERR1955529/ERR1955529

Download dataset #3: ERR3239276 (Paired-End) from 
s3://sra-pub-run-odp/sra/ERR3239276/ERR3239276

## Step 2: Download and compile baseline (BWA v0.7.17) and create index
```sh
curl -L https://github.com/lh3/bwa/releases/download/v0.7.17/bwa-0.7.17.tar.bz2 | tar jxf -
cd bwa-0.7.17 && make
./bwa index hs_asm38/Homo_sapiens_assembly38.fasta
cd ..
```

## Step 3: Download OpenOmics BWA-MEM (BWA-MEM2 v2.2.1) and create index
```sh
curl -L https://github.com/bwa-mem2/bwa-mem2/releases/download/v2.2.1/bwa-mem2-2.2.1_x64-linux.tar.bz2 | tar jxf -
cd bwa-mem2-2.2.1_x86-linux && ./bwa-mem2 index hs_asm38/Homo_sapiens_assembly38.fasta
cd ..
```

## Step 4: Running baseline and OpenOmics BWA-MEM
The c5.24xlarge instance has 48 cores. The memory available on the instance only allows for using 1 thread per core.
On the other hand, the m6i.16xlarge instance has 32 cores. The memory available on the instance allows for using both the threads on each core.

### Run baseline BWA-MEM
```sh
cd bwa-0.7.17
```
Sample commands shown below:\
For c5.24xlarge:
```sh
./bwa mem -t 48 hs_asm38/Homo_sapiens_assembly38.fasta ERR194147_1.fastq ERR194147_2.fastq > ERR194147.out.sam
```
For m6i.16xlarge:
```sh
./bwa mem -t 64 hs_asm38/Homo_sapiens_assembly38.fasta ERR194147_1.fastq ERR194147_2.fastq > ERR194147.out.sam
```

### Run OpenOmics BWA-MEM
```sh
cd ../bwa-mem2-2.2.1_x86-linux
```
Sample commands shown below:\
For c5.24xlarge:
```sh
./bwa-mem2 mem -t 48 hs_asm38/Homo_sapiens_assembly38.fasta ERR194147_1.fastq ERR194147_2.fastq > ERR194147.out.sam
```
For m6i.16xlarge:
```sh
./bwa-mem2 mem -t 64 hs_asm38/Homo_sapiens_assembly38.fasta ERR194147_1.fastq ERR194147_2.fastq > ERR194147.out.sam
```


# Step by step instructions to benchmark baseline (minimap2) and OpenOmics minimap2 (mm2-fast) on c5.12xlarge, c6i.16xlarge and m6i.16xlarge instances of AWS

## Step 1: Download datasets
Download reference genome
```sh
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/references/GRCh38/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz
```
Link to download HG002 ONT Guppy 3.6.0 dataset:
https://precision.fda.gov/challenges/10/view \
File name: HG002_GM24385_1_2_3_Guppy_3.6.0_prom.fastq.gz

Link to download HG002 HiFi 14kb-15kb dataset:
https://precision.fda.gov/challenges/10/view \
File name: HG002_35x_PacBio_14kb-15kb.fastq.gz

Download HG002 CLR dataset from 
s3://giab/data_indexes/AshkenazimTrio/sequence.index.AJtrio_PacBio_MtSinai_NIST_subreads_fasta_10082018

Download hap2 assembly dataset- 
```sh
wget https://zenodo.org/record/4393631/files/NA24385.HiFi.hifiasm-0.12.hap2.fa.gz
```

## Step 2: Download and compile baseline (minimap2 v0.2.22)
```sh
git clone https://github.com/lh3/minimap2.git -b v2.22
cd minimap2 && make
```

## Step 3: Run baseline minimap2
```sh
./minimap2 -ax [preset] [ref-seq] [read-seq] -t 48 > minimap2output
```
Example command for ONT HG002 dataset:
```sh
./minimap2 -ax map-ont  GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz HG002_ONT.fastq -t 48 > minimap2output
```

## Step 3: Download and compile OpenOmics minimap2 (mm2-fast)
```sh
git clone --recursive https://github.com/lh3/minimap2.git -b fast-contrib-v2.22 mm2-fast-contrib
cd mm2-fast-contrib && make multi
```

## Step 4: Create index for OpenOmics minimap2
```sh
./build rmi.sh path-to-ref-seq <preset flags>
<preset flags> are as follows:
ONT: map-ont
HiFi: map-hifi
CLR: map-pb
Assembly: asm5
```
Example: Create OpenOmics minimap2 index for ONT datasets for GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz
```sh
./build rmi.sh GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz map-ont
```

## Step 5: Run OpenOmics minimap2
```sh
./mm2-fast -ax [preset] [ref-seq] [read-seq] -t [num_threads] > mm2-fastoutput
```
Example command to run HG002 ONT dataset on c5.12xlarge
```sh
./mm2-fast -ax map-ont GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz HG002_ONT.fastq -t 48 > mm2-fastoutput
```
Example command to run HG002 ONT dataset on c6i.16xlarge or m6i.16xlarge
```sh
./mm2-fast -ax map-ont GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.gz HG002_ONT.fastq -t 64 > mm2-fastoutput
```

