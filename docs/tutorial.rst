==============
CoPTR Tutorial
==============

This page contains a worked example for using CoPTR. It takes you
through the steps in the :doc:`Quick Start <quick-start>`.
The data used in this example is available here:

``https://dl.dropboxusercontent.com/s/wrsxjimr6l96lcq/example-data.tar.gz``

.. contents::
    :depth: 2


Building the reference database
===============================

The first step for running CoPTR is to create a reference database from
fasta files. CoPTR is designed to run on both complete reference genomes
and draft assemblies. Each genome should be in its own fasta file. A
complete genome has a single sequence, while assemblies have multiple
sequences---one for each contig.

Precomputed reference databases can be downloaded here:
:doc:`Reference Databases <databases>`

If you want to construct your own reference database, please use the
guidelines below.


Cluster genomes at 95% ANI
--------------------------
CoPTR estimates PTRs at the species level: estimates among strains within 95%
ANI are consistent. This means you should cluster genomes by 95% ANI, an
operational defintion of a species, and select one representative genome per
species.

Select only high-quality assemblies or complete genomes
-------------------------------------------------------
Metagenomic assembly is a challenging problem, and assemblies are not guaranteed
to be correct. We set some basic criteria for using an assembly:

* It is estimated to be >90% complete
* It is estimated to have <5% contamination

The first two criteria are similar to the MIMAG criteria for high-quality
assemblies.

When choosing between an assembly, and complete reference genome, preference
should always be given to the complete genome.



Indexing the reference database
===============================

**On the example data:**

.. code-block:: text

    $ coptr index example-data/ref-db example-data/ref-db/example-index
    [INFO] [Jun 28, 2021 15:46:57] [coptr.read_mapper] Found 2 files totaling 0.00611 GB.
    [INFO] [Jun 28, 2021 15:46:57] [coptr.read_mapper] Copying FASTA files to coptr-fna-2021-06-28T19:46:57+00:00.fna with prepended genome ids (filenames).
    [INFO] [Jun 28, 2021 15:46:57] [coptr.read_mapper] Writing 2 reference genome ids to example-data/ref-db/example-index.genomes.
    [INFO] [Jun 28, 2021 15:46:57] [coptr.read_mapper] bowtie2-build coptr-fna-2021-06-28T19:46:57+00:00.fna example-data/ref-db/example-index --threads 1
    ...bowtie2 output...
    [INFO] [Jun 28, 2021 15:47:00] [coptr.read_mapper] Indexed 2 FASTA files for the reference database.
    [INFO] [Jun 28, 2021 15:47:00] [coptr.read_mapper] Cleaning up coptr-fna-2021-06-28T19:46:57+00:00.fna.



CoPTR provides a few utilities to make indexing the database with bowtie2
easier. It assumes the reference database is provided by a folder of fasta
files, one per reference genome. Reference genomes are combined in a single
file, with the filename prepended to each sequencing id. This is how CoPTR
keeps track of contigs from the same assembly.

Indexing a database of fasta files can be accomplished by calling
``coptr index``:

.. code-block:: text

    usage: coptr index [-h] [--bt2-bmax BT2_BMAX] [--bt2-dcv BT2_DCV] [--bt2-threads BT2_THREADS] [--bt2-packed] ref-fasta index-out

    positional arguments:
      ref_fasta             File or folder containing fasta to index. If a folder,
                            the extension for each fasta must be one of [.fasta,
                            .fna, .fa]
      index_out             Filepath to store index.

    optional arguments:
      -h, --help            show this help message and exit
      --bt2-bmax BT2_BMAX   Set the --bmax arguement for bowtie2-build. Used to
                            control memory useage.
      --bt2-dcv BT2_DCV     Set the --dcv argument for bowtie2-build. Used to
                            control memory usage.
      --bt2-threads BT2_THREADS
                            Number of threads to pass to bowtie2-build.
      --bt2-packed          Set the --packed flag for bowtie2-build. Used to
                            control memory usage.

It takes two arguments. The first ``ref-fasta`` is either a folder containing
fasta for the database. If it is a folder, CoPTR will scan the folder for
all files ending in ``.fasta``, ``.fna``, or ``.fa``. CoPTR will combine these
into a single fasta file, prepending the filename to each sequence id in order
to track contigs from the same reference genome. It then calls the ``bowtie2-build``
to index this file.

