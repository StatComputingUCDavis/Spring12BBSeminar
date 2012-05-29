


```r
library(gputools)
A <- matrix(rnorm(2000 * 30000), 2000, 30000)
B <- matrix(rnorm(30000 * 4000), 30000, 4000)
system.time(A %*% B)
```



```
  user  system elapsed 
  86.900   0.210   8.112 
```



```r
system.time(gpuMatMult(A, B))
```



```
    user  system elapsed 
    7.46    4.05   11.59 
```




