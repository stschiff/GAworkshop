Genotype Calling from Bam Files
===============================

In this session we will process the BAM files of your samples to derive genotype calls for a selected set of SNPs. The strategy followed in this workshop will be to analyse (ancient) test samples together with a large set of modern reference populations. While there are several choices on which data set to use as modern reference, here we will use the Human Origins (HO) data set, published in `Lazaridis et al. 2014 <http://www.nature.com/nature/journal/v513/n7518/full/nature13673.html>`_, which consists of more than 2,300 individuals world-wide human populations, which is a good representation of global human diversity. Those samples were genotyped at around 600,000 SNP positions.

For our next step, we therefore need to use the list of SNPs in the HO data set and for each test individual call a genotype at these SNPs if possible. Following `Haak et al. 2015 <http://www.nature.com/nature/journal/v522/n7555/abs/nature14317.html>`_, we will not attempt to call diploid genotypes for our test samples, but will simply pick a single read covering each SNP positions and will represent that sample at that position by a single haploid genotype. Fortunately, the population genetic analysis tools are able to deal with this pseudo-haploid genotype data.

Genotyping the test samples
---------------------------

The main work horse for this genotyping step will be ``simpleBamCaller``, a program I wrote to facilitate this type of calling. You will find it at ``/projects1/tools/workshop/2016/GenomeAnalysisWorkshop/simpleBamCaller``. You should somehow put this program in your path, e.g. by adding the directory to your ``$PATH`` environment variable. An online help for this tool is printed when starting it via ``simpleBamCaller -h``.

A typical command line for chromosome 20 may look like:

.. code-block:: bash

    simpleBamCaller -f $SNP_FILE -c 20 -r $REF_SEQ -o EigenStrat -e $OUT_PREFIX $BAM1 $BAM2 $BAM3 > $OUT_GENO

Let's go through the options one by one:

1. ``-f $SNP_FILE``: This option gives the file with the list of SNPs and alleles. The program expects a three-column format consisting of chromosome, position and the two alleles separated by a comma. For the Human Origins SNP panel we have prepared such a file. You can find it at ``/projects1/users/schiffels/PublicData/HumanOriginsData.backup/EuropeData.positions.txt``. In case you have aligned your samples to HG19, which uses chromosome names start with ``chr``, you need to use the file ``EuropeData.positions.altChromName.txt`` in the same directory.
2. ``-c 20`` The chromosome name. In simpleBamCaller you need to call each chromosome separately. It's anyway recommended to parallelise across chromosomes to speed up calling. Note that if your BAM files are aligned to HG19, you need to give the correct chromosome name, i.e. ``chr20`` in this case. However, note that the Human Origins data set uses just plain numbers as chromosomes, so you want to use the option ``--outChrom 20`` in simpleBamCaller to convert the chromosome name to plain numbers.
3. ``-r $REF_SEQ``. This is the reference sequence that was used to align your bam file to. You find them under ``/projects1/Reference_Genomes``.
4. ``-o EigenStrat``: This specifies that the output should be in EigenStrat format, which is the format of the HO data set, with which we need to merge our test data. EigenStrat formatted data sets consist of three files: i) a SNP file, with the positions and alleles of each SNP
5. ``-e $OUT_PREFIX``. This will be the file prefix for the
6. ``$BAM1 $BAM2 ...`` This are the bam files of your test samples. Note that you will call simultaneously in all samples to generate a joined EigenStrat files for a single chromosome for all samples.

You should try the command line, perhaps piping the result into `head` to only look at the first 10 lines of the genotype-output. You may encounter an error of the sort "inconsistent number of genotypes. Check that bam files have different readgroup sample names". In that case you will first have to fix your bam files (see section below). If you do not encounter this error, you can skip the next section.

.. note:: Fixing the bam read group

  For multi-sample calling, bam files need to contain a certain flag in their header specifying the read group. This flag also contains the name of the sample whose data is contained in the BAM file. Standard mapping pipelines like BWA and EAGER might not add this flag in all cases, so we have to do that manually. Fortunately, it is relatively easy to do this using the Picard software. The `documentation <http://broadinstitute.github.io/picard/command-line-overview.html#AddOrReplaceReadGroups>`_ of the ``AddOrReplaceReadGroups`` tool shows an example command line, which in our case can look like this:

  .. code-block:: bash

    picard AddOrReplaceReadGroups I=$INPUT_BAM O=$OUTPUT_BAM RGLB=$LIB_NAME RGPL=$PLATFORM RGPU=$PLATFORM_UNIT RGSM=$NAME

Here, the only really important parameter is ``RGSM=$NAME``, where ``$NAME`` should be your sample name. The other parameters are required but not terribly important: ``$LIB_NAME`` can be some library name, or you can simply use the same name as your sample name. ``$PLATFORM`` should be ``illumina`` but can be any string, ``$PLATFORM_UNIT`` can be ``unit1`` or something like that.

