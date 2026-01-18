# Batch Computing

The academic clusters that we have access to mostly have `apptainer` installed which we can use to run the images with ldmx-sw built into them.
We use `denv` when running the images manually and, fortunately, it is small enough to deploy onto the clusters as well.[^1]
```shell
# on the cluster you want to run batch jobs
curl -s https://tomeichlersmith.github.io/denv/install | sh
```

~~~admonish tip title="Image Storage"
While the `${HOME}` directory is large enough to hold the installation of `denv`,
they are usually much too small to hold copies of the images that we want to run.
For this reason, you will likely want to edit your shell configuration (e.g. `~/.bashrc`)
to change where `apptainer` will store the images.
Refer to your cluster's IT help or documentation to find a suitable place to hold these images.
For example, [the S3DF cluster at SLAC](https://s3df.slac.stanford.edu/#/reference?id=apptainer)
suggests using the `${SCRATCH}` variable they define for their users.
```shell
export APPTAINER_LOCALCACHEDIR=${SCRATCH}/.apptainer
export APPTAINER_CACHEDIR=${SCRATCH}/.apptainer
export APPTAINER_TMPDIR=${SCRATCH}/.apptainer
```
~~~

~~~admonish success title="Test"
With `denv` installed on the cluster, you should be able to run `denv` like normal manually.
For example, you can test run a light image that is fast to download.
```
denv init alpine:latest
denv cat /etc/os-release
# should say "Alpine" instead of the host OS
```
~~~

[^1]: The total disk footprint of a `denv` installation is 120KB.
This is plenty small enough to include in your `${HOME}` directory on most if not all clusters.
Additionally, most clusters share your `${HOME}` directory with the working nodes and so you don't even need to bother copying `denv` to where the jobs are being run.

## Preparing for Batch Running
The above instructions have you setup to run `denv` on the cluster just like you run `denv` on your own computer;
however, doing a few more steps is helpful to ensure that the batch jobs run reliably and efficiently.

### Pre-Building SIF Images
Under-the-hood, `apptainer` runs images from SIF files.
When `denv` runs using the image tage (e.g. `ldmx/pro:v4.2.3`), `apptainer` stores a copy of this image in a SIF file inside of the cache directory.
While the cache directory is distributed across the worker nodes on some clusters, it is not distributed on all clusters, so pre-building the image ourselves
into a known location is helpful.

The location for the image should be big enough to hold the multi-GB image (so probably not your `${HOME}` directory) _and_ needs to be shared with the computers that run the jobs.
Again, check with your IT or cluster documentation to see a precise location.
At SLAC's S3DF, `/sdf/group/ldmx` can be a good location (and may already have the image you need built!).
```
cd path/to/big/dir
apptainer build ldmx_pro_v4.2.3.sif docker://ldmx/pro:v4.2.3 # just an example, name the SIF file appropriately
```

## Running the SIF Image
How we run the image during the jobs depends on how the jobs are configured.
For the clusters I have access to (UMN and SLAC), there are two different ways for jobs to be configured
that mainly change _where_ the job is run.

~~~admonish success title="Check Where Jobs are Run"
A good way to figure this out (and learn about the batch job system that you want to use)
is to figure out how to run a job that just runs `pwd`.
This command prints out the "present working directory" and so you can see where
the job is being run from.

Refer to your cluster's IT, documentation, and the batch job system's documentation to
learn how to do this.
~~~

#### Jobs Run In Submitted Directory
At SLAC S3DF, the jobs submitted with `sbatch` are run from the directory where `sbatch` was run.
This makes it rather easy to run jobs.
We can create a denv and then submit a job running `denv` from within that directory.
```
cd batch/submit/dir
denv init /full/path/to/big/dir/ldmx_pro_v4.2.3.sif
```

For example, submitting jobs for a range of run numbers would look like
```shell
mkdir log # the SBATCH commands in submit put the log files here
sbatch --array=0-10 submit.sh
```
with
```bash
#!/bin/bash
#SBATCH --job-name my-job
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=2g
#SBATCH --time=04:00:00 # time limit for jobs
#SBATCH --output=log/%A-%a.log
#SBATCH --error=log/%A-%a.log

set -o errexit
set -o nounset

# assume the configuration script config.py takes one argument
# the run number it should use for the simulation
# and then uniquely creates the path of the output file here
denv fire config.py ${SLURM_ARRAY_TASK_ID}
# fire is run inside ldmx/pro:v4.2.3 IF SUBMITTED FROM batch/submit/dir
```
Look at the SLAC S3DF and Slurm documentation to learn more about configuring the batch jobs themselves.

~~~admonish comments title="Comments"
- _Technically_, since SLAC S3DF's `${SCRATCH}` directory is also shared across the worker nodes, you do not need to pre-build the image. However, this is not advised because if the `${SCRATCH}` directory is periodically cleaned during your jobs, the cached SIF image would be lost and your jobs could fail in confusing ways.
- Some clusters configure Slurm to limit the number of jobs you can submit at once with `--array`. This means you might need to submit the jobs in "chunks" and add an offset to `SLURM_ARRAY_TASK_ID` so that the different "chunks" have different run numbers. This can be done with bash's math syntax e.g. `$(( SLURM_ARRAY_TASK_ID + 100 ))`.
~~~

#### Jobs Run in Scratch Directory
At UMN's CMS cluster, the jobs submitted with `condor_submit` are run from a newly-created scratch directory.
This makes it slightly difficult to inform `denv` of the configuration we want to use.
`denv` has an experimental shebang syntax that could be helpful for this purpose.

`prod.sh`
```bash
#!/full/path/to/denv shebang
#!denv_image=/full/path/to/ldmx_pro_v4.2.3.sif
#!bash

set -o nounset
set -o errexit

# everything here is run in `bash` inside ldmx/pro:v4.2.3
# assume run number is provided as an argument
fire config.py ${1}
```

with the submit file `submit.sub` in the same directory.
```
# run prod.sh and transfer it to scratch area
executable = prod.sh
transfer_executable = yes

# terminal and condor output log files
#   helpful for debugging at slight performance cost
output = logs/$(run_number)-$(Cluster)-$(Process).out
error  = $(output)
log    = $(Cluster)-condor.log

# "hold" the job if there is a non-zero exit code
#   and store the exit code in the hold reason subcode
on_exit_hold = ExitCode != 0
on_exit_hold_subcode = ExitCode
on_exit_hold_reason = "Program exited with non-zero exit code"

# the 'Process' variable is an index for the job in the submission cluster
arguments = "$(Process)"
```
And then you would `condor_submit` this script with
```shell
condor_submit submit.sub --queue 10
```

~~~admonish note collapsible=true title="Alternative Script Design"
Alternatively, one could write a script _around_ `denv` like
```shell
#!/bin/bash

set -o nounset
set -o errexit

# stuff here is run outside ldmx/pro:v4.2.3
# need to call `denv` to go into image
denv init /full/path/to/ldmx_pro_v4.2.3.sif
denv fire config.py ${1}
```
The `denv init` call writes a few small files which shouldn't have a large impact on performance
(but could if the directory in which the job is being run has a slow filesystem).
This is helpful if your configuration of HTCondor does not do the file transfer for you and
your job is responsible for copying in/out any input/output files that are necessary.
~~~

~~~admonish note title="Comments"
- Similar to Slurm's `--array`, we are relying on HTCondor's `-queue` command to decide what run numbers to use. Look at HTCondor's documentation (for example [Submitting many similar jobs with one queue command](https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html#submitting-many-similar-jobs-with-one-queue-command)) for more information.
~~~
