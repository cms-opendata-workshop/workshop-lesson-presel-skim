---
title: "Running your own NanoAOD"
teaching: 10
exercises: 20
questions:
- "How can I run over many AOD files to produce NanoAOD?"
objectives:
- "Experience running an entire sample"
keypoints:
- "The configuration file can be adapted to run many files in sequence"
- "HTCondor and other tools helps when running many files in parallel"
---

By this point you have made many updates to `AOD2NanoAOD.cc`, and for a future analysis it could
look completely unique to your needs. There are several ways to run over large datasets (in contrast
to the test files we have used so far). A cloud-based solution will be covered in detail in a later
lesson, so here we will introduce a basic sequential method.

## Extending the file list

In `configs/simulation_cfg.py`, we are currently instructing CMSSW to open one file:

~~~
process.source = cms.Source("PoolSource",
        fileNames = cms.untracked.vstring('root://eospublic.cern.ch//eos/opendata/cms/MonteCarlo2012/Summer12_DR53X/TTbar_8TeV-Madspin_aMCatNLO-herwig/AODSIM/PU_S10_START53_V19-v2/00000/000A9D3F-CE4C-E311-84F8-001E673969D2.root'))

process.maxEvents = cms.untracked.PSet(input = cms.untracked.int32(200))
~~~
{: .source}

Note that `fileNames` is expecting a `vstring`, which stands for "vector of strings". So these
configuration files can easily support running over multiple input files (with the obvious increase in
run time and output file size).

Multiple files can be specified as a comma-separated list (`cms.vstring('file1.root', 'file2.root')`),
or by reading an entire text file with one ROOT file per line:

~~~
# Define files of dataset
files = FileUtils.loadListFromFile("data/CMS_MonteCarlo2012_Summer12_DR53X_TTbar_8TeV-Madspin_aMCatNLO-herwig_AODSIM_PU_S10_START53_V19-v2_00000_file_index.txt")

files.extend(FileUtils.loadListFromFile("data/CMS_MonteCarlo2012_Summer12_DR53X_TTbar_8TeV-Madspin_aMCatNLO-herwig_AODSIM_PU_S10_START53_V19-v2_20000_file_index.txt"))

process.source = cms.Source(
   "PoolSource", fileNames=cms.untracked.vstring(*files))
~~~
{: .source}

These .txt files live in the `data/` subdirectory and contain "eospublic" links to all the ROOT files
in a sample.

## Parallelization

Parallelization is important for efficiently running over many files. To get a taste for this issue,
set the `maxEvents` parameter in `simulation_cfg.py` to -1 and set the job running. This will give
a time estimate for 1 file.

~~~
// change the max events and output file name
process.maxEvents = cms.untracked.PSet(input=cms.untracked.int32(-1))

process.TFileService = cms.Service(
    "TFileService", fileName=cms.string("output_fullfile.root"))
// save and quit

$ cmsRun configs/simulation_cfg.py
~~~

Given the limits of an Open Data container, a few options for parallelization are available:

 * Execute cmsRun once per sample for jobs that process all the ROOT files in a sample.
 * Create a python or bash script to loop through a list of files, setting input and output file names
 in a configuration file template, and executing cmsRun. This will produce many ROOT files for each
 sample.
 * Use a cloud-based solution such the example coming in a later lesson.

If you use a method that produces more than one output ROOT file per sample (ex: 54 output files for VBF
Higgs production), combining them could simplify future steps of your physics analysis. This can be
done very naturally with ROOT via the `hadd` method:

~~~
$ cmsenv # if this is a new shell
$ hadd mergedfile.root input1.root input2.root input3.root #...and so on...
~~~
{: .source}

Note: the internal content of the ROOT files must be the same (ex: tree names and branch lists) for ROOT to add them intelligently. 

>##Challenge: hadd 2 root files
>
>You will have at least one `output.root` file already. Make another one from a short test (or your
>full file if it has finished!) and give it a different output name. Perform the `hadd` command and
>make sure the resulting ROOT file has the structure and merged content that you expect.
{: .challenge}


{% include links.md %}