If you encounter any problems with validation, you can additionally set the parameter ``VALIDATION_STRINGENCY=SILENT``.

In order to run this tool on all your samples you should submit jobs via a shell script, which in my case looked like this:

.. code-block:: bash

    #!/usr/bin/env bash

    BAMDIR=/data/schiffels/MyProject/mergedBams.backup

    for SAMPLE in $(ls $BAMDIR); do
        BAM=$BAMDIR/$SAMPLE/$SAMPLE.mapped.sorted.rmdup.bam
        OUT=$BAMDIR/$SAMPLE/$SAMPLE.mapped.sorted.rmdup.fixedHeader.bam
        CMD="picard AddOrReplaceReadGroups I=$BAM O=$OUT RGLB=$SAMPLE RGPL=illumina RGPU=HiSeq RGSM=$SAMPLE"
        echo "$CMD"
        # sbatch -c 2 -o $OUTDIR/$SAMPLE.readgroups.log --wrap="$CMD"
    done

As in the previous session, write your own script like that, make it executable using ``chmod u+x``, run it, check that the printed commands look correct, and then remove the comment from the `sbatch` line to submit the jobs.

Continuing with genotyping
^^^^^^^^^^^^^^^^^^^^^^^^^^

If the calling pipeline described above works with the fixed bam files, the first lines of the output (using ``head``) should look like this::

    [warning] tag DPR functional, but deprecated. Please switch to `AD` in future.
    [mpileup] 5 samples in 5 input files
    <mpileup> Set max per-file depth to 1600
    00220
    22000
    00220
    20922
    22092
    22090
    20000
    29992
    22292
    20090

