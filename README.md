# Falcon User Guide

User guide for the C3+3 Falcon HPC system, markdown format. 

## Getting started

It's possible to connect to Falcon through the traditional command line interface or through the Open OnDemand web interface, which is recommended, by pointing a browser to https://ondemand.c3plus3.org and providing your multifactor credentials. 

Select a desktop or terminal session, based on your needs.

### Glossary

- login node
- compute node

## The LMOD module system

Falcon uses the lua-based LMOD module system. If you are already familiar with TCL modules a.k.a. 'environment modules' then LMOD will mostly be familiar. If you have developed your own modules in TCL, rest assured knowing that you will still be able to write and use these under LMOD. The primary feature of LMOD is the use of module hierarchies, which exist primarily to organize large module sets keep users from inadvertently loading incompatible modules. 

On Falcon, we use LMOD to support a two-tier hierarchy: Compilers are the first tier and MPI is the second tier. Specifically, we support gcc and Intel compilers, and the MPICH and OpenMPI implementations of MPI. What this means is that you will only be able to have one compiler and one MPI implementation loaded at any given time. Further, with a given compiler/MPI combination loaded, only modules built with that combination will be visible for loading.

For example, starting with a clean module environment, let's tell LMOD to use the directory containing Core modules and see what we get access to:

```
$ module purge
$ module use /lfs/software/modules/Core
$ module avail

----------------------------------------------------------------------------------------------------- /lfs/software/modules/Core ------------------------------------------------------------------------------------------------------
   gcc/12.1.0    intel/2021.4.0

```

Note that see only the compilers. Selecting gcc, and running again, we still see the available compilers, plus the packages built with those compilers, including the MPICH and OpenMPI packages:

```
$ module load gcc
$ module avail
-------------------------------------------------------------------------------------------------- /lfs/software/modules/gcc/12.1.0 ---------------------------------------------------------------------------------------------------
   mpich/3.4.3    openmpi/4.1.3    

----------------------------------------------------------------------------------------------------- /lfs/software/modules/Core ------------------------------------------------------------------------------------------------------
   gcc/12.1.0 (L)    intel/2021.4.0
```

Going one step further, we load the mpich module and again inspect what modules are available:

```
$ module load mpich
$ module avail 

-------------------------------------------------------------------------------------------- /lfs/software/modules/mpich/3.4.3/gcc/12.1.0 ---------------------------------------------------------------------------------------------
   fftw/3.3.10       hdf5/1.12.2    lammps/20220107    netcdf-c/4.8.1          opencoarrays/2.10.0    parallel-netcdf/1.12.2    wps/4.3.1 (D)
   gromacs/2022.2    icar/2.0       namd/2.14          netcdf-fortran/4.5.4    openfoam/2112          vasp/5.4.4.pl2            wrf/4.3.3

-------------------------------------------------------------------------------------------------- /lfs/software/modules/gcc/12.1.0 ---------------------------------------------------------------------------------------------------
   mpich/3.4.3 (L)    openmpi/4.1.3

----------------------------------------------------------------------------------------------------- /lfs/software/modules/Core ------------------------------------------------------------------------------------------------------
   gcc/12.1.0 (L)    intel/2021.4.0
```

We now see that a number of packages are available which are built with the gcc/mpich combination stack. Let's load a couple of these modules:

```
$ module load hdf5 gromacs
$ module list

Currently Loaded Modules:
  1) gcc/12.1.0   2) mpich/3.4.3   3) hdf5/1.12.2   4) fftw/3.3.10   5) gromacs/2022.2

```

Loading the gromacs caused the fftw module to be loaded automatically, since it's needed by GROMACS. Where it gets interesting is when we switch components, e.g. if we swap the gcc compiler for the Intel compiler:

```
$ module load intel

Lmod is automatically replacing "gcc/12.1.0" with "intel/2021.4.0".

Inactive Modules:
  1) gromacs

Due to MODULEPATH changes, the following have been reloaded:
  1) fftw/3.3.10     2) hdf5/1.12.2     3) mpich/3.4.3

```

What happened here? Why is GROMACS now inactive, and what does it mean that the other modules have been reloaded? Let's first examine our current module state:

```
$ module list

Currently Loaded Modules:
  1) intel/2021.4.0   2) mpich/3.4.3   3) hdf5/1.12.2   4) fftw/3.3.10

Inactive Modules:
  1) gromacs

```

