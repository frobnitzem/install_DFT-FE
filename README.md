# Install DFT-FE

These install scripts provide a set of executable
functions that install the necessary dependencies
of [DFT-FE](https://github.com/dftfeDevelopers/dftfe).

To use these scripts, we assume you have cloned this
repository onto a system where you intend to install DFT-FE.
I installed it into `/lustre/orion/stf006/world-shared/$USER/DFT-FE`,
for example.

## Pre-requisites

Because it's a better shell, the scripts are written
in the [rc](http://doc.cat-v.org/plan_9/4th_edition/papers/rc)
shell language.  Install `rc` by running

    cp src/rcrc $HOME/.rcrc
    . ./bin/getrc.sh $HOME/$LMOD_SYSTEM_NAME
    rc -l

Note that getrc installs into the `$HOME/$LMOD_SYSTEM_NAME/bin`
directory, and adds that to your PATH.

Copying the rcrc startup file to your home directory provides
the module command (in case your lmod version is old,
and doesn't yet recognize the rc shell).

## Module Environment

The module environment intended to run DFT-FE has been extracted
into `env2/env.rc`.  Edit this file before proceeding any further.
Make sure that your module environment contains some version of the
pre-requisites mentioned there (e.g. python3 and openblas).
This environment file is used both by the install and run
phases of DFT-FE.

## Running the installation

The installation itself is contained within the functions in
`dftfe2.rc`.  Edit this to define its WD and INST directories
to reflect your own environment.
Then source this script using

    . ./dftfe2.rc

and then run the functions listed in that file manually, in order.
For example, 

    install_alglib
    install_libxc
    install_spglib
    install_p4est
    install_scalapack
    # install_ofi_rccl # (optional, skip this one for now)
    install_elpa
    install_dealii
    enter_venv
    install_torch
    install_dftfe

Each function follows a standard pattern - download source into `$WD/src`,
patch, compile, and install into `$INST`.  It is HIGHLY recommended
to check all warnings and errors from these installs to be sure
you have not ended up with broken packages.

A few packages require interaction.  The elpa patch seems to have become
partially out-dated since mid-March, and `compile_mlxc` requires
appropriate git credentials.

## Running DFT-FE

Two different versions of DFT-FE are built from the steps above:
the GPU branch with ROCM and the GPU branch with ROCM and torch-based
MLXC.  Both are built in real and cplx versions, depending on whether you
want to enable k-points (implemented in the cplx version only).

Assuming you have already sourced `env2/env.rc`, an example
batch script running GPU-enabled DFT-FE is below:

    #!/usr/bin/env rc
    #SBATCH -A spy007
    #SBATCH -J dft14584
    #SBATCH -t 00:25:00
    #SBATCH -p batch
    #SBATCH -N 280
    #SBATCH --gpus-per-node 8
    #SBATCH --ntasks-per-gpu 1
    #SBATCH --gpu-bind closest

    OMP_NUM_THREADS = 1
    MPICH_OFI_NIC_POLICY = NUMA
    HSA_FORCE_FINE_GRAIN_PCIE = 1 
    LD_LIBRARY_PATH = $LD_LIBRARY_PATH:$INST/lib

    n=`{echo $SLURM_JOB_NUM_NODES '*' 8 | bc}

    srun -n $n -c 7 --gpu-bind closest \
              $INST/real/dftfe parameterFileGPU.prm > output

This uses `SLURM_JOB_NUM_NODES` to compute the number of MPI
ranks to use as one per GCD (8 per node).  If you wish to run
on a different number of nodes, only the `#SBATCH -N 280`
needs to be changed.
