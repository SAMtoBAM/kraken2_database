# kraken2_database
building a kraken2 db with public genomes but not the huge prebuilt databases


```

##set up conda env
#mamba create -n kraken2 -c bioconda -c conda-forge kraken2
conda activate kraken2

##the most difficult and important step with kraken is the database of kmers to be used
##these databases are built using NCBI sequences and taxonomic assignments
##there is a standard databases but it is purely designed for human data
##therefore we need to create another
##because we are looking at species across different domains/kingdoms we need a vast database covering several very divergent clades
##the caveat is this database takes up a lot of space

##here we will try build out a database using the prebuilt protozoa library (the smallest) then add to it manually
##calling the database 'protozoa_plus'
##set up for all the protozoa genomes
kraken2-build --download-library protozoa --db protozoa_plus --no-masking
##also download the ncbi taxonomy
kraken2-build --download-taxonomy --db protozoa_plus

##now we can select a set of genomes of interest and others to fill out all diversity possible
##need to select a set of genomes of our own using a subset of bacterial, fungal genomes etc
##first need to create an env for the ncbi command line library 
#mamba create -n ncbi_datasets
#conda activate ncbi_datasets
#mamba install -c conda-forge ncbi-datasets-cli

conda activate ncbi_datasets
##need taxid IDs for each genome (for kraken2) so we can download the genomes in a loop whilst adding the taxid i.d. as 'kraken:taxid|XXX' to each header e.g. '>contig1|kraken:taxid|XXX'
mkdir manual_genomes
###to do it by species (only taking the reference genome, of which there can be multiple such as in E.coli)
###the fasta file headers are then renamed using the ncbi taxon id (taxid) as given in the jsonl file with the assembly
for species in 'Saccharomyces cerevisiae' 'Escherichia coli' 'Bacillus subtilis' 'Mycoplasma genitalium' 'Bacteroides thetaiotaomicron' 'Pseudomonas fluorescens' 'Azotobacter vinelandii' 'Halobacterium salinarum' ' Pyrococcus abyssi' 'Neurospora crassa' 'Aspergillus oryzae' 'Chlamydomonas reinhardtii' 'Drosophila melanogaster' 'Danio rerio'
do
echo ${species}
datasets download genome taxon "${species}" --assembly-source Refseq --assembly-version latest --include genome --reference
unzip ncbi_dataset.zip
rm ncbi_dataset.zip
dataformat tsv genome --fields accession,current-accession,organism-infraspecific-isolate,organism-infraspecific-strain,organism-name,organism-tax-id --inputfile ncbi_dataset/data/assembly_data_report.jsonl > temp
cat temp | awk -F "\t" '{print $1}' | tail -n+2 | while read accession
do
taxid=$( cat temp | awk -v acc="$accession" -F "\t" '{if($1 == acc) print $6}' )
species2=$( echo ${species} | sed 's/ /_/' )
##only take one of the reference assemblies it multiple
cat ncbi_dataset/data/${accession}/*.fna | awk -F " " -v taxid="$taxid" '{if($1 ~ ">") {print $1"|kraken:taxid|"taxid} else {print}}' > manual_genomes/${accession}.${species2}.fa
done
rm -r ncbi_dataset
rm md5sum.txt
rm README.md
rm temp
done

##need to add them to a pre-existing database
conda activate kraken2
ls manual_genomes/ | while read genome
do
kraken2-build --add-to-library manual_genomes/${genome} --db protozoa_plus --no-masking
done

##remove any previous builds (will skip building otherwise and not add new genomes)
rm protozoa_plus/*.map
rm protozoa_plus/*.k2d

##now actually compile the kraken db
kraken2-build --build --db protozoa_plus


##now search the database using a genome assembly
kraken2 --output kraken2_results --use-names --db protozoa_plus SEQUENCES.fa

```