Here is what's going on:
- There is no build of GROMACS with the Intel compiler, it's just not supported, so in switching compilers, there's no module for that combination, and LMOD marks it inactive. If you find yourself switching back and forth between module environments, you will find this helpful, instead of having to unload and reload modules every time you switch.
- The gcc compiler is no longer loaded, because we asked for the Intel compiler, and we're only allowed to have one compiler loaded at a time. This helps to prevents us from building or loading Frankenstein software stacks with diffent components compiled with different compilers (and MPIs). 
- The other modules we had loaded still appear, but LMOD has quietly replaced them with the versions of those packages built with the Intel compiler. Note I say quietly, not silently, as LMOD does mention that it is reloading them.

What if we don't know the modules we need to load, we just know what program we need to run? 
LMOD provides the spider command. If you are looking for lammps, you can simply:

```
$ module spider lammps

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  lammps: lammps/20220107
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    You will need to load all module(s) on any one of the lines below before the "lammps/20220107" module is available to load.

      gcc/12.1.0  mpich/3.4.3
      gcc/12.1.0  openmpi/4.1.3
      intel/2021.4.0  mpich/3.4.3
      intel/2021.4.0  openmpi/4.1.3
 
    Help:
      LAMMPS stands for Large-scale Atomic/Molecular Massively Parallel
      Simulator. This package uses patch releases, not stable release. See
      https://github.com/spack/spack/pull/5342 for a detailed discussion.

```

Upon examining the output, it's clear that there are four combinations of modules that can be used to load the different builds of LAMMPS which are available. For many codes, some builds will run faster when built with a particular compiler, or with a particular MPI, and it may actually change depending on how large your job size is. For this reason, and as much as it is possible, we try to provide a variety of different builds to users. 

That's the whirlwind tour of LMOD and modules on Falcon. Play with it until you're comfortable with it, you can't break anything. You can always `module purge` or simply log out and back in if it really gets muddy.  For the curious, there's more info about LMOD [here](https://lmod.readthedocs.io/en/latest/010_user.html).

Cheers and happy module-ing!

--The Admin Team

---

## Running interactive jobs


Falcon uses the slurm resource manager to meet the needs of multiple users sharing resources on the system. In it's simplest invocation, slurm can be called up to schedule an interactive job, wherein a user is able to allocate one or more compute nodes. ~~For example, the following one liner can used to launch a bash shell on a compute node: ` srun --nodes=1 --ntasks-per-node=1 --time=01:00:00 --pty bash -i` Note the parameters. This reserves a single node for 1 hour, and expects to run one MPI task on the node. The `bash` keyword here indicates that a new shell will be started there, and the `-i` indicates it will be interactive.~~ (srun integration not currently implemented) 

To launch an interactive session:
```$ /lfs/software/bin/interactive-session ```

This will give a single-node interactive session without a default duration of 12 hours.  In some instances, e.g. when the cluster is particularly busy, this command will block for a period of time until a node becomes available, but most of the time, it will start fairly quickly. Depending on your needs, reducing the number of hours (specify `-t time-in-hours`) requested may mean your session starts more quickly. 

Interactive sessions are particularly useful for debugging scripts that will eventually be run as batch jobs. Use an interactive session to run a job at smaller scale and for a shorter time, and once you feel confident that the script is running as intended, you can convert to a batch job. Note that a batch script can be run while in this interactive mode simply by sourcing it, i.e. 

```$ . /path/to/my/batch/script.sh```

---

## Running batch jobs

Once you've perfected your script and are reasonably sure it is behaving as you wish, it may be time to scale it up and run in batch mode. Here is an example batch script:

```
#!/bin/bash

#SBATCH --job-name=big_science
#SBATCH --partition=long
#SBATCH --cpus-per-task=1
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH -t 1:00:00 # run time (hh:mm:ss)
#SBATCH -o test-results-%j # output and error file name (%j expands to jobID)

module load gcc mpich

echo "Dumping lists of hosts: "
mpirun -n 4 hostname
echo "--------"

echo

echo "Preparing run:"
date

mpirun -n 4 /path/to/my_mpi_executable

echo "Run complete."
date
 
```

Running the batch script is as simple as submitting it to slurm for scheduling:
`sbatch /path/to/my/batch/script.sh`

Slurm will reply with a job number, and you can always check in on it with the slurm commands:
- squeue: See what jobs are running. `squeue -u some_username` will list jobs for a given user.
- sinfo: Get info on what queues are available, and their statuses. `sinfo --help` for more options

---

More information on slurm can be found in the (slurm user guide)[https://slurm.schedmd.com/quickstart.html].
