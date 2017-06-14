# Elmfire IPCC OpenMP effort

## Elmfire intro
Elmfire is implemented in Fortran 77/90. It started its life as a Monte Carlo code and was slowly updated to PIC and full-f gyro kinetics over the last 15 years. The code relies on MPI to parallelize the grid in 1D and distribute the solver over multiple cores and compute nodes, using PetSc an external library for matrix work. 

Elmfire is a Particle In Cell (PIC) code with drift kinetic electrons, meaning that it allows more complex physics at smaller scales than other competing codes. While this addition provides new possibilities it also requires more memory and a smaller time step, making it highly computationally intensive. 

## Test cases
The test cases provided have been optimized to study both grid and particle memory data. Case 1 was a small grid with average particle count, Case 2 a medium grid and again average particle count and Case 3 a large grid with a higher particle count. These have been used to push the limits of the OpenMP paradigm. 

## Test systems
<!-- what systems we used for testing and their setup -->
Basic testing and development work was carried out on single a ninja development workstation, the machine contains one 7210 Intel Xeon Phi many core processor, clocked at 1.3 GHz with 16GB of MCDRAM and 96 GB of DDR4 memory.

Multi node measurements were carried out on the Marconi at Cineca in Italy. The Xeon Phi partition of the Marconi cluster consists of 3600 compute nodes, each node contains one 68-core Intel Xeon Phi 7250 CPU clocked at 1.40 GHz, with 16 GB of MCDRAM and 96 GB of DDR4 memory. The interconnect between the compute nodes is based on 100 Gb/s Intel Omni path. 

For all testing the MCDRAM memory was used in cache mode.

## Optimization
The original implementation requires some elements of the simulation to be replicated on all active MPI ranks, which increases the memory requirement for the solver as the resolution of the simulation is increased. The memory requirement for the solver significantly restricts the resolution that can be use for the simulations. As such the primary goal with the effort was not to improve the performance but rather to attempt to reduce the memory usage to enable the code to efficiently utilize the intel many core architecture.

### OpenMP work
Transitioning the code to a shared memory programming model has the possibility to reduce the memory usage, the idea would be to move the code from one MPI rank per core to a few MPI ranks per node and then be able to use OpenMP to further parallelize the code within the node to utilize all the cores within the node. Since that would reduce the number of MPI ranks we need to utilize all cores within a node it would also reduce the number of copies of the "field data" we need to maintain. 

<!-- Correctness -->
The MPI program relies heavily on variables placed in the global or module namespaces. In order to correctly parallelize the code using OpenMP all of these variables had to be identified and declared as being thread private, otherwise the correctness of the output could not be guaranteed. To correctly identify the variables that needed to be shared we used the Intel Inspector memory and thread debugger. 

<!-- relies on reductions and atomic operations -->
The code also does a large number of reduction type operations, either values that are reduced into a single variable or values in an array that get modified by multiple threads. When transitioning the code to a more thread parallel programming model, these operations cause serious race conditions that compromise the validity of the results. OpenMP parallel regions does support reduction operations for both scalar and array type variables, using this OpenMP will create a private copy of the variables for each thread and then at the end of the region the local copies will be reduced into the original variable. The issue with this approach is that the reductions of arrays requires that each thread allocates its own copy of the array leading to a increase in the memory usage and the requirement to set a large OpenMP stack size for each thread. 

The main parts of the code doing reductions to arrays is the I/O and the part of the code that updates each particles effect on the field. The I/O functions are only called at specific intervals and as such are not as performance critical, in this case we solved the reductions into arrays using OpenMP atomic statements as not to increase the need for a large OpenMP stack size. The part that updates the effect the particles have on the field updates not only the cell which the current particle resides in but also adjacent cells, when doing this in a shared memory parallel case multiple threads can easily end up wanting to update the same cells at the same time. Since we also determined this function is one of the ones where the majority of the time is spent we cannot solve these race conditions by applying atomic operations to these update since the performance penalty would be to great. In this case we need to use OpenMP reduction cause, even though it comes with an increase in the memory usage.

<!-- scheduling -->
The load distribution between the iterations of certain loops is also a thing worth examining, in this code as we found one loops that had an uneven load distribution, the loop responsible for computing the interaction between particles. In this loop there is significantly more work that needs to be done in the beginning of it than in the end, leading to the first threads having to do more work and the last ones sitting idle most of the time in the case where the default static scheduling is used. Looking at the profiler results this becomes fairly obvious, so for this loop dynamic scheduling is a better option.

<!-- random number generation -->
Parts of the code relies on random number generation, which in the original version was not generated in a way that would have been safe to be called from multiple threads. We rewrote the random number generation to be safe to be called from multiple threads. This involved giving each thread its own random number generation and seeding these with a seed that was unique to that thread. 

### Particle order in memory

<!-- Why -->
Examining the code using the VTune profiling tool we observed that the majority of the time was spent in the function writing the effects of the particles on the field specifically the part of the code where the values are written out to memory. For each particle there is roughly 100-150 values that need to be written, the values update are at least partially adjacent to each other, but the order the particles are updated in is fairly random since they move large distances each time step. Since the order of the particles are random it will cause the program to jump around in memory when writing the data associated with the particles, causing a large amount of cache misses since the prefetcher cannot predict where the next memory access will go.

<!-- what we did -->
If we simply reorder the particles based on their position we can easily improve the performance of the writes. This has to be done every iteration, meaning we cannot use a complex sorting algorithm since any performance gains from that would immediately be eclipsed by the runtime of the sorting. This was done via a tweaked method based on Bowers, K. J. "Accelerating a particle-in-cell simulation using a hybrid counting sort." Journal of Computational Physics 173.2 (2001): 393-411., as it offers a good balance between runtime and performance increases.

