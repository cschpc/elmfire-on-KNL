# Elmfire IPCC level 1 

## Elmfire intro (RR/FR(will fix the computer sciency stuff))
<!-- what it simulates -->
<!-- how it is implemented -->
<!-- I.e it’s a pure MPI fortran code, how it is distributed etc. -->
## Test cases (RR)
<!-- what test cases we used and how they compare to the real simulations -->
## Test systems (FR)
<!-- what systems we used for testing and their setup -->
Basic testing and development work was carried out on single a ninja development workstation, the machine contains one xxxx Intel Xeon Phi many core processor with 16GB of MCDRAM and xxx GB of DDR4 memory.

Final performance measurements were carried out on the Marconi at Cineca in Italy. The Xeon Phi partition of the Marconi cluster consists of 3600 compute nodes, each node contains one 68-core Intel Xeon Phi 7250 CPU clocked at 1.40 GHz, with 16 GB of MCDRAM and 96 GB of DDR4 memory. The interconnect between the compute nodes is based on 100 Gb/s Intel Omni path. 

For all testing the MCDRAM memory was used in cache mode.

## Optimization  (FR/(RR fills in specific details))
The original implementation requires "the field data" to be replicated on all active MPI ranks, which increases the memory requirement for the solver as the resolution of the simulation is increased. The memory requirement for the solver significantly restricts the resolution that can be use for the simulations. As such the goal with the effort was not to improve the performance but rather to attempt to reduce the memory usage to enable the code to efficiently utilize the intel many core architecture.

### OpenMP work
Transitioning the code to a shared memory programming model has the possibility to reduce the memory usage, the idea would be to move the code from one MPI rank per core to a few MPI ranks per node and then be able to use OpenMP to further parallelize the code within the node to utilize all the cores within the node. Since that would reduce the number of MPI ranks we need to utilize all cores within a node it would also reduce the number of copies of the "field data" we need to maintain. 

<!-- Correctness -->
The MPI program relies heavily on variables placed in the global or module namespaces. In order to correctly parallelize the code using OpenMP all of these variables had to be identified and declared as being thread private, otherwise the correctness of the output could not be guaranteed. To correctly identify the variables that needed to be shared we used the Intel Inspector memory and thread debugger. 

<!-- relies on reductions and atomic operations -->
The code also does a significant amount of reduction type operations, either values that are reduced into a single variable or values in an array that get modified by multiple threads. When transitioning the code to a more thread parallel programming model, these operations cause significant race conditions that compromise the validity of the results. OpenMP parallel regions does support reduction operations for both scalar and array type variables, using this OpenMP will create a private copy of the variables for each thread and then at the end of the region the local copies will be reduced into the original variable. The issue with this approach is that the reductions of arrays requires that each thread allocates its own copy of the array leading to a significant increase in the memory usage and the requirement to set a large OpenMP stack size for each thread. 

The main parts of the code doing reductions to arrays is the I/O and the part of the code that updates each particles effect on the field. The I/O functions are only called at specific intervals and as such are not as performance critical, in this case we solved the reductions into arrays using OpenMP atomic statements as not to increase the need for a large OpenMP stack size. The part that "updates the effect the particles have on the field updates not only the cell which the current particle resides in but also adjacent cells", when doing this in a shared memory parallel case multiple threads can easily end up wanting to update the same cells at the same time. Since we also determined this function is one of the ones where the majority of the time is spent we cannot solve these race conditions by applying atomic operations to these update since the performance penalty would be to great. In this case we need to use OpenMP reduction cause, even though it comes with an increase in the memory usage.

<!-- scheduling -->
The load distribution between the iterations of certain loops is also a thing worth exameing in this code as we found a few loops that hadan uneven load distribution. The main target here was the loop responsible for computing the interaction between particles. In this loop there is significantly more work that needs to be done in the begining of it than in the end, leading to the first threads having to do more work and the last ones sitting idle most of the time in the case where the default static scheduling is used, so for this loop dynamic scheduling was applied.

<!-- random number generation -->
Parts of the code relies on random number generation, which in the original version was not generated in a way that would have been safe to be called from multiple threads. We rewrote the random number generation to be safe to be called from multiple threads. This involved giving each thread its own random number generation and seeding these with a seed that was unique to that thread. 

<!-- reporducability of the results, do we need to mention anything about this ? basically the results change based on the number of OMP threads, but it did the same wiht MPI -->

<!-- performance and memory usage -->


### Particle order in memory

Examining the code using the VTune profiling tool we observed that a significant amout of time was spent in the function "writing the effects of the particles on the field" specificly the part of the code where the values are written out to memory. For each particle there is rougly XX values that need to be written, the values update are at least partially adjecent to eachother, but the order the particles are updated in is fairly random since they move significantly each time step. Since the order of the particles are random it will cause the program to jump around in memory when writing the data associated witht the particles, causing a significant amount of cache misses since the prefetcher cannot predic where the next memory access will go.
<!-- Why -->



<!-- what we did -->
If we simply reorder the particles based on their position we can easily improve the performance of the writes. This has to be done every iteration...

<!-- performance -->


### Binning particles to remove the need for OpenMP reductions
<!-- pseudo code on how this was done -->
<!-- why did this work -->

## Results (FR)
<!-- original vs improved code -->
<!-- comment about what can now be simulated -->
## Further work (FR)

<!-- IO -->
<!-- account for angle when binning -->
Currently we are only accounting for the distance from the center when binning the particles for updating the field. While this increased the performance and allowed us to do away with the memroy intensive OpenMP reduction it does come with some limitations. Firstly there is a limit on how many bins we can create this way, which will put a hard limit on the number of threads the operation can be distributed over, for the cases we tested we get xx bins, which would put the hard limit on threads at that number. Ideally we would like to have more bins than we have threads to enable us to balance the load between the threads in since some bins will include significantly more work than others. One wau to acomplish this would be to alos account for the 



## Conclusion (FR)
<!-- what did we learn -->
<!-- make a point about the modifications being in the master branch and ready to use -->