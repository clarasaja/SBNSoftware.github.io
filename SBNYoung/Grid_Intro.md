---
layout: page
title: The Grid
---

## Overview

This page collects the basic steps for running ICARUS and SBND jobs on the grid from a GPVM or build node.

The workflow below uses `project.py` from LArBatch together with the experiment products environment.

## Certificates

### FNAL - Update March 27, 2025
New way of getting bearer token after CILogon CA shutoff:

```bash
htgettoken -a htvaultprod.fnal.gov -i <experiment>
```
Please setup the ICARUS or SBND products and the right experiment tag.

It might be needed to get a new version of LArBatch to use the updated `project.py` version that can handle the token:

```bash
unsetup larbatch
setup larbatch v01_61_00
```

### CNAF

```bash
source /cvmfs/icarus.opensciencegrid.org/products/icarus/setup_icarus.sh
setup cigetcert
cigetcert -i 'Fermi National Accelerator Laboratory' --myproxyserver=myproxy.cnaf.infn.it --myproxyretrievers='/DC=org/DC=incommon/C=US/ST=Illinois/O=Fermi Research Alliance/CN=(fifebatch|(hepcjobsub0(1|2))|jobsubdevgpvm01).fnal.gov' --myproxyhours=24 --hours=672 -u <fnal_user>
voms-proxy-init -noregen -rfc -voms 'fermilab:/fermilab/<experiment>/Role=Analysis'
```
Remember to setup FIFE stuff (and `samweb`, e.g. needed to prestage):

```bash
setup fife_utils
```

## Prestaging
Create a `samweb` definition and prestage the related files:

```bash
# list files
samweb list-files "run_number 12174 and data_tier raw"

# check files in the definition
samweb list-definition-files triozzi_raw_run12174_offbeambnbminbiascalib

# locate a file
# XRootD: --schema=root
samweb locate-file <filename>.root

# check the metadata of a file, if you care
samweb get-metadata <filename>.root

# create a samweb defintion
samweb create-definition triozzi_raw_run12174_offbeambnbminbiascalib "run_number 12174 AND data_tier raw"

# prestage
# ...possibly on tmux! This WILL hang for a while and
# all background processes launched from a SL7 container
# will die when closing it.
nohup sam_prestage_dataset --defname=triozzi_raw_run12174_offbeambnbminbiascalib &
```
The prestaging process will appear to hang and only print file names as they are successfully staged.
You can check the status of a prestaging process using:
[https://samicarus.fnal.gov:8483/station_monitor/icarus/stations/icarus/projects/](https://samicarus.fnal.gov:8483/station_monitor/icarus/stations/icarus/projects/)

## Submitting jobs

Submit jobs with LArBatch/`project.py`, with specifications (release, code, paths) from a custom XML file:

```bash
project.py --xml /path/to/<xml-file> --stage <stage> --submit
```
The wrapper can be used also for fetching job logs (`--check`) and trying to re-run failed jobs (`--makeup`), with the same sintax.

You can do this on a GPVM or from a build node.

You need to modify the XML, and if you prepare it I can take a look at it.

## XML Example

To be written.

## Checking the job status

Using the `project.py` wrapper:

```bash
# get the job status
jobsub_q --user $USER -G <experiment>

# REMOVE job by process id
jobsub_rm --jobid <PID>
```
Check the status here: [https://fifemon.fnal.gov/monitor/d/000000116/user-batch-details?orgId=1&var-cluster=fifebatch&var-user=$USER](https://fifemon.fnal.gov/monitor/d/000000116/user-batch-details?orgId=1&var-cluster=fifebatch&var-user=$USER)
For "held" jobs, the reasons can be checked here ([https://fifemon.fnal.gov/monitor/d/000000146/why-are-my-jobs-held?orgId=1&var-user=$USER](https://fifemon.fnal.gov/monitor/d/000000146/why-are-my-jobs-held?orgId=1&var-user=$USER)). Useful stuff:

```bash
# get held-code of held jobs
jobsub_q --hold --user $USER -G <experiment>

# RELEASE held job
jobsub_release --constraint '(JobStatus=?=5) && (Owner=?="$USER")'

# REMOVE held jobs
jobsub_rm --constraint '(JobStatus=?=5) && (Owner=?="$USER")'

# REMOVE ANY job from you:BE CAREFUL!
jobsub_rm --constraint '(Owner=?="$USER")'
```
Jobs are usually held when resources needed to process the job exceed what has ben asked to the grid via XML. Suggested to just get rid of the jobs, and a run a `--check` and `--makeup` with updated resources.
