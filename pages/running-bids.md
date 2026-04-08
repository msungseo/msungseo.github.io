---
title: How to run mriqc and fmriprep on the cluster
date: 2018-02-05
author: Maureen Ritchey    # optional
aliases: 
  - /resources/2018/02/05/running-bidsapps-on-cluster/
---

We have recently made the leap to using the BIDS apps [mriqc](https://mriqc.readthedocs.io/en/latest/index.html) and [fmriprep](https://fmriprep.readthedocs.io/en/latest/index.html) for fMRI quality control and pre-processing, respectively.

The instructions for installation and usage can be found in the documentation, but in some cases, I had to cobble together information from different sources. So here's a run-down of how to run fmriprep on the Boston College Linux cluster (Sirius). If you're not at BC, you might still find the instructions useful for setting everything up on your own cluster. The instructions are nearly identical for mriqc-- I'll note some differences at the end.

### How to run FMRIPREP

-   Install [Docker](https://www.docker.com/) on your local computer. If you want to test out the apps on your local machine before using them on the cluster, increase the default memory settings (*Preferences \> Advanced*).

-   Use Docker to pull the most recent version of fmriprep: `docker pull poldracklab/fmriprep:latest`

-   Create a singularity image of the Docker container. This is assuming that you cannot run Docker directly on your cluster, which is the case for us. Instructions [here](http://fmriprep.readthedocs.io/en/latest/installation.html#singularity-container). This will take some time. It's a big file.

-   Copy the singularity image to a folder on the cluster. Make sure singularity is installed on your cluster. Research Services already installed it on the BC cluster (Sirius).

-   Get a FreeSurfer [license file](https://surfer.nmr.mgh.harvard.edu/fswiki/License) and save it somewhere in your directory. If you want to include FreeSurfer processing, you'll need to point fmriprep to this file.

-   Make sure your dataset follows BIDS format using the helpful [BIDS validator tool](http://incf.github.io/bids-validator/).

-   You're ready to run! Let's start by running it on an interactive node. The following command will run fmriprep for a single participant (`sub-s001`) on 8 processors, which speeds it up a lot (`--nthreads 8`). It skips the FreeSurfer recon-all step because it's rather time-consuming (`--fs-no-reconall`), although I've left in the license file path (`--fs-license-file`). It will save out the functional data in T1w (native) space and template (MNI) space (`--output-space`). In the Terminal, enter the following:

    ```         
    qsub -I -X -l walltime=24:00:00,nodes=1:ppn=8,mem=30gb
    ```

    Wait until the compute node opens, then:

    ```         
    module load singularity
    singularity run /data/ritcheym/singularity/poldracklab_fmriprep_latest-2018-01-04-866557a8f305.img /data/ritcheym/data/fmri/orbit/data/sourcedata/ /data/ritcheym/data/fmri/orbit/data/derivs/ participant --participant-label s001 -w /data/ritcheym/data/fmri/orbit/data/work/ --nthreads 8 --fs-license-file ~/freesurfer/license.txt --fs-no-reconall --output-space {T1w,template}
    ```

-   Check out the output, which should be saved in `derivs/fmriprep` (or whatever path you specified above).

    -   Isn't it pretty? Make sure it did everything you expected it to do before you move forward with running the entire batch of participants. For instance, it will skip slice time correction if you don't have `SliceTiming` in the json file describing your functional data.
    -   It will also skip field map-based correction if you don't have the appropriate info in your fmap json files. One note about that: you must include the `IntendedFor` parameter to tell it which functional runs correspond to the field map.

-   Now you're ready to run it as a job array, which will let you simultaneously process a batch of participants. Create a pbs file so that it looks something like this:

    ```         
    #!/bin/tcsh
    #PBS -l mem=30gb,nodes=1:ppn=8,walltime=24:00:00
    #PBS -m abe -M ritcheym@bc.edu
    #PBS -N orbit_fmriprep_1-6
    #PBS -t 1-6

    module load singularity
    singularity run /data/ritcheym/singularity/poldracklab_fmriprep_latest-2018-01-04-866557a8f305.img /data/ritcheym/data/fmri/orbit/data/sourcedata/ /data/ritcheym/data/fmri/orbit/data/derivs/ participant --participant-label s00${PBS_ARRAYID} -w /data/ritcheym/data/fmri/orbit/data/work/ --nthreads 8 --fs-license-file ~/freesurfer/license.txt --fs-no-reconall --output-space {T1w,template}
    ```

-   In the Terminal, navigate to the directory where you've saved your pbs file and call: `qsub fmriprep_job.pbs` (or whatever you've named it).

-   You're done! Now check your data.

### Adjustments for MRIQC

Running mriqc is very similar. Follow all of the steps ahead, with 2 exceptions: 1) You do not need the FreeSurfer license file, and 2) The command should look something like this:

```         
singularity run /data/ritcheym/singularity/poldracklab_mriqc_latest-2018-01-18-9c5425cc1abb.img /data/ritcheym/data/fmri/orbit/data/sourcedata/ /data/ritcheym/data/fmri/orbit/data/derivs/mriqc/ participant --participant-label s001 -w /data/ritcheym/data/fmri/orbit/data/work/mriqc/ --n_procs 8
```

By the way, I'm not sure why fmriprep seems to want the `--nthreads` flag and mriqc wants to the `--n_procs` flag. The online documentation gives you multiple options for both, but I got a bug when using the other options in my job array.

After you've run mriqc for multiple subjects, you can generate a group report by running:

```         
singularity run /data/ritcheym/singularity/poldracklab_mriqc_latest-2018-01-18-9c5425cc1abb.img /data/ritcheym/data/fmri/orbit/data/sourcedata/ /data/ritcheym/data/fmri/orbit/data/derivs/mriqc/ group -w /data/ritcheym/data/fmri/orbit/data/work/mriqc/
```

This runs super quickly (like, 2 seconds).

A final note: We've been running into permissions issues with running singularity on files owned by another user (even when we have read/write access). We don't have a solution other than making sure the owner is the one who actually sets up the job arrays. If anyone knows the answer, please share.
