# Elmfire IPCC level 1 

## Elmfire intro (RR/FR(will fix the computer sciency stuff))
<!-- what it simulates -->
<!-- how it is implemented -->
<!-- I.e it’s a pure MPI fortran code, how it is distributed etc. -->
## Test cases (RR)
<!-- what test cases we used and how they compare to the real simulations -->
## Test systems (FR)
<!-- what systems we used for testing and their setup -->
Basic testing and development work was carried out on single a ninja development workstation, the machine containes one xxxx Intel Xeon Phi many core processor with 16GB of MCDRAM and xxx GB of DDR4 memory.

Final performance measurements were carried out on the Marconi at Cineca in Italy. The Xeon Phi partition of the Marconi cluster consists of 3600 compute nodes, each node contains one 68-core Intel Xeon Phi 7250 CPU clocked at 1.40 GHz, with 16 GB of MCDRAM and 96 GB of DDR4 memory. The interconnect between the compute nodes is based on 100 Gb/s Intel Omni path. 

For all testings the MCDRAM memory was used in cache mode.

## Optimization  (FR/(RR fills in specific details))
The original implementation requieres "the field data" to be replicated on all active MPI ranks, which increases the memory requirement for the solver as the resolution of the simulation is increased. The memory requirement for the solver singificanlty restricts the resolution that can be use for the simulations. As such the goal with the effort was not to improve the performance but rather to attempt to reduce the memory usage to enable the code to efficiently utilize the intel many core architecture.

### OpenMP work
Transitioning the code to a shared memory progamming model has the posibility to reduce the memory uasge, the idea would be to move the code from one MPI rank per core to a few MPI ranks per node and then be able to use OpenMP to further parallelize the code within the node to utilize all the cores within the node. Since that would reduce the number of MPI ranks we need to utilize all cores within a node it would also reduce the number of copies of the "field data" we need to maintain. 

<!-- Correctness -->
The MPI program relies heavily on variables placed in the global or module namespaces. In order to correctly paralleize the code using OpenMP all of these variables had to be identified and delcared as beign thread private, othwerwise the corectness of the output could not be guaraneteed. To correctly identify the variables that needed to be shared we used the Intel Inspector memory and thread debugger. 

<!-- relies on reductions and atomic operations -->
The code also does a significant amout of reduction type operations, either values that are reduced into a single variable or values in an array that get modified by multiple threads. When transitioning the code to a more thread parallel programming model, these operations cause significant race conditons that compromize the validity of the results. OpenMP parallel regions does support reduction operations for both scalar and array type variables, using this OpenMP will create a private copy of the variables for each thread and then at the end of the region the local copies will be reduced into the origial variable. The issue with this aproach is that the reductions of arrays requiers that each thread allocates its own copy of the array leading to a significant increase in the memory usage and the requirement to set a large OpenMP stack size for each thread. // moar

<!-- scheduling -->


<!-- random number generation -->
Parts of the code relies on random number generation, which in the origial version was not generated in a way that would have been safe to be called from multiple threads. We rewrote the random number generation to be safe to be called from multiple threads. This involved giving each thread its own random number generation and seeding these with a seed that was unique to that thread. 

<!-- reporducability of the results -->
With 

<!-- performance and memory usage -->

### Particle order in memory
<!-- Why -->
<!-- what we did -->
<!-- performance -->
## Results (FR)
<!-- original vs improved code -->
<!-- comment about what can now be simulated -->
## Further work (FR)

<!-- IO -->
<!-- account for angle when binning -->
## Conclusion (FR)
<!-- what did we learn -->
<!-- make a point about the modifications being in the master branch and ready to use -->
