

# ELMFIRE 

The ELMFIRE full f nonlinear gyrokinetic code calculates the evaluation of the whole distribution function for ions and electrons and is thereby very well suitable to study turbulent transport in strongly perturbed plasmas including finite orbit effects, steep gradients and rapid dynamic changes. The code is based on a particle-in-cell algorithm which solves the equations of motion averaged over the gyro-orbit. This simplifies the calculation while keeping the time and length resolutions high enough for turbulence study. The guiding-center equations involve E×B, gradient and curvature drifts, collisions and polarization drifts. Sampling is performed for every grid cell at every time step and the time series of the density, temperature and potential are obtained. 

Originally the code was based on pure MPI parallelism, however
recently some rudimentary OpenMP was added on to it in an attempt to
reduce the memory footprint of the code. 

## Porting 

### Platform

The code has been ported to be run on a test platform with the Intel
Knights Landing (KNL) processor, specifically an Xeon Phi Processor
7210, sporting 64 cores running at a base frequency of 1.3 GHz with
DDR4-2133 memory.  The operating system is CentOS 7.2 and Intel
Parallel Studio version 17.0.1 was used.

### Code modifications

The code needed to be updated to conform to the PETSC API changes in version xx to be able to run when compiled with the intel compiler and newer versions of PETSC otherwise it will fail. 

The code does PETSC calls from within parallel regions and in an
effort to guarantee correct operations it needs to be built with
thread safety.

### Building thread safe PETSC for KNL

The configure command used to generate the configuration for PETSC on
the KNL architecture was:

```
./configure --with-blas-lapack-dir=/opt/intel/compilers_and_libraries/linux/mkl/lib/intel64 --with-gnu-compilers=0 --with-vendor-compilers=intel --with-mpi-dir=/opt/intel/compilers_and_libraries_2017/linux/mpi/intel64/ --CFLAGS="-O3 -xMIC-AVX512 -qopenmp" --CXXFLAGS="-O3 -xMIC-AVX512 -qopenmp" --FFLAGS="-O3 -xMIC-AVX512 -qopenmp" --with-debugging=0 --with-openmp=1 --with-threadsafety=1 --with-log=0
```

## Performance

### Test cases 

<!--- Ask Ronan for a quick description of the test case --->

Small test case allegedly has x.x million particles

### Compiler options and vectorization 

To begin with we tested various compiler flags and the different impact they had on the performance, first
different levels of optimization. There are some no real performance gains to
be had when going from the default -O2 to -O3.

There is no explicit vectorization in the code, nor is it designed in
a way that favors the compiler vectorizing the code. There does appear
to be some parts of the code that benefits from vectorization to some
small degree. Switching on the AVX512 vectotization in the compiler reduces the time taken to complete one iteration by 4.3%. 

The final compile flags we used where:

```
-O3  -xMIC-AVX512 -qopenmp 
```

### Memory allocators

Tbbmalloc was enabled through the use of LD_PRELOAD

```
export LD_PRELOAD=$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc_proxy.so.2:$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc.so.2
```

To enable TBBMalloc to utilize huge pages, set 

```
export TBB_MALLOC_USE_HUGE_PAGES=1
```

There are some minor performance gains of less than 1% of reduction in the time for one time steps to be had from utilizing TBBMalloc as opposed to the default system memory allocator. The total runtime of the program was reduces with roughly 1% when switching to using TBBMalloc.

### Optimal run parameters

Since the original design relies on MPI for the parallelization and
with the relative immaturity of the OpenMP implementation the optimal
run parameters right now favor using a larger amount of MPI tasks.

The small test case is created with the intent of running around 32
MPI tasks and still being able to fit into the memory space of the KNL
system used. We were able to use the case for testing up to 128 MPI
ranks, 256 ranks did not fit into the memory of the system.

The memory usage, as reported by the program itself for 32-128 MPI ranks was between 400
and 500 MB per MPI rank for all the cases tested. For the case with 16 MPI ranks the memory usage was between 650 and 759 MB Meaning there is a significantly lower overall memory usage for the runs using fewer MPI
ranks. 

The performance observed was measured from the time an iteration which
did not output the results took, since the outputting of data currently involves a significant amount of critical sections the, the time is measured in seconds:

| MPI Ranks | 16 threads | 32 threads | 64 threads | 128 thread | 256 threads |
|:---------:|------------|------------|------------|------------|-------------|
| 16        | 686        | 377        | 242        | 191        | 233         |
| 32        | -          | 349        | 212        | 157        | 156         |
| 64        | -          | -          | 179        | 140        | 139         |
| 128       | -          | -          | -          | 136        | 133         |

The benefit from using all available 256 threads, regardless of the
number of ranks used, is very small. In the 16 rank case there is also a significant reduction in the performance when going to 256 threads, this is likely due to how the OpenMP is currently implemented in the code.

### MCDRAM utilization

For all tests the MCDRAM was in cache mode.

### Performance comparison

To compare the performance the same test case was also run on a single
compute node consisting of two twelve core Intel Haswell E5-2690v3
processors, running at 2,6GHz.

The test where conducted with both a pure MPI run as well as only
using 8 MPI ranks and 3 OpneMP threads per rank, the run time was
measured for an iteration that did not output the results. The pure
MPI case took 93 seconds and the hybrid case took 109 seconds, both
faster than the fastest run on the KNL system.

## Further improvements

First and foremost we need to make sure the OpenMP variant of the code
will actually give correct results, there are a few sections that
currently cast doubt on the correctness of the results from the OpenMP
variant of the code.

The random number generation in the OpenMP also seems flawed since it
ends up reusing the random numbers generated in certain cases.

The OpenMP variant of the code relies heavily on critical sections and
atomic operations to do what is essentially reduction operations. At
least the scalar values operated on in this fashion should all be
replaced with reductions of those variables.

In particular the section handling the periodic output of data
contains some large critical sections with a lot of computation being
done in them and not just reading and writing values. With these
sections being called every 10 iterations they do have a significant
impact on the performance.

Schedule of some loops need to be changed as the load within them is
unbalanced and will cause some threads to sit idle for most of the
execution part of those loops.

Currently the code does not autovectorize well, we should explore the possibility to transform some parts of the code into being more friendly for compiler auto vectorization. With the complexity of the code rewriting it to utilize intrinsic vectorization would be a difficult thing to do.

One potentially big issue in converting the code to more OpenMP
focused is the usage of PETSC. Since PETSC relies on MPI only to
parallelize the work it does, eventually the serial computation caused
by it will start to dominate the runtime of the entire simulation.

## Conclusions

One of the long standing issues with the elmfire code has been the
memory usage of the code. There are some data structures that need to
be duplicated on every rank in the simulation, leading to a large per
compute node memory usage in the cases with compute nodes with many
cores. The OpenMP parallelization mitigates the memory usage issues by
not having to have the duplicate data structures for all cores in a
node but just for each rank on that node. The savings in memory usage
are substantial enough to warrant the usage of the OpenMP version and
even with the slight loss in performance still be a worthwhile effort
to peruse.