The first three lines are just output to stderr (they won't appear in the file when you pipe via ``> $OUT_FILE``). The 10 last lines are the called genotypes on the input SNPs. Here, a 2 denotes the reference allele, 0 denotes the alternative allele, and 9 denotes missing data. If you also look at the first 10 lines of the `*.snp.txt` file, set via the `-e` option above, you should see something like this::

    20_97122	20	0	97122	C	T
    20_98930	20	0	98930	G	A
    20_101362	20	0	101362	G	A
    20_108328	20	0	108328	C	A
    20_126417	20	0	126417	A	G
    20_126914	20	0	126914	C	T
    20_126923	20	0	126923	C	A
    20_127194	20	0	127194	T	C
    20_129063	20	0	129063	G	A
    20_140280	20	0	140280	T	C

which is the EigenStrat output for the SNPs. Here, the second column is the chromosome, the fourth column is the position, and the 5th and 6th are the two alleles. Note that simpleBamCaller automatically restricts the calling to the two alleles given in the input file. EigenStrat output also generates an ``*.ind.txt`` file, again set via the ``-e`` prefix flag. We will look at it later.

OK, so now that you know that it works in principle, you need to again write a little shell script that performs this calling for all samples on all chromosomes. In my case, it looks like this:

.. code-block:: bash

    #!/usr/bin/env bash

    BAMDIR=/data/schiffels/MyProject/mergedBams.backup
    REF=/projects1/Reference_Genomes/Human/hs37d5/hs37d5.fa
    SNP_POS=/projects1/users/schiffels/PublicData/HumanOriginsData.backup/EuropeData.positions.autosomes.txt
    OUTDIR=/data/schiffels/MyProject/genotyping
    mkdir -p $OUTDIR

    BAM_FILES=$(ls $BAMDIR/*/*.mapped.sorted.rmdup.bam | tr '\n' ' ')
    for CHR in {1..22}; do
        OUTPREFIX=$OUTDIR/MyProject.HO.eigenstrat.chr$CHR
        OUTGENO=$OUTPREFIX.geno.txt
        CMD="simpleBamCaller -f $SNP_POS -c $CHR -r $REF -o EigenStrat -e $OUTPREFIX $BAM_FILES > $OUTGENO"
        echo "$CMD"
        # sbatch -c 2 -o $OUTDIR/$SAMPLE.sexDetermination.log --mem=2000 --wrap="$CMD"
    done

Note that I am now looping over 22 chromosomes instead of samples (as we have done in other scripts). The line beginning with ``BAM_FILES=...`` looks a bit cryptic. The syntax ``$(...)`` will put the output of the command in brackets into the ``BAM_FILES`` variable. The ``tr '\n' ' '`` bit takes the listing output from ``ls`` and convert new lines into spaces, such that all bam files are simply put behind each other in the ``simpleBamCaller`` command line. Before you submit, look at the output of this script by piping it into ``less -S``, which will not wrap the very long command lines and allows you to inspect whether all files are given correctly. When you are sure it's correct, remove the comment from the ``sbatch`` line and comment out the ``echo`` line to submit.

.. note:: A word about DNA damage

  If the samples you are analysing are ancient samples, the DNA will likely contain DNA damage, so C->T changes which are seen as C->T and G->A substitutions in the BAM files. There are two ways how to deal with that. First, if your data is not UDG-treated, so if it contains the full damage, you should restrict your analysis to Transversion SNPs only. To that end, simply add the ``-t`` flag to ``simpleBamCaller``, which will automatically output only transversion SNPs. If your data is UDG-treated, you will have much less damage, but you can still see damaged sites in particular in the boundary of the reads in your BAM-file. In that case, you probably want to make a modified bam file for each sample, where the first 2 bases on each end of the read are clipped. A useful tool to do that is `TrimBam <http://genome.sph.umich.edu/wiki/BamUtil:_trimBam>`_, which we will not discuss here, but which I recommend to have a look at if you would like to analyse Transition SNPs from UDG treated libraries.

Merging across chromosomes
^^^^^^^^^^^^^^^^^^^^^^^^^^

Since the EigenStrat format consists of simple text files, where rows denote SNPs, we can simply merge across all chromosomes using the UNIX ``cat`` program. If you ``cd`` to the directory containing the eigenstrat output files for all chromosomes and run ``ls`` you should see something like::

    MyProject.HO.eigenstrat.chr10.geno.txt
    MyProject.HO.eigenstrat.chr10.ind.txt
    MyProject.HO.eigenstrat.chr10.snp.txt
    MyProject.HO.eigenstrat.chr11.geno.txt
    MyProject.HO.eigenstrat.chr11.ind.txt
    MyProject.HO.eigenstrat.chr11.snp.txt
    ...

A naive way to merge across chromosomes might then simply be:

.. code-block:: bash

    cat MyProject.HO.eigenstrat.chr*.geno.txt > MyProject.HO.eigenstrat.allChr.geno.txt
    cat MyProject.HO.eigenstrat.chr*.snp.txt > MyProject.HO.eigenstrat.allChr.snp.txt

(Note that the ``*.ind.txt`` file will be treated separately below). However, these ``cat`` command lines won't do the job correctly, because they won't merge the chromosomes in the right order. To ensure the correct order, I recommend printing all files in a loop in a sub-shell like this:

.. code-block:: bash

    (for CHR in {1..22}; do cat MyProject.HO.eigenstrat.chr$CHR.geno.txt; done) > MyProject.HO.eigenstrat.allChr.geno.txt
    (for CHR in {1..22}; do cat MyProject.HO.eigenstrat.chr$CHR.snp.txt; done) > MyProject.HO.eigenstrat.allChr.snp.txt

Here, each ``cat`` command only outputs one file at a time, but the entire loop runs in a sub-shell denoted by brackets, whose output will be piped into a file.

Now, let's deal with the ``*.ind.txt`` file. As you can see, ``simpleBamCaller`` created one ``*.ind.txt`` per chromosome, but we only need one file in the end, so I suggest you simply copy the one from chromosome 1. But at the same time, we want to fix the third column of the ``*.ind.txt`` file to something more tellinf than "Unknown". So copy the file from chromosome 1, and open it in an editor, and replace all "Unknown" to the population name of that sample. The sex (2nd column) is not necessary.

.. note::

  You should now have three final eigenstrat files for your test samples: A ``*.geno.txt`` file, a ``*.snp.txt`` file and a ``*.ind.txt`` file.

Merging the test genotypes with the Human Origins data set
----------------------------------------------------------

As last step in this session, we need to merge the data set containing your test samples with the HO reference panel. To do this, we will use the ``mergeit``-program from the `Eigensoft package <https://data.broadinstitute.org/alkesgroup/EIGENSOFT/>`_, which is already installed on the cluster.

This program needs a parameter file that - in my case - looks like this::

    geno1:	/projects1/users/schiffels/PublicData/HumanOriginsData.backup/EuropeData.eigenstratgeno.txt
    snp1:	/projects1/users/schiffels/PublicData/HumanOriginsData.backup/EuropeData.simple.snp.txt
    ind1:	/projects1/users/schiffels/PublicData/HumanOriginsData.backup/EuropeData.ind.txt
    geno2:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.allChr.geno.txt
    snp2:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.allChr.snp.txt
    ind2:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.ind.txt
    genooutfilename:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.merged.geno.txt
    snpoutfilename:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.merged.snp.txt
    indoutfilename:	/data/schiffels/GAworkshop/genotyping/MyProject.HO.eigenstrat.merged.ind.txt
    outputformat: EIGENSTRAT

If you have such a parameter file, you can run ``mergeit`` simply like this::

    mergeit -p $PARAM_FILE

and to submit to SLURM::

    sbatch -o $LOG --mem=2000 --wrap="mergeit -p $PARAM_FILE"

where ``$PARAM_FILE`` should be replaced by your parameter file, of course.

To test whether it worked correctly, you should check the resulting "indoutfilename" as specified in the parameter file, to see whether it contains both the individuals of the reference panel and the those of your test data set.

Note that the output of the ``mergeit`` program is by default a binary format called "PACKEDANCESTRYMAP", which is fine for smartpca but not for other analyses we'll be doing later, so I explicitly put the outputformat in the parameter file to force the output to be eigenstrat.
