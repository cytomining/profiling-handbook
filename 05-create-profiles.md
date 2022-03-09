# Create Profiles

## Confirm Environment Configuration

You CAN, if you choose, make your backends with the same machine that you have used to make CSVs and run DCP for CellProfiler.  However, we typically do not, since backend creation can take a long time so it is desirable to use 2 machines - a small inexpensive machine with only a few CPUs for all the steps before backends and a larger machine with at least as many CPUs as (the number of plates in your batch +1) for backend creation. Both machines can be turned off when not in use and when off will generate only minimal charges.

To make a new backend machine, follow the identical instructions as in the section below, only with a larger Instance Type (such as an m4.10xlarge).

- [Launch an AWS Virtual Machine for making CSVs and running Distributed-CellProfiler](02-config:aws)

Since backend creation also typically requires a large amount of hard disk space, which you pay for whether or not the machine is on, we recommend attaching a large EBS volume to your backend creation machine only when needed, then detaching and terminating it when not in use.

## Add a large EBS volume to your machine

Follow the AWS instructions for [creating](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-volume.html) and [attaching](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-attaching-volume.html) the EBS volume. Two critical factors to note- the volume must be created in the same subnet as your backend creation machine, and should be approximately 2X the size as the analysis files in your batch - this is most easily figured by navigating to that location in the S3 web console and selecting "Actions -\> Calculate total size".

Once the volume is created and attached, ensure the machine is started and SSH connect to it.

Create a temp directory which is required when creating the database backed using `cytominer-database` (discussed later).

```sh
mkdir ~/ebs_tmp
```

Get the name of the disk and attach it.

```sh
# check the name of the disk
lsblk

#> NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
#> xvda    202:0    0     8G  0 disk
#> └─xvda1 202:1    0     8G  0 part /
#> xvdba    202:80   0   100G  0 disk

# check if it has a file system
sudo file -s /dev/xvdba
# ...likely not, in which case you get:
#> /dev/xvdf: data

# if no file system, then create it
sudo mkfs -t ext4 /dev/xvdba

# mount it
sudo mount /dev/xvdba /home/ubuntu/ebs_tmp/

# change perm
sudo chmod 777 ~/ebs_tmp/
```

If you are starting from here, make sure the following steps have been completed on your ec2 instance and/or session before proceeding

- [Configure Environment for Full Profiling Pipeline](02-config)
- [Create list of plates](03-setup-images:create-plates)

## Create Database Backend

Run creation of sqlite backend as well as aggregation of measurements into per-well profiles. This process can be very slow since the files are read from s3fs/EFS. We recommend first downloading the CSVs files locally on an EBS volume attached to the ec2 instance you are running on, and then ingesting.

To do so, first recreate the analysis output folder structure on the EBS volume:

```sh
mkdir -p ~/ebs_tmp/${PROJECT_NAME}/workspace/software

cd ~/ebs_tmp/${PROJECT_NAME}/workspace/software

if [ -d pycytominer ]; then rm -rf pycytominer; fi

git clone https://github.com/cytomining/pycytominer.git

cd pycytominer

git checkout jump

python3 -m pip install -e .
```

The command below first calls `cytominer-database ingest` to create the SQLite backend, and then pycytominer's `aggregate_profiles` to create per-well profiles. Once complete, all files are uploaded to S3 and the local cache are deleted. This step takes several hours, but metadata creation and GitHub setup can be done in this time.

