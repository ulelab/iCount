********
Tutorial
********

**iCount** provides all commands needed to process FASTQ files with iCLIP sequencing data and
generate BED files listing identified and quantified cross-linked sites.

**iCount** uses and generates a number of files. We suggest you run this tutorial in an empty
folder::

    $ mkdir tutorial_example
    $ cd tutorial_example


Preparing a genome index
========================

iCLIP sequencing reads must be mapped to a reference genome. The user can prepare its own
:func:`FASTA genome sequence <iCount.genomes.ensembl.genome>` and
:func:`GTF genome annotation <iCount.genomes.ensembl.annotation>` files.

Another option is to download a release from `ensembl`_. You can use the command ``releases`` to
get a list of available releases supported by **iCount**::

    $ iCount releases

    There are 26 releases available: 84,83,82,81,80,79,78,77,76,75,74,73,72,
    71,70,69,68,67,66,65,64,63,62,61,60,59


You can then use the command ``species`` to get a list of species available in a release::

    $ iCount species -r 84

    There are 70 species available: ailuropoda_melanoleuca,anas_platyrhynchos,
    ancestral_alleles,anolis_carolinensis,astyanax_mexicanus,bos_taurus,
    ...
    gorilla_gorilla,homo_sapiens,ictidomys_tridecemlineatus,latimeria_chalumnae,
    ...
    tursiops_truncatus,vicugna_pacos,xenopus_tropicalis,xiphophorus_maculatus

.. note::
    Current version of iCount is tested to work with human and rat genomes only.

Let's download the human genome sequence from release 84::

    $ iCount genome homo_sapiens -r 84 --chromosomes 21 MT

    Downloading FASTA file into: /..././homo_sapiens.84.chr21_MT.fa.gz
    Fai file saved to : /..././iCount/homo_sapiens.84.chr21_MT.fa.gz.fai
    Done.

.. note::
    Processing the entire genome is computationally very expensive. For this reason, we are
    limiting the tutorial example to chromosomes 21 and MT.

And the annotation of the human genome from release 84::

    $ iCount annotation homo_sapiens -r 84

    Downloading GTF to: /..././homo_sapiens.84.gtf.gz
    Done.

The next step is to generate a genome index that is used by `STAR`_ mapper. Let's call the index
``hs84`` and use ensembl's GTF annotation on genes::

    $ mkdir hs84  # folder should be empty
    $ iCount mapindex homo_sapiens.84.chr21_MT.fa.gz hs84 \
    --annotation homo_sapiens.84.gtf.gz

    Building genome index with STAR for genome homo_sapiens.84.fa.gz
    <timestamp> ..... Started STAR run
    <timestamp> ... Starting to generate Genome files
    <timestamp> ... starting to sort  Suffix Array. This may take a long time...
    <timestamp> ... sorting Suffix Array chunks and saving them to disk...
    <timestamp> ... loading chunks from disk, packing SA...
    <timestamp> ... Finished generating suffix array
    <timestamp> ... starting to generate Suffix Array index...
    <timestamp> ..... Processing annotations GTF
    <timestamp> ..... Inserting junctions into the genome indices
    <timestamp> ... writing Genome to disk ...
    <timestamp> ... writing Suffix Array to disk ...
    <timestamp> ... writing SAindex to disk
    <timestamp> ..... Finished successfully
    Done.

.. note::
    A subfolder ``hs84`` will be created in current working directory. You can specify
    alternative relative or absolute paths, e.g., ``indexes/hs84``.

We are now ready to start mapping iCLIP data to the human genome!

.. _`ensembl`:
    https://www.ensembl.org

.. _`STAR`:
    https://github.com/alexdobin/STAR


Preparing iCLIP data for mapping
================================

Let's process one of the *hnRNP C* sequencing data files from the original `iCLIP publication`_::

    $ wget http://icount.fri.uni-lj.si/data/20101116_LUjh03/\
    SLX-2605.CRIRUN_501.s_4.sequence.txt.gz -O hnRNPC.fq.gz

This is a single file that contains five iCLIP experiments. Each experiment is marked with a
unique barcode sequence at the very beginning of the sequencing reads. Part of the barcode are
also so-called randomer nucleotides that are used to identify unique cDNA molecules after mapping.

We can extract the sample assignment and randomer sequence with the command ``demultiplex``. The
command expects the adapter sequence AGATCGGAAGAGCGGTTCAG, followed by the sample barcodes, in our
case five, expected to be present in the sequencing file::

    $ mkdir demultiplexed  # make sure that folder exists
    $ iCount demultiplex hnRNPC.fq.gz AGATCGGAAGAGCGGTTCAG NNNGGTTNN NNNTTGTNN \
    NNNCAATNN NNNACCTNN NNNGGCGNN --out_dir "demultiplexed"

    Allowing max 1 mismatches in barcodes.
    Demultiplexing file: hnRNPC.fq.gz
    Saving results to:
        demultiplexed/demux_nomatch_raw.fastq.gz
        demultiplexed/demux_NNNGGTTNN_raw.fastq.gz
        demultiplexed/demux_NNNTTGTNN_raw.fastq.gz
        demultiplexed/demux_NNNCAATNN_raw.fastq.gz
        demultiplexed/demux_NNNACCTNN_raw.fastq.gz
        demultiplexed/demux_NNNGGCGNN_raw.fastq.gz
    Trimming adapters (discarding shorter than 15)...

.. note::
    Position of a randomer nucleotide in barcode is indicated with the letter ``N``.


