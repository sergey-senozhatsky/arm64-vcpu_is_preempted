## ARM64 vcpu_is_preempted()

Test environment:    
-- 4 CPU arm64 board (rpi4, 4G)    
-- 4 VCPU qemu-kvm (very minimalistic openbox based initramfs)    
-- Host is idle during the test    
-- 5.13 kernel

Executed tests:    
-- schbench -t 3 -m 3 -p 4096    
-- perf bench sched all    

perf bench seems to start a lot of processes (400), which probably does not exactly match our potential use cases. Yet still it shows quite a bit of improvement.

##### schbench
##### Results
```
Disabled vcpu_is_preempted()	                      |  Enabled vcpu_is_preemption()
        50.0th: 1 (3556427 samples)                   |          50.0th: 1 (3507010 samples)
        75.0th: 13 (879210 samples)                   |          75.0th: 13 (1635775 samples)
        90.0th: 15 (893311 samples)                   |          90.0th: 16 (901271 samples)
        95.0th: 18 (159594 samples)                   |          95.0th: 24 (281051 samples)
        *99.0th: 118 (224187 samples)                 |          *99.0th: 114 (255581 samples)
        99.5th: 691 (28555 samples)                   |          99.5th: 382 (33051 samples)
        99.9th: 7384 (23076 samples)                  |          99.9th: 6392 (26592 samples)
        min=1, max=104218                             |          min=1, max=83877
avg worker transfer: 25192.00 ops/sec 98.41MB/s       |  avg worker transfer: 28613.39 ops/sec 111.77MB/s       
```

##### perf bench sched
 
perf bench sched consists of two tests:
```
   Running sched/messaging benchmark...
   20 sender and receiver processes per group
   10 groups == 400 processes run
```
and
```
   Running sched/pipe benchmark...
   Executed 1000000 pipe operations between two processes
```
###### Results
```
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
	                                                    |     
     Total time: 3.366 [sec]                                |          Total time: 2.435 [sec]
	                                                    |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
	                                                    |     
     Total time: 29.893 [sec]                               |          Total time: 27.408 [sec]
	                                                    |     
      29.893106 usecs/op                                    |           27.408089 usecs/op                                     
          33452 ops/sec                                     |               36485 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.086 [sec]                                |          Total time: 2.716 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.813 [sec]                               |          Total time: 25.377 [sec]
                                                            |     
      29.813438 usecs/op                                    |           25.377074 usecs/op                                     
          33541 ops/sec                                     |               39405 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.328 [sec]                                |          Total time: 2.044 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.971 [sec]                               |          Total time: 27.130 [sec]
                                                            |     
      29.971262 usecs/op                                    |           27.130576 usecs/op                                     
          33365 ops/sec                                     |               36858 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.068 [sec]                                |          Total time: 2.544 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.890 [sec]                               |          Total time: 26.282 [sec]
                                                            |     
      29.890193 usecs/op                                    |           26.282635 usecs/op                                     
          33455 ops/sec                                     |               38047 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.397 [sec]                                |          Total time: 1.923 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.895 [sec]                               |          Total time: 26.408 [sec]
                                                            |     
      29.895972 usecs/op                                    |           26.408488 usecs/op                                     
          33449 ops/sec                                     |               37866 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 1.723 [sec]                                |          Total time: 2.094 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.747 [sec]                               |          Total time: 28.634 [sec]
                                                            |     
      29.747595 usecs/op                                    |           28.634530 usecs/op                                     
          33616 ops/sec                                     |               34922 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.464 [sec]                                |          Total time: 2.364 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.868 [sec]                               |          Total time: 29.079 [sec]
                                                            |     
      29.868606 usecs/op                                    |           29.079261 usecs/op                                     
          33479 ops/sec                                     |               34388 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 2.075 [sec]                                |          Total time: 2.620 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.766 [sec]                               |          Total time: 26.879 [sec]
                                                            |     
      29.766806 usecs/op                                    |           26.879253 usecs/op                                     
          33594 ops/sec                                     |               37203 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 1.671 [sec]                                |          Total time: 1.726 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.887 [sec]                               |          Total time: 28.277 [sec]
                                                            |     
      29.887707 usecs/op                                    |           28.277614 usecs/op                                     
          33458 ops/sec                                     |               35363 ops/sec                                        
                                                            |     
# Running sched/messaging benchmark...	                    |     # Running sched/messaging benchmark...
# 20 sender and receiver processes per group	            |     # 20 sender and receiver processes per group
# 10 groups == 400 processes run	                    |     # 10 groups == 400 processes run
                                                            |     
     Total time: 1.627 [sec]                                |          Total time: 2.235 [sec]
                                                            |     
# Running sched/pipe benchmark...	                    |     # Running sched/pipe benchmark...
# Executed 1000000 pipe operations between two processes    |     # Executed 1000000 pipe operations between two processes
                                                            |     
     Total time: 29.669 [sec]                               |          Total time: 28.425 [sec]
                                                            |     
      29.669366 usecs/op                                    |           28.425186 usecs/op                                     
          33704 ops/sec                                     |               35180 ops/sec                                        
```

###### Student’s T-test (ops/sec) perf bench:
  
```
x base
+ patched
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|x xx x x  x                  +               +      +    +                               +          +         +                  +    +                                      +|
| |_MA__|                                      |_____________________________________________A_______M_____________________________________|                                   |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    N           Min           Max        Median           Avg        Stddev
x  10         33365         33704         33479       33511.3     100.92467
+  10         34388         39405         36858       36571.7      1607.454
Difference at 95.0% confidence
	3060.4 +/- 1070.09
	9.13244% +/- 3.19321%
	(Student's t, pooled s = 1138.88)
```
