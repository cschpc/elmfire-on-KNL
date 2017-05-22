#Elmfire IPCC level 1 

##Elmfire introm (RR/FR(will fix the computer sciency stuff))
<!-- what it simulates -->
<!-- how it is implemented -->
<!-- I.e it’s a pure MPI fortran code, how it is distributed etc. -->
##Test cases (RR)
<!-- what test cases we used and how they compare to the real simulations -->
##Test systems (FR)
<!-- what systems we used for testing and their setup -->
##OpenMP work (FR/RR)
The original implementation requieres the xx to be replicated on all active MPI ranks, which increases the memory requirement for the solver as the resolution of the simulation is increased. 
<!-- Correctness -->
The MPI program relies heavily on global variables //module variables. In order to correctly paralleize the code using OpenMP all of these variables had to be identified and delcared as beign thread private, othwerwise the corectness of the output could not be guaraneteed. To correctly identify the variables that needed to be shared we used the Intel Inspector memory and thread debugger. 
<!-- relies on reductions and atomic operations -->
The code also does 
<!-- performance and memory usage -->
##Particle order in memory (FR/RR)
<!-- Why -->
<!-- what we did -->
<!-- performance -->
##Scaling (FR)
<!-- original vs improved code -->
<!-- comment about what can now be simulated -->
##Further work (FR)

<!-- IO -->
<!-- account for angle when binning -->
##Conclusion (FR)
<!-- what did we learn -->
<!-- make a point about the modifications being in the master branch and ready to use -->
