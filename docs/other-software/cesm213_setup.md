# First-Time setup of CESM 2.1.3

Due to the nature of the CESM program, a centrally installed version of the code is not provided on ARCHER2. Instead, a user needs to download and set up the program themselves in their `/work` area. The installation is done in three steps:

1. [Download the code and set up the directory structure](#downloading-cesm-2.1.3-and-setting-up-the-directory-structure)
2. [Link and Download Components](#linking-and-downloading-components)
3. [Build CPRNC](#building-cprnc)

After setup, CESM is ready to run a simple case.

## Downloading CESM 2.1.3 And Setting Up The Directory Structure

For ease of use, a setup script has been created which downloads CESM 2.1.3, creates the directory structure needed for running CESM2 cases and creates a hidden file in your home directory containing environment variables needed by CESM.

To execute this script, run the following in an archer2 terminal

```bash
source /work/n02/shared/CESM2/setup_cesm213.sh
```

This script will create a directory, defaulting to `/work/$GROUP/$GROUP/$USER/cesm/CESM2.1.3`, where `$GROUP` is your default group, for example n02, and populate it with the following subdirectories:
* `archive` - short-term archiving for completed runs,
* `ccsm_baselines` - baseline files,
* `cesm_inputdata` - input data downloaded and used when running cases,
* `runs` - location of the case files used when running a case,
* cesm directory - location of the cesm source code and the various components. Defaults to `my_cesm_sandbox`

The default locations for the CESM root directory and the CESM location can be overridden during installation either by entering new paths at runtime when prompted or by providing them as command line arguments, for example

```bash
source /work/n02/shared/CESM2/setup_cesm213.sh -p /work/n03/n03/$USER/CESM213 -l cesm_prog
```

### Manual setup instructions

If you have trouble with running the setup script, you can install manually by running the following commands:

```bash
PREFIX="path/to/your/desired/cesm/root/location"
CESM_DIR_LOC="name_of_install_directory_for_cesm"

mkdir -p $PREFIX
cd $PREFIX
mkdir -p archive
mkdir -p ccsm_baselines
mkdir -p cesm_inputdata
mkdir -p runs

CESM_LOC=$PREFIX/$CESM_DIR_LOC

git clone -b release-cesm2.1.3  https://github.com/ESCOMP/CESM.git $CESM_LOC
cd $CESM_LOC
git checkout release-cesm2.1.3

tee ${HOME}/.cesm213 <<EOF > /dev/null
### CESM 2.1.3 on ARCHER2 Path File
### Do Not Edit This File Unless You Know What You Are Doing
CIME_MODEL=cesm
CESM_ROOT=$PREFIX
CESM_LOC=$PREFIX/$CESM_DIR_LOC
CIMEROOT=$PREFIX/$CESM_DIR_LOC/cime
EOF

echo "module use /work/n02/shared/CESM2/module" >> ~/.bashrc
module use /work/n02/shared/CESM2/module
module load CESM2/2.1.3
```

## Linking And Downloading Components

CESM utilises multiple components, including CAM (atmosphere), CICE (sea ice), CISM (ice sheets), CTSM (land), MOSART (adaptive river transport), POP2 (ocean), RTM (river transport) and WW3 (waves), all of which are connected using the Common Infrastructure for Modelling the Earth (CIME). These components are hosted on github, and during the setup process they are downloaded.

Before downloading the external components, you must first modify the file `$CESM_LOC/Externals.cfg`. This will change the version of CIME from the default cime 5.6.32 to the maintained cime 5.6 branch. This is done by modifying the file so that the cime section goes from

```bash
[cime]
tag = cime5.6.32
protocol = git
repo_url = https://github.com/ESMCI/cime
local_path = cime
required = True
```

to

```bash
[cime]
branch = maint_5.6_archer2_port
protocol = git
repo_url = https://github.com/cemac-ccs/cime
local_path = cime
required = True
```

By making this change, the configurations for archer2 are brought in along with some bug fixes

Once this has been done you are free to download the external components by executing the commands

```bash
cd $CESM_LOC
./manage_externals/checkout_externals
```

The first time you run the checkout_externals script, you may be asked to accept a certificate, and you may also get an error of the form

```
    svn: E120108: Error running context: The server unexpectedly closed the connection.
```
If this happens, rerun the checkout_externals script and it should download the external components correctly.

## Building cprnc

cprnc is a generic tool for analyzing a netcdf file or comparing two netcdf files. It is used in various places by CESM and the source is included with cime.

To build, execute the following commands

```bash
module load CESM2/2.1.3
cd $CIMEROOT/tools/cprnc
../configure --macros-format=Makefile --mpilib=mpi-serial
sed -i '/}}/d' .env_mach_specific.sh
source ./.env_mach_specific.sh && make
```

It is likely you will see a warning message of the form

```
The following dependent module(s) are not currently loaded: cray-hdf5-parallel (required by: CESM2/2.1.3), cray-netcdf-hdf5parallel (required by: CESM2/2.1.3), cray-parallel-netcdf (required by: CESM2/2.1.3)
```

This is due to serial netCDF and hdf5 libraries being loaded as a result of the `--mpilib=mpi-serial` flag. This warning message is safe to ignore.

In a small number of cases you may also see a warning of the form

```bash
-bash: export: '}}': not a valid identifier
```

This warning should also be safe to ignore, but can be solved by opening the file `./.env_mach_specific.sh` in a text editor and commenting out or deleting the line

```bash
export OMP_NUM_THREADS={{ thread_count }}
```

Then rerunning the command

```bash
source ./.env_mach_specific.sh && make
```

Once this step has been completed, you are ready to run a [simple test case](cesm213_run.md).