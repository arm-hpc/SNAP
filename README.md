SNAP: SN (Discrete Ordinates) Application Proxy
===============================================

Description
-----------

SNAP serves as a proxy application to model the performance of a modern discrete ordinates neutral particle transport application. SNAP may be considered an update to [Sweep3D](http://www.ccs3.lanl.gov/PAL/software.shtml), intended for hybrid computing architectures. It is modeled off the Los Alamos National Laboratory code PARTISN. PARTISN solves the linear Boltzmann transport equation (TE), a governing equation for determining the number of neutral particles (e.g., neutrons and gamma rays) in a multi-dimensional phase space. SNAP itself is not a particle transport application; SNAP incorporates no actual physics in its available data, nor does it use numerical operators specifically designed for particle transport. Rather, SNAP mimics the computational workload, memory requirements, and communication patterns of PARTISN. The equation it solves has been composed to use the same number of operations, use the same data layout, and load elements of the arrays in approximately the same order. Although the equation SNAP solves looks similar to the TE, it has no real world relevance.

The solution to the time-dependent TE is a "flux" function of seven independent variables: three spatial (3-D spatial mesh), two angular (set of discrete ordinates, directions in which particles travel), one energy (particle speeds binned into "groups"), and one temporal. PARTISN, and therefore SNAP, uses domain decomposition over these dimensions to coherently distribute the data and the tasks associated with solving the equation. The parallelization strategy is expected to be the most efficient compromise between computing resources and the iterative strategy necessary to converge the flux.

The iterative strategy is comprised of a set of two nested loops. These nested loops are performed for each step of a time-dependent calculation, wherein any particular time step requires information from the preceding one. No parallelization is performed over the temporal domain. However, for time-dependent calculations two copies of the unknown flux must be stored, each copy an array of the six remaining dimensions. The outer iterative loop involves solving for the flux over the energy domain with updated information about coupling among the energy groups. Typical calculations require tens to hundreds of groups, making the energy domain suitable for threading with the node's (or nodes') provided accelerator. The inner loop involves sweeping across the entire spatial mesh along each discrete direction of the angular domain. The spatial mesh may be immensely large. Therefore, SNAP spatially decomposes the problem across nodes and communicates needed information according to the KBA method. KBA is a transport-specific application of general parallel wavefront methods. Lastly, although KBA efficiency is improved by pipelining operations according to the angle, current chipsets operate best with vectorized operations. During a mesh sweep, SNAP operations are vectorized over angles to take advantage of the modern hardware.

SNAP should be tested with problem sizes that accurately reflect the types of calculations PARTISN frequently handles. The spatial domain shall be decomposed to 2,000--4,000 cells per node (MPI rank). Each node will own all the energy groups and angles for that group of cells; typical calculations feature 10--100 energy groups and as few as 100 to as many as 2,000 angles. Moreover, sufficient memory must be provided to store two full copies of the solution vector for time-dependent calculations. The preceding parameters assume current trends in available per core memory. Significant advances or detriments affecting this assumption shall require reconsideration of appropriate parameters per compute node.

Compilation
-----------