<!-- performance -->
Sorting the particles simply by how far from the center they are we are able to improve the performance of the solver by 6%, the sorting all the particles that are within a specific ring segment of the field next to each other. We should aim to divide the larger ring segments into sectors that would fit into cache memory for potential further speedups.

### Binning particles to remove the need for OpenMP reductions
<!-- why -->
One of the major drawback of our OpenMP version was the reliance on reductions, due to how the code that sets up the particles effect on the field was structured, the array for the field can have multiple threads update the same value at the same time, and the only way to solve that and maintain performance was to mark the entire array as a reduction variable, which increased the memory requirement of the code. 

<!-- why did this work -->
A better solution is to divide particles into separate bins, each bin contains a subset of particles whose writes go to a specific part of the array. We can then for each bin create a thread than handles the particles within that bin, the thread would create its own subset of the array to where the writes would initially go, once the thread has processed all the particles the local chunk of the array would be written out to the global array, taking care only to allow one thread to write to the global array at one time. That way we completely eliminate the need to mark the array as a reduction variable, saving the need for a larger OpenMP stack size and saving memory usage.

Since we already sort the particles based which grid ring they fall into it makes sense to use the same approach to the binning, we created bins based on the grid rings and included a few rings into each bin. As each ring needs a significant amount of ghost data around them making the rings too thin will negatively affect the performance.

<!-- pseudo code on how this was done -->
```
Bin particles based on location
#OMP paralell loop
For bins
    Allocate local array
    For particles
        Write particles effect on field to array
    #OMP atomic
    Write local array to global
```

## Results
<!-- original vs improved code -->

Testing the original MPI only code with the small test case, and the ninja developer platform, we see that we can reduce the memory usage by lowering the number of MPI ranks but at a large reduction in the overall performance of the simulation. The time measured is for the fourth time step and the memory usage is the maximum memory usage as reported by Elmfires internal reporting.

| MPI Ranks | Original   | Original memory |
|:---------:|------------|-----------------|
| 32        | 349 sec    | 24.1 GB         |
| 64        | 179 sec    | 37.0 GB         |
| 128       | 136 sec    | 60.1 GB         |

While the memory usage for the original version can be reduced by simply lowering the number of MPI ranks that is comes at the cost of a significant reduction in the speed of the solver. 

With the OpenMP version the goal is to be able to run the solver using fewer MPI ranks and gain back the performance lost by reducing the number of ranks through OpenMP threading. Running the small test case with the OpenMP enabled solver yields the results shown in the table below.

| MPI ranks | Total thread count | Time (s) | Total memory |
|-----------|--------------------|----------|--------------|
| 16        | 32                 | 352.14   | 18.7234375   |
| 16        | 64                 | 200.2    | 21.309375    |
| 16        | 128                | 160.51   | 26.8625      |
| 16        | 256                | 176.27   | 32.984375    |
||
| 32        | 32                 | 318.09   | 23.371875    |
| 32        | 64                 | 176.48   | 28.428125    |
| 32        | 128                | 134.96   | 32.146875    |
| 32        | 256                | 141.51   | 43.45        |
||
| 64        | 64                 | 178.48   | 35.175       |
| 64        | 128                | 129.48   | 46.69375     |
| 64        | 256                | 131.79   | 53.6875      |
||
| 128       | 128                | 128.45   | 60.2125      |
| 128       | 256                | 138.09   | 84.4375      |

The best performance is achieved when running on 128 threads, the original code took 136 seconds per time step when running with 128 threads and our optimized version also achieves its best performance when running on 128 threads. With the OpenMP version, and the optimizations carried out on the grid update function, we can achieve the same performance we previously did with 128 MPI ranks on 32 MPI ranks and running 4 OpenMP threads per rank. At 32 MPI rank we are able to reduce the memory usage of the program by 47%. If want to sacrifice some performance than can be further reduced by running 16 MPI ranks and 8 OpenMP threads, in that case we have 15% worse performance but 60% less memory required than the original version running on 128 MPI ranks.

For the medium test case running on Marconi ...

<!-- comment about what can now be simulated -->
With the reduction in memory usage the application user are able to scale up the simulations they are running. Previously the memory usage of the code was the main limiting factor on the type of simulation that could be run, the reactor size was limited to roughly 0.069m³. With the OpenMP version the reactor size can be scaled up, to a volume of around 21m³, enabling the team to simulate the Tore Supra reactor. 

## Further work

<!-- IO -->
The I/O functions of the code was largely ignored since it is rarely called and the development team is in the process of redesigning the I/O functionally.

<!-- account for angle when binning -->
Currently we are binning and sorting the particles based on which ring segment of the field they fall into. While this allowed us to do away with the memory intensive OpenMP reduction it does come with some limitations. Firstly there is a limit on how many bins we can create this way, which will put a hard limit on the number of threads the operation can be distributed over. Ideally we would like to have more bins than we have threads to enable us to balance the load between the threads in since some bins will include more work than others. One way to accomplish this would be to also account for the angle each particle sits at when binning, splitting the larger rings into segments. This would increase the number of bins and allow us to have more parallelism for that section and also allow us to better balance the load between threads for that section. Additionally it should also offer us a speedup if we can make the bins small enough to fit into cache, as then we could see significant cache reuse when writing the particles effect on the grid.

## Conclusion

Through the use of OpenMP shared memory parallelism we are able to reduce the amount of memory needed by the Elmfire solver, while still maintaining the performance. The memory reductions enabled the Elmfire team to simulate reactors that they had previosly been unable to. The code modifications are present in the version of the solver they are using to simulate these reactor types.
<!-- what did we learn -->
<!-- make a point about the modifications being in the master branch and ready to use -->



