# Appendix

## Project folder structure guidance

The guidances notes provide an overview of the folder structure used in
this handbook

-   All projects live in an S3 bucket
-   The directory structure is always
    `<bucket-name>/projects/<project_name>`
-   In the `<project_name>` folder there are sub directories
    -   The first subdirectory is `workspace`
    -   Other subdirectories are batches of data
-   The batches of data are labeled by date and include `images` and
    `illum` folders
    -   In the `images` folder there exist different plates storing raw
        image data
    -   The `illum` folder is identical to the `images` folder in terms
        of structure
        -   `illum` is an output of the first stage of cell profiler
            pipeline that stores a function to adjust the plates in
            `images`
-   `workspace` also has subdirectories
    -   `analysis` - includes subfolders mirroring the `Batch` nesting
        -   Within each `batch` folder, the CellProfiler results are
            stored in `plate_id`
            -   Within each `plate folder` there is an `analysis` folder
                -   Inside this `analysis` folder, each well has its own
                    folder (e.g. `A01-1`)
                    -   `A` and `01` refer to the row and column of the
                        plate, 1 refer to sites per well
                    -   If the grouping was done by well instead of by
                        site, this would be `A01`, without the suffix of
                        `-1`
                -   Note that this `analysis` folder is customizable
                    -   There are typically 384 (# of wells) x 9 (# of
                        sites per well) subfolders
                        -   384 well plate
                        -   9 different pictures
                -   Within the site folder (e.g. `A01-1`) there are five
                    csv files
                    -   `Cells.csv`
                        -   Each row are measurements of one cell
                    -   `Cytoplasm.csv`
                        -   Another object similar to Cells.csv
                    -   `Nuclei.csv`
                        -   Another object similar to Cells.csv
                        -   These three object files can be concatenated
                            by column `Objects.csv`
                    -   `Experiment.csv`
                        -   Stores metadata for the CellProfiler run,
                            including the CellProfiler pipeline itself
                    -   `Image.csv`
    -   `backend` - also includes `batch` nesting
        -   `batch` nesting
            -   `plate` nesting - stores summaries of each plate (all
                .csv files also have .gct formats (for input into
                [Morpheus](https://software.broadinstitute.org/morpheus/))
                -   `<plate_id>.sqlite` - inner join of all objects in a
                    well, and then stacked (so all data for each well in
                    a single plate)
                -   `<plate_id>.csv` - per well
                    [means](https://github.com/broadinstitute/cytominer_scripts/blob/master/aggregate.R#L44)
                    for each well on the plate
                -   `<plate_id>.augmented.csv` - same as .csv except it
                    includes the metadata
                -   `<plate_id>._normalized.csv` - some z scored version
                    of augmented
                -   `<plate_id>._normalized_variable_selected.csv` -
                    across all the plates in the batch
                    -   Three feature selection steps
                        -   Variance threshold
                        -   Correlation threshold (decorrelate feature
                            set)
                        -   Replicate correlation filter (\>0.6)
    -   `parameters` - same structure as `backend` but with metadata
        results (e.g. the features selected in variable selection)
    -   `software`
        -   This is where the project's github repository lives.
        -   The scripts in the handbook assume that this be named as the
            same name as the Project folder. To rename it, pay careful
            attention to paths when executing the commands in the
            handbook.

## Directory structure

    ├── 2016_04_01_a549_48hr_batch1
    │   ├── illum
    │   │   └── SQ00015167
    │   │       ├── SQ00015167_IllumAGP.mat
    │   │       ├── SQ00015167_IllumDNA.mat
    │   │       ├── SQ00015167_IllumER.mat
    │   │       ├── SQ00015167_IllumMito.mat
    │   │       ├── SQ00015167_IllumRNA.mat
    │   │       └── SQ00015167.stderr
    │   └── images
    │       └── SQ00015167__2016-04-21T03_34_00-Measurement1
    │         ├── Assaylayout
    │         ├── FFC_Profile
    │         └── Images
    │             ├── r01c01f01p01-ch1sk1fk1fl1.tiff
    │             ├── r01c01f01p01-ch2sk1fk1fl1.tiff
    │             ├── r01c01f01p01-ch3sk1fk1fl1.tiff
    │             ├── r01c01f01p01-ch4sk1fk1fl1.tiff
    │             └── r01c01f01p01-ch5sk1fk1fl1.tiff
    └── workspace
        ├── audit
        │    └── 2016_04_01_a549_48hr_batch1
        ├── analysis
        │    └── 2016_04_01_a549_48hr_batch1
        │        └── SQ00015167
        │            └── analysis
        │                └── A01-1
        │                    ├── Cells.csv
        │                    ├── Cytoplasm.csv
        │                    ├── Experiment.csv
        │                    ├── Image.csv
        │                    ├── Nuclei.csv
        │                    └── outlines
        │                        └── SQ00015167
        │                            ├── A01_s1--cell_outlines.png
        │                            └── A01_s1--nuclei_outlines.png
        ├── backend
        │   └── 2016_04_01_a549_48hr_batch1
        │       └── SQ00015167
        │           ├── SQ00015167.csv
        │           └── SQ00015167.sqlite
        ├── images
        │   └── 2016_04_01_a549_48hr_batch1 -> /home/ubuntu/bucket/projects/2015_10_05_DrugRepurposing_AravindSubramanian_GolubLab_Broad/2016_04_01_a549_48hr_batch1/images/
        ├── load_data_csv
        │   └── 2016_04_01_a549_48hr_batch1
        │       └── SQ00015167
        │           ├── load_data.csv
        │           └── load_data_with_illum.csv
        ├── log 
        │   ├── create_csv_from_xml
        │   └── collate       
        ├── metadata
        │   └── 2016_04_01_a549_48hr_batch1
        │       ├── barcode_platemap.csv
        │       └── platemap
        │           └── C-7161-01-LM6-006.txt
        ├── pipelines
        ├── status
        └── software
            ├── Distributed-CellProfiler
            └── pe2loaddata