SNAP has been written to the Fortran 90/95 standard. It has been successfully built with, but not necessarily limited to, gfortran and ifort. Moreover, the code has been built with the profiling tool [Byfl](https://github.com/losalamos/byfl). The accompanying Makefile retains some of the old make options for different build types. However, the current build system depends on the availability of MPI and OpenMP libraries. Builds without these libraries will require modification to the source code to remove related subroutine calls and directives.

MPI implementations typically suggest using a "wrapper" compiler to compile the code. SNAP has been built and tested with OpenMPI. OpenMPI allows one to set the underlying Fortran compiler with the environment variable OMPI_FC, where the variable is set to the (path and) compiler of choice, e.g., ifort, gfortran, etc.

The makefile currently uses:

    FORTRAN = mpif90

and all testing has been performed with

    OMPI_FC = [path]/ifort

Fortran compilation flags can be set according to the underlying compiler. The current flags are set for the ifort compiler and using OpenMP for parallel threading.

    TARGET = snap
    FFLAGS = -03 -openmp
    FFLAG2 =

where `FFLAG2` is reserved for additional flags that may need applied differently, depending on the compiler. To make SNAP with these default settings, simply type

    make

on the command line within the SNAP directory.

A debugging version of SNAP can be built by typing

    make OPT=no

on the command line. The unoptimized, debugging version of SNAP features bounds checking, back-tracing an error, and the necessary debug compiler flags. With ifort, these flags appear as:

    FFLAGS = -g -O0 -check bounds -traceback -openmp
    FFLAG2 =

The values for these compilation variables have been modified for various Fortran compilers and the Makefile provides details of what has been used previously. These lines are commented out for clarity at this time and to ensure that changes to the build system are made carefully before attempting to rebuild with a different compiler.

The SNAP directory can be cleaned up of its module and object files if the user desires with:

    make clean

This removes all the `*.mod` and `*.o` files, as well as `*.bc` files from Byfl builds. Moreover, it will enforce complete recompilation of all files upon the next instance of `make` or `make OPT=no`. Currently, there is no separate directory for the compilation files of separate optimized and unoptimized builds. The user must do a `make clean` before building the code if the previous build used the opposite command.

Lastly, a line count report is generated with:

    make count

The line count report excludes blank lines and comments. It counts the number of code lines in all `*.f90` files and sums the results. The information is printed to the the `Lines` file.

Usage
-----

Because SNAP currently requires building with MPI, to execute SNAP, use the following command:

    mpirun -np [#] [path]/snap [infile] [outfile]

This command will automatically run with the number of threads specified by the input file, which is used to set the number of OpenMP threads, overwriting any environment variable to set the number of threads. Testing has shown that to ensure proper concurrency of work, the above command can be modified to

    mpirun -cpus-per-proc [#threads] -np [#procs] [path]/snap [infile] [outfile]

Lastly, a user may wish to test the various thread affinity settings used to bind threads to processing elements. Testing has been done with a disabled Intel thread affinity interface.

    setenv KMP_AFFINITY disabled (csh)

The command line is read for the input/output file names. If one of the names is missing, the code will not execute. Moreover, the output file overwrites any pre-existing files of the same name.

Sample Input
------------

The following is a sample input of a SNAP job. Several other examples are provided as part of the small set of regression testing. For more information about the valid range of values and descriptions of the input variables, please see the user manual.

    ! Input from namelist
    &invar
      nthreads=2
      npey=2
      npez=2
      ndimen=3
      nx=4
      lx=1.0
      ny=4
      ly=1.0
      nz=4
      lz=1.0
      ichunk=4
      nmom=2
      nang=8
      ng=2
      mat_opt=0
      src_opt=0
      timedep=0
      it_det=0
      tf=0.0
      nsteps=1
      iitm=5
      oitm=100
      epsi=1.E-4
      fluxp=0
      scatp=0
      fixup=0
    /

Sample Output
-------------

The following is the corresponding output to the above case. A brief outline of the output file contents is version and run time information, echo of input (or default) values of the namelist variables, echo of relevant parameters after setup, iteration monitor, mid-plane flux output, and the timing summary. Warning and error messages may be printed throughout the output file to alert the user to some problem with the execution. Unlike errors, warnings do not cause program termination.

     SNAP: SN (Discrete Ordinates) Application Proxy
     Version Number..  1.00 
     Version Date..  03-07-2013
     Ran on  3-13-2013 at time 14: 5:39
    
    ********************************************************************************
    
              Input Echo - Values from input or default
    ********************************************************************************
    
      NML=invar
         npey=     2
         npez=     2
         ichunk=     4
         nthreads=     2
         ndimen=  3
         nx=     4
         ny=     4
         nz=     4
         lx=  1.0000E+00
         ly=  1.0000E+00
         lz=  1.0000E+00
         nmom=   2
         nang=    8
         ng=    2
         mat_opt=  0
         src_opt=  0
         scatp=  0
         epsi=  1.0000E-04
         iitm=   5
         oitm=  100
         timedep=  0
         tf=  0.0000E+00
         nsteps=     1
         it_det=  0
         fluxp=  0
         fixup=  0
    
    ********************************************************************************
    
              Calculation Run-time Parameters Echo
    ********************************************************************************
    
      Geometry
        ndimen = 3
        nx =     4
        ny =     4
        nz =     4
        lx =  1.0000E+00
        ly =  1.0000E+00
        lz =  1.0000E+00
        dx =  2.5000E-01
        dy =  2.5000E-01
        dz =  2.5000E-01
    
      Sn
        nmom = 2
        nang =    8
        noct = 8
    
        w =  1.5625E-02   ... uniform weights
    
              mu              eta               xi
         6.25000000E-02   9.37500000E-01   3.42326598E-01
         1.87500000E-01   8.12500000E-01   5.51985054E-01
         3.12500000E-01   6.87500000E-01   6.55505530E-01
         4.37500000E-01   5.62500000E-01   7.01560760E-01
         5.62500000E-01   4.37500000E-01   7.01560760E-01
         6.87500000E-01   3.12500000E-01   6.55505530E-01
         8.12500000E-01   1.87500000E-01   5.51985054E-01
         9.37500000E-01   6.25000000E-02   3.42326598E-01
    
      Material Map
        mat_opt = 0   -->   nmat = 1
        Base material (default for every cell) = 1
        
      Source Map
        src_opt = 0
        Source strength per cell (where applied) = 1.0
        Source map:
            Starting cell: (     1,     1,     1 )
            Ending cell:   (     4,     4,     4 )
    
      Pseudo Cross Sections Data
        ng =   2
    
        Material 1
        Group         Total         Absorption      Scattering
           1       1.000000E+00    5.000000E-01    5.000000E-01
           2       1.010000E+00    5.050000E-01    5.050000E-01
    
      Solution Control Parameters
        epsi =  1.0000E-04
        iitm =   5
        oitm =  100
        timedep = 0
        it_det = 0
        fluxp = 0
        fixup = 0
    
    
      Parallelization Parameters
        npey =     2
        npez =     2
        nthreads =    2
    
              Thread Support Level
               0 - MPI_THREAD_SINGLE
               1 - MPI_THREAD_FUNNELED
               2 - MPI_THREAD_SERIALIZED
               3 - MPI_THREAD_MULTIPLE
        thread_level =  0
    
    ********************************************************************************
    
              Iteration Monitor
    ********************************************************************************
      Outer
        1    Dfmxo= 5.5956E-01    No. Inners=   10
        2    Dfmxo= 1.3849E-01    No. Inners=    8
        3    Dfmxo= 2.8164E-03    No. Inners=    6
    
      No. Outers=   3    No. Inners=   24
    
    ********************************************************************************
    
              Calculation Final Scalar Flux Solution
    ********************************************************************************
    
     ***********************************
      Group=   1   Z Mid-Plane=    3
     ***********************************
    
         y    x    1      x    2      x    3      x    4
         4  3.1216E-01  3.9666E-01  3.9666E-01  3.1216E-01
         3  3.9683E-01  5.0638E-01  5.0638E-01  3.9683E-01
         2  3.9683E-01  5.0638E-01  5.0638E-01  3.9683E-01
         1  3.1216E-01  3.9666E-01  3.9666E-01  3.1216E-01
    
    ********************************************************************************
    
    
     ***********************************
      Group=   2   Z Mid-Plane=    3
     ***********************************
    
         y    x    1      x    2      x    3      x    4
         4  3.8033E-01  4.9130E-01  4.9130E-01  3.8033E-01
         3  4.9210E-01  6.3838E-01  6.3838E-01  4.9210E-01
         2  4.9210E-01  6.3838E-01  6.3838E-01  4.9210E-01
         1  3.8033E-01  4.9130E-01  4.9130E-01  3.8033E-01
    
    ********************************************************************************
    
              Timing Summary
    ********************************************************************************
    
      Code Section                          Time (seconds)
     **************                        ****************
        Parallel Setup                       1.0433E-03
        Input                                4.7398E-04
        Setup                                5.8007E-04
        Solve                                2.2089E-03
           Parameter Setup                   2.1720E-04
           Outer Source                      1.8597E-05
           Inner Iterations                  1.8098E-03
              Inner Source                   3.0279E-05
              Transport Sweeps               1.4706E-03
              Inner Misc Ops                 3.0899E-04
           Solution Misc Ops                 1.6332E-04
        Output                               2.6178E-04
      Total Execution time                   1.5435E-02
    
      Grind Time (nanoseconds)         2.2471E+01
    
    ********************************************************************************
    
Additional outputs in the form of `slgg` and `flux` files are available when requested according to the `scatp` and `fluxp` input variables, respectively.

License
-------

Los Alamos National Security, LLC owns the copyright to "SNAP: SN (Discrete Ordinates) Application Proxy, Version 1.x (C13087)". The license is BSD with standard clauses regarding indicating modifications before future redistribution:

Copyright (c) 2013, Los Alamos National Security, LLC
All rights reserved.

Copyright 2013. Los Alamos National Security, LLC. This software was produced under U.S. Government contract DE-AC52-06NA25396 for Los Alamos National Laboratory (LANL), which is operated by Los Alamos National Security, LLC for the U.S. Department of Energy. The U.S. Government has rights to use, reproduce, and distribute this software. NEITHER THE GOVERNMENT NOR LOS ALAMOS NATIONAL SECURITY, LLC MAKES ANY WARRANTY, EXPRESS OR IMPLIED, OR ASSUMES ANY LIABILITY FOR THE USE OF THIS SOFTWARE. If software is modified to produce derivative works, such modified software should be clearly marked, so as not to confuse it with the version available from LANL.

Additionally, redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
* Neither the name of Los Alamos National Security, LLC, Los Alamos National Laboratory, LANL, the U.S. Government, nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY LOS ALAMOS NATIONAL SECURITY, LLC AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL LOS ALAMOS NATIONAL SECURITY, LLC OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.


Classification
--------------

SNAP is Unclassified and contains no Unclassified Controlled Nuclear Information. It has been assigned Los Alamos Computer Code number LA-CC-13-016.

Authors
-------

* Joe Zerr, CCS-2, Los Alamos National Laboratory
* Randal Baker, CCS-2, Los Alamos National Laboratory

Questions and issues: snap@lanl.gov

