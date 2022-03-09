# Run each CellProfiler step

The steps below are outlined specifically for running Distributed
CellProfiler on AWS. If you are not doing so, the steps will still be more-or-less the same

- Make sure CellProfiler pipelines are accessible
- Make sure CellProfiler knows where the input images are, either via a CSV or a batch file
- Run each CellProfiler pipeline, in sequence, with appropriate input folder, output folder, and grouping

## Upload your pipelines to S3

In your project's workspace directory, create a batch specific folder and upload your pipelines there. If there are previous batches from the same project or a similar one, you may find it easiest to copy the files directly. Once uploaded and/or copied, the file structure should look like the below.

```
└── pipelines
    └── 2016_04_01_a549_48hr_batch1
        ├── illum_without_batchfile.cppipe
        └── analysis_without_batchfile.cppipe
```

## Configure Distributed-CellProfiler's `run_batch_general` script

```{note}
Note that [`run_batch_general`](https://github.com/CellProfiler/Distributed-CellProfiler/blob/master/run_batch_general.py) is not required;  the Distributed CellProfiler [handbook](https://github.com/CellProfiler/Distributed-CellProfiler/wiki/Step-2%3A-Submit-jobs) lays out a number of different ways of creating jobs.   However, we find it the most efficient way to run numerous pipelines on the same data.  If you do not wish to use it, you can adjust steps 3 and 4 in the "Run each CellProfiler step" to "Create a job file" and "Execute `python3 run.py submitJob jobFileName.json`"
```

`run_batch_general.py` can be configured once at the beginning of the run of a batch of data, and then can be run for each step simply by uncommenting the name of the step to run. The following variables in the `project specific stuff` section of the script should be configured:

- `topdirname` and `batchsuffix` should match your `PROJECT_NAME` and `BATCH_ID`, respectively
- `appname` is typically the same as `topdirname`, but if that name is long and cumbersome you can create an abbreviated version here (ie `2015_10_05_DrugRepurposing` rather than `2015_10_05_DrugRepurposing_AravindSubramanian_GolubLab_Broad`). This will be used in your `config.py` file
- `rows`, `columns`, and `sites` should reflect the imaging conditions used
- `platelist` should contain a list of plates, comma separated, ie `['SQ00015167','SQ00015168']`
- If you are using pipeline files with the LoadData module and CSVs, you should make sure that the pipeline names reflect your pipeline names (or adjust if not). Otherwise, you should make sure that the batch file names reflect your batch file names.
- If following the recommended structures and procedures, none of the `not project specific` section of the script should need to be adapted, but if you are making changes you may need to.

## Configure Distributed-CellProfiler's fleet file

