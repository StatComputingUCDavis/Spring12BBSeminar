# Parallelization on High-Performance Clusters in R


# Parallel high performance computing environments for R

This is a minimal tutorial on parallel computing environments for R. While an array of packages (and a lot of native R since the 2.14.0 release) supports multi-core parallelization, allowing R to take advantage of the multiple processors found on a single chip that are now standard in laptop and desktop machines, this focus here is on running R in larger clusters. 

Large clusters connect many "compute nodes" together in a way that allows them to share access to data through the hard disk, but each node has it's own processor and it's own memory.  The challenge comes entirely from this latter situation -- unlike the multicore chips on laptops and desktops, the different processors each look at their own memory sitting next to them -- they cannot see the memory (or RAM) on another node.  This requires data to be passed back and forth between the nodes explicitly. The Message Passing Interface (MPI) has been the standard protocol to do this on large supercomputers, and will be our focus here.  You'll notice that most of these commands directly deal with this challenge of passing data to the different compute nodes.  

Note that the cluster or supercomputer architecture is designed essentially with this in mind.  All the nodes can still access the same harddisk, and the approach still requires the nodes in the cluster to all be directly connected.  This is distinct from more distributed computing or cloud computing architectures where the nodes are not as tightly coupled, and passing data between nodes becomes even more challenging.  Also note that this approach still assumes data can be loaded into memory.  While that might be several gigabytes per node on most compute clusters, due to the rapid increase in the number of cores or processors we can put on a single chip, the amount of memory per processor is actually been flat or decreasing. GPU architectures are a prime example -- providing 100s of processors on a single node, but very little memory per processor.  This trend further exacerbates the basic problem of big data.  

## Onto the Code

We will ignore these issues for the time being and simply introduce the syntax for running a command across a series of processors in these high-performance computing environments. The code is usually submitted to a cluster using a queue, which allows multiple users to efficiently share access to the computational resource.  The syntax of these queues differs with different software and hardware.  In this example, I use [this script](https://github.com/cboettig/sandbox/blob/master/mpi.sh) to request my run on the cluster.  The job is then submitted to the cluster using the command 

```
qsub mpi.sh
```
## RMPI

The direct Rmpi way:



```r
library(Rmpi)
mpi.spawn.Rslaves(nslaves = 3)
```



```
## 	3 slaves are spawned successfully. 0 failed.
## master (rank 0, comm 1) of size 4 is running on: c0-6 
## slave1 (rank 1, comm 1) of size 4 is running on: c0-6 
## slave2 (rank 2, comm 1) of size 4 is running on: c0-6 
## slave3 (rank 3, comm 1) of size 4 is running on: c0-6 
```



```r
slavefn <- function() {
    print(paste("Hello from", foldNumber))
}
mpi.bcast.cmd(foldNumber <- mpi.comm.rank())
mpi.bcast.Robj2slave(slavefn)
result <- mpi.remote.exec(slavefn())
print(result)
```



```
## $slave1
## [1] "Hello from 1"
## 
## $slave2
## [1] "Hello from 2"
## 
## $slave3
## [1] "Hello from 3"
## 
```



```r
mpi.close.Rslaves()
```



```
## [1] 1
```




# Benchmark



```r
A <- matrix(rnorm(1e+06), 1000)
system.time(A %*% A)
```



```
##    user  system elapsed 
##   1.475   0.004   1.480 
```






## SNOW



```r
library(snow)
cluster <- makeCluster(4, type = "MPI")
```



```
## 	4 slaves are spawned successfully. 0 failed.
```



```r
clusterEvalQ(cluster, library(utils))  # load a library
```



```
## [[1]]
## [1] "snow"      "Rmpi"      "methods"   "stats"     "graphics"  "grDevices"
## [7] "utils"     "datasets"  "base"     
## 
## [[2]]
## [1] "snow"      "Rmpi"      "methods"   "stats"     "graphics"  "grDevices"
## [7] "utils"     "datasets"  "base"     
## 
## [[3]]
## [1] "snow"      "Rmpi"      "methods"   "stats"     "graphics"  "grDevices"
## [7] "utils"     "datasets"  "base"     
## 
## [[4]]
## [1] "snow"      "Rmpi"      "methods"   "stats"     "graphics"  "grDevices"
## [7] "utils"     "datasets"  "base"     
## 
```



```r
clusterExport(cluster, ls())  # export everything
out <- parSapply(cluster, 1:4, function(x) print(paste("snow hello from ", 
    x)))
print(out)
```



```
## [1] "snow hello from  1" "snow hello from  2" "snow hello from  3"
## [4] "snow hello from  4"
```



```r
system.time(parMM(cluster, A, A))
```



```
##    user  system elapsed 
##   0.741   0.310   1.051 
```



```r
stopCluster(cluster)
```



```
## [1] 1
```




## SNOWFALL 
(default "SOCK" type, for multicore machines).



```r
library(snowfall)
sfInit(parallel = TRUE, cpus = 4)
```



```
## R Version:  R version 2.15.0 (2012-03-30) 
## 
```



```
## snowfall 1.84 initialized (using snow 0.3-8): parallel execution on 4 CPUs.
## 
```



```r
sfExportAll()
sfLibrary(utils)
```



```
## Library utils loaded.
```



```
## Library utils loaded in cluster.
## 
```



```
## Warning message: 'keep.source' is deprecated and will be ignored
```



```r
out <- sfSapply(1:4, function(x) print(paste("snow hello from ", 
    x)))
print(out)
```



```
## [1] "snow hello from  1" "snow hello from  2" "snow hello from  3"
## [4] "snow hello from  4"
```



```r
system.time(sfMM(A, A))
```



```
##    user  system elapsed 
##   0.277   0.030   0.733 
```



```r
sfStop()
```



```
## 
## Stopping cluster
## 
```




Snowfall using MPI mode, for distributing across nodes in a cluster (that use a shared hard disk but don't share memory).



```r
library(snowfall)
sfInit(parallel = TRUE, cpus = 4, type = "MPI")
sfExportAll()
sfLibrary(utils)
out <- sfSapply(1:4, function(x) print(paste("snow hello from ", 
    x)))
print(out)
system.time(sfMM(A, A))
sfStop()
```



For reasons unknown to me, this last command does not work on farm, though it works fine on NERSC cluster.  

snow's close command, which shuts down and quits from script.   



```r
mpi.quit(save = "no")
```