This should have generated six files in subfolder demultiplexed::

    $ ls -lh demultiplexed

    total 704M
    -rw-r--r-- 1 <user> <group>  30M <timestamp> demux_NNNACCTNN.fastq.gz
    -rw-r--r-- 1 <user> <group> 110M <timestamp> demux_NNNCAATNN.fastq.gz
    -rw-r--r-- 1 <user> <group> 311M <timestamp> demux_NNNGGCGNN.fastq.gz
    -rw-r--r-- 1 <user> <group>  16M <timestamp> demux_NNNGGTTNN.fastq.gz
    -rw-r--r-- 1 <user> <group> 188M <timestamp> demux_NNNTTGTNN.fastq.gz
    -rw-r--r-- 1 <user> <group>  52M <timestamp> demux_nomatch.fastq.gz

.. note::
    Reads that cannot be assigned to any of the specified sample barcodes (for the given number of
    allowed mismatches) are stored in a separate file named ``demux_nomatch.fastq.gz``. You
    should have a look at such reads and try to understand why they do not conform to expectations.


.. _`iCLIP publication`:
    https://www.ncbi.nlm.nih.gov/pubmed/20601959


Mapping sample reads to the genome
==================================

Let's focus on iCLIP experiment with barcode **NNNACCTNN** and process it further. Same steps
should be taken to process each experiment.

First, create a folder to store the mapping results::

    $ mkdir mapping_NNNACCTNN

Then, map the reads in the selected FASTQ file using STAR and the genome index we have generated
at the very beginning of this tutorial::

    $ iCount map demultiplexed/demux_NNNACCTNN.fastq.gz hs84 mapping_NNNACCTNN \
    --annotation homo_sapiens.84.gtf.gz

    Mapping reads from demultiplexed/demux_NNNACCTNN.fastq.gz
    <timestamp> ..... Started STAR run
    <timestamp> ..... Loading genome
    <timestamp> ..... Processing annotations GTF
    <timestamp> ..... Inserting junctions into the genome indices
    <timestamp> ..... Started mapping
    <timestamp> ..... Started sorting BAM
    <timestamp> ..... Finished successfully
    Done.

This should have generated a file ``Aligned.sortedByCoord.out.bam`` in folder ``map_NNNNACCTNN``::

    $ ls -lh map_NNNNACCTNN

    total 842M
    -rw-r--r-- 1 <user> <group>  50M Nov 15 05:28 Aligned.sortedByCoord.out.bam
    -rw-r--r-- 1 <user> <group> 1.7K Nov 15 05:28 Log.final.out
    -rw-r--r-- 1 <user> <group> 792M Nov 15 05:28 Log.out
    -rw-r--r-- 1 <user> <group>  718 Nov 15 05:28 Log.progress.out
    -rw-r--r-- 1 <user> <group> 165K Nov 15 05:28 SJ.out.tab
    drwxr-xr-x 1 <user> <group>  306 Nov 15 05:18 _STARgenome

Quantifying cross-linked sites
==============================

Command ``xlsites`` reads a BAM file and generates a BED file with identified and quantified
cross-linked sites::

    $ iCount xlsites mapping_NNNACCTNN/Aligned.sortedByCoord.out.bam \
    NNNACCTNN_cDNA_unique.bed  NNNACCTNN_cDNA_multiple.bed NNNACCTNN_cDNA_skipped.bam \
    --group_by start --quant cDNA

This will generate a BED file where interaction strength is measured by the number of unique
cDNA molecules (randomer barcodes are used for this quantification).

You may generate a BED files where interaction strength is determined by the number of reads::

    $ iCount xlsites mapping_NNNACCTNN/Aligned.sortedByCoord.out.bam \
    NNNACCTNN_reads_unique.bed  NNNACCTNN_reads_multiple.bed NNNACCTNN_reads_skipped.bam \
    --group_by start --quant reads

By comparing the ration of cDNA vs reads counts we can estimate the level of over-amplification.
Ideally, this ratio should be close to one.


Identifying significantly cross-linked sites
============================================

The peak finding analysis expects an annotation file with information about the segmentation of
the genome into regions of different types, such as intergenic, UTR3, UTR5, ncRNA, intron, CDS
regions.

Command ``segment`` can read the annotation obtained from `ensembl`_ and generate a new
annotation file with genome segmentation::

    $ iCount segment homo_sapiens.84.gtf.gz hs84seg.gtf.gz \
    homo_sapiens.84.chr21_MT.fa.gz.fai

    Calculating intergenic regions...
    Segmentation stored in hs84seg.gtf.gz

Command ``peaks`` reads a genome segmentation GTF file, a BED file with cross-linked sites and
generates a BED file with subset of significantly cross-linked sites::

    $ iCount peaks hs84seg.gtf.gz NNNACCTNN_cDNA_unique.bed peaks.bed \
    --scores scores.tsv

    Loading annotation file...
    918 out of 31271 annotation records will be used (30353 skipped).
    Loading cross-links file...
    Calculating intersection between annotation and cross-link file...
    Processing intersections...
    Peaks caculation finished. Writing results to files...
    BED6 file with significant peaks saved to: peaks.bed
    Scores for each cross-linked position saved to: scores.tsv
    Done.

.. note::
    P-value and FDR scores of all cross-linked sites can be stored by providing the parameter ``--scores``.


Generating RNA maps
===================

TODO


Identifying enriched k-mers
===========================

TODO
