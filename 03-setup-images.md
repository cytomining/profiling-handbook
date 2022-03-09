# Setup Images

## Upload Images

Your image files should be uploaded to AWS from your local compute environment via a tool like [Cyberduck](https://cyberduck.io/) or the [AWS CLI](https://aws.amazon.com/cli/) (`aws s3 sync /local/path s3://BUCKET/PROJECT_NAME/BATCH_ID/images`) (see also Appendix A.2 for more information on folder structures). Some important tips BEFORE uploading (these are much more difficult to fix once uploaded):

- Ensure your image sets are complete i.e. all image sets should have the same number of channels and z-planes, and that this is true across the entire batch of plates you are processing.
- Avoid folder names with spaces
- Plate names should not have leading 0's (ie `123` not `000123`)
- **VERY IMPORTANT**: If using `pe2loaddata` (described later) to generate your image CSVs, please ensure the folder name contains the plate name given when imaging on the Phenix microscope (can be checked in the `Index.idx.xml`)

## Prepare Images

(if using `pe2loaddata` to create image sets)

Create soft link to the image folder. Note that the relevant S3 bucket has been mounted at `/home/ubuntu/bucket/`.

```{note}
The folder structure for `images` differs between `S3` and `EFS`. This can be potentially confusing. However note that the step below simply creates a soft link to the images in S3; no files are copied. Further, when `pe2loaddata` is run, the `--sub-string-out` and `--sub-string-in` flags ensure the resulting LoadData CSV files end up having the paths to the images as they exist on S3. Thus the step below (of creating a softlink) only serves the purpose of making the `images` folder have a similar structure as the others (e.g. `load_data_csv`, `metadata`, `analysis`).

If you're Z-projecting images and the unprojected images are in a folder with a different name (such as /unprojected_images/), you should create the soft link to that folder:
`ln -s ~/bucket/projects/${PROJECT_NAME}/${BATCH_ID}/unprojected_images/ ${BATCH_ID}`
```

```sh
cd ~/efs/${PROJECT_NAME}/workspace/
mkdir images #Run this only if this is the first batch for this project
cd images
ln -s ~/bucket/projects/${PROJECT_NAME}/${BATCH_ID}/images/ ${BATCH_ID}
cd ..
```

This is the resulting structure of the image folder on EFS (one level below `workspace`):

```
└── images
    └── 2016_04_01_a549_48hr_batch1 -> /home/ubuntu/bucket/projects/2015_10_05_DrugRepurposing_AravindSubramanian_GolubLab_Broad/2016_04_01_a549_48hr_batch1/images/
```

This is the structure of the image folder on S3 (one level above `workspace`, under the folder `2016_04_01_a549_48hr_batch1`.) Here, only one plate (`SQ00015167__2016-04-21T03_34_00-Measurement1`) is show but there are often many more.

```
└── images
    └── 2016_04_01_a549_48hr_batch1
        └── SQ00015167__2016-04-21T03_34_00-Measurement1
            ├── Assaylayout
            ├── FFC_Profile
            └── Images
                ├── r01c01f01p01-ch1sk1fk1fl1.tiff
                ├── r01c01f01p01-ch2sk1fk1fl1.tiff
                ├── r01c01f01p01-ch3sk1fk1fl1.tiff
                ├── r01c01f01p01-ch4sk1fk1fl1.tiff
                └── r01c01f01p01-ch5sk1fk1fl1.tiff
```

`SQ00015167__2016-04-21T03_34_00-Measurement1` is the typical nomenclature followed by Broad Chemical Biology Platform for plate names. `Measurement1` indicates the first attempt to image the plate. `Measurement2` indicates second attempt and so on.

(03-setup-images:create-plates)=
## Create List of Plates

(if using `pe2loaddata` to create image sets)

Create a text file with one plate id per line. The plate IDs, if using `pe2loaddata`, must match the plate IDs given when operating the Phenix. Otherwise, they should match CellProfiler's understanding of the `Plate` grouping variable, whether that is explicitly stated in a loaddata CSV OR produced from the Metadata module if the CSVs and/or batch files are created using CellProfiler's input modules. For downstream purposes, i.e. `cytominer`, you may choose to use only so much of the plate name as you need to keep the plates unique (e.g. `SQ00015167` instead of `SQ00015167__2016-04-21T03_34_00-Measurement1` to keep the names compact.

```sh
mkdir -p ~/efs/${PROJECT_NAME}/workspace/scratch/${BATCH_ID}/

PLATES=$(readlink -f ~/efs/${PROJECT_NAME}/workspace/scratch/${BATCH_ID}/plates_to_process.txt)

FULL_PLATES=$(readlink -f ~/efs/${PROJECT_NAME}/workspace/scratch/${BATCH_ID}/full_plates_to_process.txt)

ls ~/efs/${PROJECT_NAME}/workspace/images/${BATCH_ID}/ | cut -d '_' -f 1 >> $PLATES

ls ~/efs/${PROJECT_NAME}/workspace/images/${BATCH_ID}/ >> $FULL_PLATES
```

Check that your plate names are correct by `nano $PLATES` and `nano $FULL_PLATES`. If your plate names contain underscores, you may need to fix the simplified plate names. Both should have the same number of rows as the number of plates in your batch.

`SAMPLE_PLATE_ID=PLATE_NAME_1` This can be any single plate name, again using the portion of the name that is before the double underscore `__`.

`SAMPLE_FULL_PLATE_NAME=FULL_PLATE_NAME_1` This can be any single plate name, this time using the full name.

## Create LoadData CSVs

(if using `pe2loaddata` to create image sets)

The script below works only for Phenix microscopes -- it reads a standard XML file (`Index.idx.xml`) and writes a LoadData CSV file. For other microscopes, you will have to roll your own (see the overview chapter for more information). The script below requires `config.yml`, which specifies

1. The mapping between channel names in `Index.idx.xml` and the channel names in the CellProfiler pipelines
2. Metadata to extract from `Index.idx.xml`

Here's a truncated sample `config.yml` (here is an example of the [full file](https://github.com/broadinstitute/pe2loaddata/blob/220ac512bfc0c2e582d379b19411c1585272aee3/config.yml))

```
channels:
    HOECHST 33342: OrigDNA
    Alexa 568: OrigAGP
    Alexa 647: OrigMito
    Alexa 488: OrigER
    488 long: OrigRNA
    Brightfieldlow: OrigBrightfield
metadata:
    Row: Row
    Col: Col
    FieldID: FieldID
    PlaneID: PlaneID
    ChannelID: ChannelID
    ChannelName: ChannelName
    ImageResolutionX: ImageResolutionX
    [...]
```

Often, the values of the keys for channels are different in
`Index.idx.xml`, so for example, above, we have
`Brightfieldlow: OrigBrightfield` but the keys for channels could be
different in `Index.idx.xml`:

```
$ tail -n 500 ~/efs/${PROJECT_NAME}/workspace/images/${BATCH_ID}/${SAMPLE_FULL_PLATE_NAME}/Images/Index.idx.xml|grep ChannelName|sort -u

        <ChannelName>488 long</ChannelName>
        <ChannelName>Alexa 488</ChannelName>
        <ChannelName>Alexa 568</ChannelName>
        <ChannelName>Alexa 647</ChannelName>
        <ChannelName>Brightfield CP</ChannelName>
        <ChannelName>HOECHST 33342</ChannelName>
```

Copy the text into a text file somewhere on your computer so you can refer to it.

Now navigate to your pe2loaddata repository and ensure that it is up to date

``` sh
cd ~/efs/${PROJECT_NAME}/workspace/software/pe2loaddata

git pull

pyenv shell 3.8.10

pip3 install -e . 
```

Adjust any discrepancies between the list of channels from your index file and the config by editing `config.yml`:

```
HOECHST 33342: OrigDNA
Alexa 568: OrigAGP
Alexa 647: OrigMito
Alexa 488: OrigER
488 long: OrigRNA
Brightfield CP: OrigBrightfield
```

```{note}
- Ensure that all the metadata fields defined in `config.yml` are present in the `Index.idx.xml`. 
- Ensure that the channel names are the same in `config.yml` and `Index.idx.xml` 
- Ensure that the LoadData csv files don't already exist; if they do, delete them. 
- The `max-procs` option is set as 1 because pe2loaddata accesses the image files on `s3fs`, which doesn't handle multiple requests well. 
- If your images require Z projection, make sure that `sub-string-in` is set to the folder that you soft-linked to in the previous step.
```

```sh
pyenv shell 3.8.10
parallel \
  --link \
  --max-procs 1 \
  --eta \
  --joblog ../../log/${BATCH_ID}/create_csv_from_xml.log \
  --results ../../log/${BATCH_ID}/create_csv_from_xml \
  --files \
  --keep-order \
  pe2loaddata config.yml \
    ~/efs/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID}/{1}/load_data.csv \
    --index-directory ~/efs/${PROJECT_NAME}/workspace/images/${BATCH_ID}/{2}/Images \
    --illum \
    --illum-directory /home/ubuntu/bucket/projects/${PROJECT_NAME}/${BATCH_ID}/illum/{1} \
    --plate-id {1} \
    --illum-output ~/efs/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID}/{1}/load_data_with_illum.csv \
    --sub-string-out efs/${PROJECT_NAME}/workspace/images/${BATCH_ID} \
    --sub-string-in bucket/projects/${PROJECT_NAME}/${BATCH_ID}/images :::: ${PLATES} ${FULL_PLATES}
```

This is the resulting structure of `load_data_csv` on EFS (one level below `workspace`). Files for only `SQ00015167` are shown.

```
└── load_data_csv
    └── 2016_04_01_a549_48hr_batch1
        └── SQ00015167
            ├── load_data.csv
            └── load_data_with_illum.csv
```

`load_data.csv` will be used by `illum.cppipe` and, optionally, `qc.cppipe`. `load_data_with_illum.csv` will be used by `analysis.cppipe` and, optionally, `assaydev.cppipe`. When creating `load_data_with_illum.csv`, the script assumes a specific location for the folder containing the illumination correction files.

```{note}
If your files must be Z projected, your load_data.csv will be correct for that step. Once that step is executed, edit BOTH of your CSVs to ensure that

- The `Orig` image paths are updated to the location of the projected files rather than than the unprojected files
- Only the last-numbered plane from each site are kept

These steps can be done manually in e.g. in Excel but are easier to script for large numbers of plates.\
You should then upload your edited CSVs to S3.
```

## Upload image location files to S3

If using `pe2loaddata`, run the command below

Copy the load data files to S3:

```sh
aws s3 sync \
    ~/efs/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID}/ \
    s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID}/
```

If using your own home-created load data CSVs, load them to the same location - `s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/load_data_csv/${BATCH_ID}` - we strongly recommend the structure of making one subfolder with CSVs in it per plate and then using the names `load_data.csv` for CSVs without illum files and `load_data_with_illum.csv` for those with the illum files.

If using batch files, we recommend uploading them to `s3://${BUCKET}/projects/${PROJECT_NAME}/workspace/batchfiles/${BATCH_ID}`, giving each batch file a unique name to reflect which step it is for - since the process of creating batchfiles can be onerous, subsequent steps assume you will make one batchfile per batch rather than per plate, but you may adjust this if you like.