coPTR additionally outputs an ``index_name.genomes`` file with a list of ids for each
reference genome.

Notes on memory usage
---------------------
Sometimes the database is too large for a single index. You can split the
reference database into multiple parts, and index each separately. After
read mapping, the resulting BAM files can be merged with ```coptr merge```.

Mapping reads
=============

**On the example data:**

.. code-block:: text

    $ coptr map example-data/ref-db/example-index example-data/fastq example-data/bam
    [INFO] [Jun 28, 2021 15:48:26] [coptr.read_mapper] Mapping example-data/fastq to example-data/bam/ERR969281.sam
    [INFO] [Jun 28, 2021 15:48:26] [coptr.read_mapper] bowtie2 -x example-data/ref-db/example-index example-data/fastq/ERR969281.fastq.gz --no-unal -p 1 -k 10
    10818 reads; of these:
      10818 (100.00%) were unpaired; of these:
        4071 (37.63%) aligned 0 times
        6709 (62.02%) aligned exactly 1 time
        38 (0.35%) aligned >1 times
    62.37% overall alignment rate
    [INFO] [Jun 28, 2021 15:48:27] [coptr.read_mapper] Converting example-data/bam/ERR969281.sam to example-data/bam/ERR969281.bam.
    [INFO] [Jun 28, 2021 15:48:27] [coptr.read_mapper] Cleaning up example-data/bam/ERR969281.sam.
    ....
    [INFO] [Jun 28, 2021 15:48:40] [coptr.read_mapper] Converting example-data/bam/ERR969430.sam to example-data/bam/ERR969430.bam.
    [INFO] [Jun 28, 2021 15:48:40] [coptr.read_mapper] Cleaning up example-data/bam/ERR969430.sam.


Once you have indexed a reference database. You can then map reads against
the database. CoPTR provides a wrapper around bowtie2 to make read mapping
convenient:

.. code-block:: text

    usage: coptr map [-h] [--threads INT] [--paired] index input out-folder

    positional arguments:
      index              Name of database index.
      input              File or folder containing fastq reads to map. If a
                         folder, the extension for each fastq must be one of
                         [.fastq, .fq, .fastq.gz, fq.gz]
      out_folder         Folder to save mapped reads. BAM files are output here.

    optional arguments:
      -h, --help         show this help message and exit
      --paired           Set for paired end reads. Assumes fastq files end in _1.*
                         and _2.*
      --threads THREADS  Number of threads for bowtie2 mapping.
      --bt2-k BT2_K      (Default 10). Number of alignments to report. Passed to -k flag of
                         bowtie2.

The name of the database index corresponds to the name used from ``coptr index``.
The input can either be a single fastq file, or a folder of fastq files to map.
It also takes an optional ``--threads`` argument that allows bowtie2 to use
multiple threads. Reads are output as ``bam`` files to save space.

For paired-end sequencing, we recommand mapping reads from end only (e.g. the files
ending in either _1.* or _2.*).


Merging mapped reads from multiple indexes
------------------------------------------
For large reference databases, it is sometimes necessary to create several
indexes for subsets of the data and map reads against each index. Results
from each index need to be merged to select reads with the best MAPQ across
indexes. You can use ```coptr merge``` to merge multiple bam files.

.. code-block:: text

    usage: coptr merge [-h] in-bam1 in-bam2 ... in-bamN out-bam

    positional arguments:
      in-bams     A space separateed list of BAM files to merge. Assumes same
                  reads were mapped against different indexes.
      out-bam     Path to merged BAM.

    optional arguments:
      -h, --help  show this help message and exit


Computing coverage maps
=======================
**On the example data:**

.. code-block:: text

    $ coptr extract example-data/bam example-data/coverage-maps
    [INFO] [Jun 28, 2021 15:49:48] [coptr.bam_processor] Processing example-data/bam/ERR969281.bam.
    [INFO] [Jun 28, 2021 15:49:48] [coptr.bam_processor] Determining reference genomes.
    [INFO] [Jun 28, 2021 15:49:48] [coptr.bam_processor] Collecting multi-mapped reads.
    [INFO] [Jun 28, 2021 15:49:48] [coptr.bam_processor] Grouping reads by reference genome.
    ...
    [INFO] [Jun 28, 2021 15:50:00] [coptr.cli] Found 190 reference sequences corresponding to 2 genomes.