If running in a fresh clone of Distributed-CellProfiler, you will need to configure a single fleet file, which will be used in all subsequent steps. Refer to the [manual](https://github.com/CellProfiler/Distributed-CellProfiler/wiki/Step-3%3A-Start-your-cluster) for instructions.

## Change required parameters in Distributed-CellProfiler's config file

If running in a fresh clone of Distributed-CellProfiler, you will need to set the `AWS_REGION`, `SSH_KEY_NAME`, `AWS_BUCKET`, and `SQS_DEAD_LETTER_QUEUE` settings to appropriate settings for your account. Refer to the [manual](https://github.com/CellProfiler/Distributed-CellProfiler/wiki/Step-1%3A--Configuration) for instructions.

## Run each CellProfiler step

You may have as many as 5 or as few as 2 CellProfiler steps

- (optional) Z projection
- (optional) QC - see also section 4.6
- illumination correction
- (optional) assay development - see also section 4.6
- analysis

For each step, the steps you will run will be identical:

1. Configure the `config.py` file
2. Execute `python3 run.py setup`
3. Uncomment the correct step name in your `run_batch_general.py` file (and ensure all other steps are commented out)
4. Execute `python3 run_batch_general.py`
5. Execute `python3 run.py startCluster files/yourFleetFileName.json`, where you have set the name of the fleet file previously created or located
6. Execute `python3 run.py monitor files/APP_NAMESpotFleetRequestId.json`, where APP_NAME matches the APP_NAME variable set in step 1.

Information on all of these steps is available in the [Distributed-CellProfiler wiki](https://github.com/CellProfiler/Distributed-CellProfiler/wiki). 

You need only absolutely change the variables stated above and below for Distributed-CellProfiler to function, but other variables may be useful, such as using a non-default profile, adjusting whether or not you would like to pre-download files and/or use plugins and/or restart only parts of a batch of data.

In general, as long as you are running inside a tmux session and it isn't killed, the monitor should destroy any and all infrastructure created on AWS as part of the running Distributed-CellProfiler, but it is the user's responsibility to check that this has completed appropriately; failure to do so may lead to spot fleets generating charges after all useful work has completed.

### (Optional) Z projection

- Your `APP_NAME` variable should be set to the `appname` set in `run_batch_general.py` plus `_Zproj`, ie `2015_10_05_DrugRepurposing_Zproj`
- Your number of `CLUSTER_MACHINES` should be medium-large, ie a hundred or few hundred.
- Your `SQS_MESSAGE_VISIBILITY` should be short, such as `5*60` (5 minutes)

### (Optional) QC

- Your `APP_NAME` variable should be set to the `appname` set in `run_batch_general.py` plus `_QC`, ie `2015_10_05_DrugRepurposing_QC`
- Your number of `CLUSTER_MACHINES` should be medium-large, ie a hundred or few hundred.
- Your `SQS_MESSAGE_VISIBILITY` should be short, such as `5*60` (5 minutes)

### Illumination Correction

- Your `APP_NAME` variable should be set to the `appname` set in `run_batch_general.py` plus `_Illum`, ie `2015_10_05_DrugRepurposing_Illum`
- Your number of `CLUSTER_MACHINES` should be set to the number of plates you have divided by 4 then rounded up, ie 6 for 22 plates
- Your `SQS_MESSAGE_VISIBILITY` should be 12 hours `720*60`

### (Optional) Assay Dev

- Your `APP_NAME` variable should be set to the `appname` set in `run_batch_general.py` plus `_AssayDev`, ie `2015_10_05_DrugRepurposing_AssayDev`
- Your number of `CLUSTER_MACHINES` should be medium-large, ie a hundred or few hundred.
- Your `SQS_MESSAGE_VISIBILITY` should be short, such as `5*60` (5 minutes)

### Analysis

- Your `APP_NAME` variable should be set to the `appname` set in `run_batch_general.py` plus `_Analysis`, ie `2015_10_05_DrugRepurposing_Analysis`
- Your number of `CLUSTER_MACHINES` should be as many as possible per your account limits, ideally at least a few hundred.
- Your `SQS_MESSAGE_VISIBILITY` should be 10-20 minutes for images with a binning of 2, longer (30-120 minutes) for unbinned images and/or CellProfiler 2 or 3 runs. This value is the most potentially variable -once you've run a single analysis workflow, you can adjust this value accordingly based on your log files.

## (Optional) Do any post-CellProfiler steps

The optional QC and assay development steps have post-CellProfiler components. These pipelines need only be run if the user plans to do these post-CellProfiler steps.

- QC may require visual inspection of images, [creation of a machine learning classifier to detect poor quality images](https://github.com/CellProfiler/tutorials/tree/master/QualityControl), and/or running scripts to evaluate CV. How best to evaluate quality is left to the user.
- The assay development step creates images to be visually evaluated, either individually or [after stitching for easier evaluation](https://currentprotocols.onlinelibrary.wiley.com/doi/full/10.1002/cpz1.89). If your segmentation is not ideal, you may need to update your assay dev and analysis pipelines by manually tuning the segmentation steps on a local set of representative data until they perform better on your images, then making sure to update the segmentation in the assay dev and analysis pipelines on your cluster accordingly.
- After running the final analysis pipelines, proceed to the next step of this guide.
