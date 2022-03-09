# (PART) Configuration {-}

# Configure Environment for Full Profiling Pipeline

This workflow assumes you have already set up an AWS account with an S3 bucket and EFS, and created a VM per the instructions in the link below.

## Launch an AWS Virtual Machine for making CSVs and running Distributed-CellProfiler

Launch an EC2 node using AMI `cytomining/images/hvm-ssd/cytominer-bionic-trusty-18.04-amd64-server-*`, created using [cytominer-vm](https://github.com/cytomining/cytominer-vm).

You will need to create an AMI for your own infrastructure because the provisioning includes mounting S3 and EFS, which is account specific.
We recommend using an `m4.xlarge` instance, with an 8Gb EBS volume.

Note: Proper configuration is essential to mount the S3 bucket.
The following configuration provides an example, named `imaging-platform` (modifications will be necessary).

  * Launch an ec2 instance on AWS
  * AMI: `cytomining/images/hvm-ssd/cytominer-ubuntu-trusty-18.04-amd64-server-1529668435`
  * Instance Type: m4.xlarge
  * Network: vpc-35149752 
  * Subnet: Default (imaging platform terraform)
  * IAM role: `s3-imaging-platform-role`
  * No Tags
  * Select Existing Security Group: `SSH_HTTP`
  * Review and Launch
  * `ssh -i <USER>.pem ubuntu@<Public DNS IPv4>`

After starting the instance, ensure that the S3 bucket is mounted on `~/bucket`.
If not, run `sudo mount -a`.


Log in to the EC2 instance.


Enter your AWS credentials

```sh
aws configure
```

The infrastructure is configured with one S3 bucket.
Mount this S3 bucket (if it is not automatically mounted)

```sh
sudo mount -a
```

Check that the bucket was was mounted.
This path should exist:

```sh
ls ~/bucket/projects
```

## Create a tmux session

You will want to retain environment variables once defined, and for processes to run when you are not connected, so you should create a tmux session to work in

```sh
tmux new -s sessionname
```

You can detach from this session at any time by typing `Ctl+b`, then `d`.  
To reattach to an existing session, type `tmux a -t sessionname`

You can list existing sessions with `tmux list-sessions` and kill any poorly-behaving session with `tmux kill-session -t sessionname`

## Define Environment Variables

These variables will be used throughout the project to tag instances, logs etc so that you know which machines are working on what, which files to operate on, where your logs are, etc.

```sh
PROJECT_NAME=2015_10_05_DrugRepurposing_AravindSubramanian_GolubLab_Broad

BATCH_ID=2016_04_01_a549_48hr_batch1

BUCKET=imaging-platform

MAXPROCS=3 # m4.xlarge has 4 cores; this should be # of cores on your instance - 1
```

## Create Directories

```sh
mkdir -p ~/efs/${PROJECT_NAME}/workspace/

cd ~/efs/${PROJECT_NAME}/workspace/

mkdir -p log/${BATCH_ID}
```

## Download Software

```sh
cd ~/efs/${PROJECT_NAME}/workspace/
mkdir software
cd software
git clone https://github.com/broadinstitute/pe2loaddata.git
git clone https://github.com/CellProfiler/Distributed-CellProfiler.git

cd ..
```

If these repos have already been cloned, `git pull` to make sure they are up to date.

This is the resulting structure of `software` on EFS (one level below `workspace`):
```
└── software
    ├── Distributed-CellProfiler
    └── pe2loaddata
```
