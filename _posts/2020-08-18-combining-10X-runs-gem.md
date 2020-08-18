---
layout: posts
title: "I ran a one 10X single cell gene expression library on two or more flow cells. What now?"
date: 2020-08-18
---

I recently submitted a fairly large number of 10X single cell gene expression libraries for sequencing at the [Harvard Bauer Core](https://bauercore.fas.harvard.edu/). 
Due to the large estimated number of cells, I requested two NovaSeq flow cells to ensure the reads/cell I wanted. I initially requested that the core run half
of the samples on one flow cell and the other half on the second flow cell. However, the folks at the core recommended splitting all the samples between the two
flow cells, to allow them to optimize conditions between the two runs. None of our samples had index conflicts, so that seemed like a great idea.

The issue is that the Bauer Core automates running of the [Cell Ranger](https://support.10xgenomics.com/single-cell-gene-expression/software/overview/welcome) 
pipeline as soon as each flow cell has finished. Given the obvious manpower savings inherent to this approach, I suspect virtually all sequencing cores use a 
similar system. The upshot is that the core delivers you *two* 10X outputs for each sample, one from each flow cell, and each of which has 1/2 the sequencing
depth you were anticipating. In principle this is an easy problem to fix - you just combine the two datasets, and viola, you have one dataset with the desired
sequencing depth. Pratically, however, one needs to figure out how to do that.

The other issue was that half of our samples were from mouse tissue, and half were from human cells. Unfortunately, the initial pipeline was run using a mouse
reference genome for all of the samples. So, I also needed to rerun the pipeline for human cells with a human reference genome.

The [10X website](https://www.10xgenomics.com/) offers some insight into solving this problem [here](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/count) and [here](https://kb.10xgenomics.com/hc/en-us/articles/360000930812-Can-I-combine-data-from-multiple-sequencing-runs-for-the-same-10x-library-).
It turns out it is as simple as rerunning 'cellranger count' for each sample with the FASTQ files from both flow cells. (Of course, this means you need to either 
have access to a computing cluster or a very powerful workstation you don't mind tying up for an extended period.) Since the FASTQs are stashed in different 
locations by the original Cell Ranger runs, you can use a comma-separated list of FASTQ directories. For each library, the Cell Ranger command will be:

> cellranger count `--`id=sample_id `\`<br>
> `--`fastqs=/path/to/run1/fastqs,/path/to/run2/fastqs,etc `\`<br>
> `--`transcriptome=/path/to/refernce/genome `\`<br>
> `--`chemistry=auto `\`<br>
> `--`nosecondary
  
I did not want to have to manually write a script and submit it to the cluster for every library, especially if I ever run across this issue again (and given the
amount of single cell gene expression analysis I anticipate doing, that seems likely). So, I wrote a pair of shell scripts to automate this process, and to assign
each library to the appropriate reference genome. You can see and download the code [here](https://github.com/riguyer/shell-functions/tree/master/combine_10X). Of
course, these scripts are specific to the LSF platform, which is what MGB (formerly Partners HealthCare) uses to schedule jobs on their resarch computing cluster.
If your organization uses a different scheduler, the scripts will need to be modified accordingly.

To use these scripts, place both .sh files in a directory containing the automated Cell Ranger output from each flow cell. Modify the 'run_dirs' variable to
contain the path to each run. Then run the following from the command line:

> bash parallel_combine.sh -hs {human samples} -mm {mouse samples}

This will automatically generate .lsf files for each library with the appropriate reference genome, stash them in a subdirectory, and submit the scripts to the
LSF scheduler. I wrote [make_lsf.sh](https://github.com/riguyer/shell-functions/blob/master/combine_10X/make_lsf.sh)
to request 6 cores and 32 GB memory for each submission, which was sufficient to successfully run the pipeline on each sample in a little over 24 hrs. You can
tweak this based on the resource limits imposed by your organization.