Once reads have been mapped, the next step is to compute the coverage along
each reference genome. In this step, starting positions of each read are
extracted from each bam file, and reads from different contigs of the same
assembly are collected.

.. code-block:: text

    usage: usage: coptr extract [-h] [--ref-genome-regex REF_GENOME_REGEX] [--check-regex]
                    in-folder out-folder

    positional arguments:
      in_folder             Folder with BAM files.
      out_folder            Folder to store coverage maps.

    optional arguments:
      -h, --help            show this help message and exit
      --ref-genome-regex REF_GENOME_REGEX
                            Regular expression extracting a reference genome id
                            from the sequence id in a bam file.
      --check-regex         Check the regular expression by counting reference
                            genomes without processing

The important argument here is the ``--ref-genome-regex``. This is a regular
expression that extracts the reference genome id from a sequence id. The default
argument will work with the index created by ```coptr index```, and works by
prepending the name of the fasta file, and special character ```|``` to each
sequence id.

Using other BAM files
---------------------
If you already have BAM files that were not computed with CoPTR, you will need
to set the ``--ref-genome-regex`` flag. This flag is a regular expression that
extracts a genome id from a sequence id in a fasta file. It is used to group 
contigs together. The default argument ``[^\|]+`` matches all characters up
to the first ``|``, and uses them as a genome id.

You can check your regular expression using the ``--check-regex`` flag, which
skips the extract step and instead outputs a list of all genome ids.


Estimating PTRs
===============

**On the example data:**

.. code-block:: text

    $ coptr estimate example-data/coverage-maps out --min-reads 2500
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli] Grouping reads by reference genome.
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli] Saving to example-data/coverage-maps/coverage-maps-genome:
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969281.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969282.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969283.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969285.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969286.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969428.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969429.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli]  ERR969430.cm.pkl
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli] Grouping by reference genome: Complete.
    [INFO] [Jun 28, 2021 15:50:15] [coptr.cli] The --restart flag can be used to start from here.
    [INFO] [Jun 28, 2021 15:50:15] [coptr.coptr_ref] Checking reference genomes.
    [INFO] [Jun 28, 2021 15:50:24] [coptr.coptr_ref] Running l-gasseri-ref.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.coptr_ref] Finished l-gasseri-ref.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.coptr_contig] Checking reference genomes.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.coptr_contig] Running e-coli-mag.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.coptr_contig] Finished e-coli-mag.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.cli] Writing out.csv.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.cli] Done.
    [INFO] [Jun 28, 2021 15:50:25] [coptr.cli] You may now remove the folder example-data/coverage-maps/coverage-maps-genome.

The final stage is to estimate PTR ratios from coverage maps. This is accomplished
with the ``estimate`` command. **It is strongly recommended that you perform this step
on all samples at once.**

.. code-block:: text

    usage: coptr estimate [-h] [--min-reads MIN_READS] [--min-cov MIN_COV] [--threads THREADS] coverage-map-folder out-file

    positional arguments:
      coverage_map_folder   Folder with coverage maps computed from 'extract'.
      out_file              Filename to store PTR table.

    optional arguments:
      -h, --help            show this help message and exit
      --min-reads MIN_READS
                            Minimum number of reads required to compute a PTR
                            (default 5000).
      --min-cov MIN_COV     Fraction of nonzero 10Kb bins required to compute a
                            PTR (default 0.75).
      --min-samples MIN_SAMPLES
                            CoPTRContig only. Minimum number of samples required
                            to reorder bins (default 5).

This combines all coverage maps by species, then estimates PTRs for each species.
We have tried to set sensible default parameters for PTR estimatation. We set
the minimum number of reads for the example data to 2500 in order to keep the
size of the example data small, but the default of 5000 reads is recommended.

The output is a CSV file where, the rows are reference genomes, and the
columns are samples. Each entry is the estimated log2 PTR.