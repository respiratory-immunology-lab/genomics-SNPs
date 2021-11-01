# genomics-SNPs

conda create -n OVarFlow

conda activate OVarFlow

wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/OVarFlow_dependencies_mini.yml

conda install -c bioconda snpeff
SnpSift filter "(ANN[*] has 'HIGH') | (ANN[*] has 'MODERATE')" variants_annotated.vcf.gz > high_moderate_impact.vcf.gz
    "(Cases[0] = 3) & (Controls[0] = 0) & ((ANN[*].IMPACT = 'HIGH') | (ANN[*].IMPACT = 'MODERATE'))" \


https://www.biorxiv.org/content/10.1101/2020.12.10.419663v1.full


conda env update --file OVarFlow_dependencies_mini.yml
rm -rf  OVarFlow_dependencies_mini.yml

mkdir [var_calling_project]
cd [var_calling_project]
wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/Snakefile
wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/samples_and_read_groups.csv
wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/config.yaml

mkdir scripts
cd scripts 
wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/scripts/average_coverage.awk
wget https://gitlab.com/computational-biology/ovarflow/-/raw/master/OVarFlow_src/scripts/createIntervalLists.py

chmod +x average_coverage.awk
chmod +x createIntervalLists.py

snakemake -np

cd REFERENCE_INPUT_DIR
wget http://ftp.ensembl.org/pub/release-98/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna_sm.primary_assembly.fa.gz
wget http://ftp.ensembl.org/pub/release-98/gff3/homo_sapiens/Homo_sapiens.GRCh38.98.gff3.gz

# Get the sample names quickly
for f in *_R1.fastq.gz; do Basename=${f%_merged_*}; echo $Basename; done

# Create CSV
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow --account=of33 --time=01:00:00 --mem-per-cpu=1G --ntasks=1 --cpus-per-task=1 --partition=genomics --qos=genomics" -j 2 -w 30 -p create_CSV_template

# Update .csv file with sample information and other

# Create annotation file
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow --account=of33 --time=04:00:00 --mem-per-cpu=24G --ntasks=1 --cpus-per-task=6 --partition=genomics --qos=genomics" -j 2 -w 60 -p create_snpEff_db

RUN PIPELINE

# Run OVarFlow on comp partition
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow_Monday --account=of33 --time=2-00:00:00 --mem=30G --nodes=1 --ntasks=1 --cpus-per-task=4" -j 50 -w 30 -p

# Step 1
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow_QC --account=of33 --time=04:00:00 --mem=30G --ntasks=1 --cpus-per-task=4 --partition=genomics --qos=genomics" -j 50 -w 30 -p QC_all

# Step 2
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow_mapping --account=of33 --time=04:00:00 --mem=120G --nodes=1 --ntasks=1 --cpus-per-task=24 --partition=genomics --qos=genomics" -j 10 -w 30 -p mapping_all

# Step 3
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow_STEP3 --account=of33 --time=04:00:00 --mem=120G --nodes=1 --ntasks=1 --cpus-per-task=4 --partition=genomics --qos=genomics" -j 50 -w 30 -p sortingmarking_all

# Step 4
snakemake --configfile config.yaml --cluster "sbatch --job-name=OVarFlow_STEP4 --account=of33 --time=04:00:00 --mem=120G --nodes=1 --ntasks=1 --cpus-per-task=4 --partition=genomics --qos=genomics" -j 50 -w 30 -p genotyping_all