[collate.py](https://github.com/cytomining/pycytominer/blob/jump/pycytominer/cyto_utils/collate.py) ingests and indexes the database.

```sh
pyenv shell 3.8.10

mkdir -p  ../../log/${BATCH_ID}/
parallel \
--max-procs ${MAXPROCS} \
--ungroup \
--eta \
--joblog ../../log/${BATCH_ID}/collate.log \
--results ../../log/${BATCH_ID}/collate \
--files \
--keep-order \
python3 pycytominer/cyto_utils/collate.py ${BATCH_ID}  pycytominer/cyto_utils/ingest_config.ini {1} \
--temp ~/ebs_tmp \
--remote=s3://${BUCKET}/projects/${PROJECT_NAME}/workspace :::: ${PLATES}
```

```{note}
`collate.py` does not recreate the SQLite backend if it already exists in the local cache. Add `--overwrite` flag to recreate.
```

```{note}
or pipelines that use FlagImage to skip the measurements modules if the image failed QC, the failed images will have Image.csv files with fewer columns that the rest (because columns corresponding to aggregated measurements will be absent). The ingest command will show a warning related to sqlite: `expected X columns but found Y - filling the rest with NULL`. This is expected behavior.
```

```{note}
There is a known [issue](https://github.com/cytomining/cytominer-database/issues/100) where if the alphabetically-first CSV failed QC in a pipeline where "Skip image if flagged" is turned on, the databases will not be created. We are working to fix this, but in the meantime we recommend either not skipping processing of your flagged images (and removing them from your data downstream) or deleting the alphabetically-first CSVs until you come to one where the pipeline ran completely.
```

This is the resulting structure of `backend` on S3 (one level below `workspace`) for `SQ00015167`:

```
└── backend
    └── 2016_04_01_a549_48hr_batch1
        └── SQ00015167
            ├── SQ00015167.csv
            └── SQ00015167.sqlite
```


At this point, the user needs to use the [profiling template](https://github.com/cytomining/profiling-template) to use [pycytominer](https://github.com/cytomining/pycytominer/) to annotate the profiles with metadata, normalize them, and feature select them.

## Create Metadata Files

First, get metadata for the plates. This should be created beforehand and uploaded into S3.

This is the structure of the metadata folder (one level below `workspace`):

```
└── metadata
    └── platemaps
        └── 2016_04_01_a549_48hr_batch1
            ├── barcode_platemap.csv
            └── platemap
                └── C-7161-01-LM6-006.txt
```

`2016_04_01_a549_48hr_batch1` is the batch name -- the plates (and all related data) are arranged under batches, as seen below.

`barcode_platemap.csv` is structured as shown below. `Assay_Plate_Barcode` and `Plate_Map_Name` are currently the only mandatory columns (they are used to join the metadata of the plate map with each assay plate). Each unique entry in the `Plate_Map_Name` should have a corresponding tab-separated file `.txt` file under `platemap` (e.g. `C-7161-01-LM6-006.txt`).

```
Assay_Plate_Barcode,Plate_Map_Name
SQ00015167,C-7161-01-LM6-006
```

The tab-separated files are plate maps and are structured like this:
(This is the typical format followed by Broad Chemical Biology Platform)

```
plate_map_name  well_position broad_sample  mg_per_ml mmoles_per_liter  solvent
C-7161-01-LM6-006 A07 BRD-K18895904-001-16-1  3.12432000000000016 9.99999999999999999 DMSO
C-7161-01-LM6-006 A08 BRD-K18895904-001-16-1  1.04143999999919895 3.33333333333076923 DMSO
C-7161-01-LM6-006 A09 BRD-K18895904-001-16-1  0.347146666668001866  1.11111111111538462 DMSO
```

```{note}
- `plate_map_name` should be identical to the name of the file (without extension). 
- `plate_map_name` and `well_position` are currently the only mandatory columns.
```

The external metadata is an optional file tab separated `.tsv` file that contains the mapping between a perturbation identifier to other metadata. The name of the perturbation identifier column (e.g. `broad_sample`) should be the same as the column in platemap.txt.

The external metadata file should be placed in a folder named `external_metadata` within the `metadata` folder. If this file is provided, then the following should be the folder structure

```
└── metadata
    ├── external_metadata
    │   └── external_metadata.tsv
    └── platemaps
        └── 2016_04_01_a549_48hr_batch1
            ├── barcode_platemap.csv
            └── platemap
                └── C-7161-01-LM6-006.txt
```

## Set up GitHub

Once and only once - fork the [profiling recipe](https://github.com/cytomining/profiling-recipe) to your own user name (Each time you make a new project - you may want to [keep your fork up to date](https://docs.github.com/en/github/collaborating-with-pull-requests/working-with-forks/syncing-a-fork)

Once per new PROJECT, not new batch - make a copy of the [template repository](https://github.com/cytomining/profiling-template) into your preferred organization with a project name that is similar OR identical to its project tag on S3 and elsewhere.

## Make Profiles

### Optional - set up compute environment

These final steps are small and can be done either in your local environment or on your node used to build the backends. Conda is currently required and not currently on the backend creation VMs. Use these commands to install Miniconda for a Linux ecosystem (otherwise, search for your own OS).

```sh
# Miniconda installation
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
bash ~/miniconda.sh -b -p $HOME/miniconda
export PATH="/home/ubuntu/miniconda/bin:$PATH"
source .bashrc
conda init bash
source .bashrc
```

We recommend using DVC for large file management and DVC requires the use of AWS CLI. AWS CLI is already on the backend creation VMs. If you are not using the backend creation VM, use these commands to install AWS CLI for a Linux ecosystem (otherwise, search for your own OS).

```sh
# AWS CLI installation - if not on your machine already
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Configure your AWS CLI installation with
```
aws configure
```

It will prompt you for: 
- `AWS Access Key ID:` YOUR-KEY
- `AWS Secret Access Key:` YOUR-SECRET-KEY 
- `Default region name:` e.g. us-east-1 
- `Default output format:` json

If not using the same machine + tmux as for making backends, where your environment variables are already set, set them up

- [Configure Environment for Full Profiling Pipeline](02-config)

### Set new environment variables

Specifically, `ORG` and `DATA` should be the GitHub organization and repository name used when creating the data repository from the template. `USER` should be your GitHub username. CONFIG_FILE will be the name of the config file used for this run, so something that makes it distinguishable (ie, batch numbers being run at this time) is helpful.

```
ORG=broadinstitute
DATA=2015_10_05_DrugRepurposing_AravindSubramanian_GolubLab_Broad
USER=gh_username
CONFIG_FILE=config_batch1
```

### If a first batch in this compute environment, make some directories

```sh
mkdir -p ~/work/projects/${PROJECT_NAME}/workspace/{backend,software}
```

### Add your backend files

```
aws s3 sync s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/backend/${BATCH_ID} ~/work/projects/${PROJECT_NAME}/workspace/backend/${BATCH_ID} --exclude="*" --include="*.csv"
```

### If a first batch in this compute environment, clone your repository

```sh
cd ~/work/projects/${PROJECT_NAME}/workspace/software
git clone git@github.com:${ORG}/${DATA}.git
# depending on your repo/machine set up you may need to provide credentials here
cd ${DATA}
```

### If a first batch in this project, weld the recipe into the repository

```sh
git submodule add https://github.com/${USER}/profiling-recipe.git profiling-recipe
git add profiling-recipe
git add .gitmodules
git commit -m 'finalizing the recipe weld'
git push
# depending on your repo/machine set up you may need to provide credentials here
git submodule update --init --recursive
```

### If a first batch in this compute environment, set up the environment

```sh
cp profiling-recipe/environment.yml .
conda env create --force --file environment.yml
```

### Activate the environment

```sh
conda activate profiling
```

### Set up DVC

Initialize DVC for this project and set it to store large files in S3. This needs to happen once per project, not per batch. Skip this step if not using DVC.

```sh
# Navigate
cd ~/work/projects/${PROJECT_NAME}/workspace/software/${DATA}/profiling-recipe
# Initialize DVC
dvc init
# Set up remote storage
dvc remote add -d S3storage s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/software/${DATA}_DVC
# Commit new files to git
git add .dvc/.gitignore .dvc/config
git commit -m "Setup DVC"
```

### If a first batch in this project, create the necessary directories

```sh
profiling-recipe/scripts/create_dirs.sh
```

### Download the load_data_CSVs

```sh
aws s3 sync s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID} load_data_csv/${BATCH_ID}
gzip -r  load_data_csv/${BATCH_ID}
```

### Download the metadata files

```sh
aws s3 sync s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/metadata/${BATCH_ID} metadata/platemaps/${BATCH_ID}
```

### Make the config file

```sh
cp profiling-recipe/config_template.yml config_files/${CONFIG_FILE}.yml
nano config_files/${CONFIG_FILE}.yml
```

```{note}
The changes you will likely need to make for most small use cases following this handbook. For `aggregate` set `perform` to `false`, for `annotate` sub-setting `external` set `perform` to `false`, in `feature_select` set `gct` to `true`,
and finally at the bottom set the batch(es) and plates names

For large batches with many DMSO wells and external metadata ala the JUMP project - set `perform` under `external` to `true`, set `file` to the name of the external metadata file and set `merge_column` to the name of the compound identifier column in platemap.txt and external_metadata.tsv.
```


### Set up the profiles
Note that the “find” step can take a few seconds/minutes

```
mkdir -p profiles/${BATCH_ID}
find ../../backend/${BATCH_ID}/ -type f -name "*.csv" -exec profiling-recipe/scripts/csv2gz.py {} \;
rsync -arzv --include="*/" --include="*.gz" --exclude "*" ../../backend/${BATCH_ID}/ profiles/${BATCH_ID}/
```


### Run the profiling workflow
Especially for large number of plates, this will take some time.  Output will be logged to the console as different steps proceed.

```
python profiling-recipe/profiles/profiling_pipeline.py --config config_files/{$CONFIG_FILE}.yml
```

### Push resulting files back up to GitHub
If using a data repository, push the newly created profiles to DVC and the .dvc files and other files to GitHub as follows

```sh
dvc add profiles/${BATCH} --recursive
dvc push
git add profiles/${BATCH}/*.dvc profiles/*.gitignore
git commit -m 'add profiles'
git add *
git commit -m 'add files made in profiling'
git push
```

If not using DVC but using a data repository, push all new files to GitHub as follows

```sh
git add *
git commit -m 'add profiles for batch _'
git push
```



### Push resulting files up to S3

```
parallel aws s3 sync {1} s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/{1} ::: config_files gct profiles quality_control
```
