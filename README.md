## ARM64 vcpu_is_preempted()

Test environment:

-- 4 CPU arm64 board (rpi4, 4G)
-- 4 VCPU qemu-kvm (very minimalistic openbox based initramfs)
-- in-house vcpu_is_preempted() implementation, that just works (no fancy stuff)
-- Host is idle during the test
-- 5.7 kernel (PREEMPT_VOLUNTARY)

Executed tests:
-- schbench -t 3 -m 3 -p 4096
-- perf bench sched all

I deliberately did not run overloaded/overcommitted VM (except for perf bench test), nor did I want to run lock-torturing tests.

The basic idea is to have one or more idle vCPUs (potentially preempted) during the test.

schbench demonstrates that when scheduler knows if the vcpu is preempted it may take better wake up decisions. One particular case when this becomes very visible is the stress-ng aio test - vcpu_is_preempted() boosts the performance of that test quite considerably:
```
x Disabled vcpu_is_preempted()
stress-ng: info:  [100] stressor       bogo ops real time  usr time  sys time   bogo ops/s   bogo ops/s
stress-ng: info:  [100]                           (secs)    (secs)    (secs)   (real time) (usr+sys time)
stress-ng: info:  [100] aio              222927     10.01      0.89     27.61     22262.04      7822.00
stress-ng: info:  [139] aio              217043     10.01      1.00     26.80     21685.46      7807.30
stress-ng: info:  [178] aio              217261     10.01      1.08     26.79     21709.36      7795.51
---
+ Enabled vcpu_is_preempted()
stress-ng: info:  [100] aio              432750     10.00      1.14     19.03     43264.33     21455.13
stress-ng: info:  [139] aio              426771     10.01      1.09     18.67     42629.13     21597.72
stress-ng: info:  [179] aio              533039     10.00      1.42     20.39     53281.70     24440.12
```
A very quick tracing run shows that during the stress-ng aio test (3 concurrent processes on a 4 vCPU vm, 10 seconds run), the scheduler attempts to enqueue a task on idle preempted vCPU 29082 times, and 630183 times on idle available vCPU (https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/core.c?h=v5.8-rc3#n4743). This is definitely highly workload dependent, etc. Sometimes the ratio can be better (less tasks enqueued on preempted vCPU) sometimes worse.

  
perf bench seems to start a lot of processes (400), which probably does not exactly match our potential use cases. Yet still it shows quite a bit of improvement.

##### schbench
##### Results
```
Disabled vcpu_is_preempted()	                        |       	Enabled vcpu_is_preemption()
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 16	                                |       		95.0th: 15
	*99.0th: 23	                                |       		*99.0th: 22
	99.5th: 28	                                |        		99.5th: 26
	99.9th: 49	                                |       		99.9th: 38
	min=0, max=12028	                        |       		min=0, max=11983
avg worker transfer: 44800.57 ops/sec 175.00MB/s	|       	avg worker transfer: 48658.35 ops/sec 190.07MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 15	                                |       		95.0th: 14
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 27	                                |       		99.5th: 23
	99.9th: 45	                                |       		99.9th: 38
	min=0, max=12661	                        |       		min=0, max=9943
avg worker transfer: 43763.87 ops/sec 170.95MB/s	|       	avg worker transfer: 49048.86 ops/sec 191.60MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 10
	95.0th: 15	                                |       		95.0th: 13
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 27	                                |       		99.5th: 23
	99.9th: 48	                                |       		99.9th: 37
	min=0, max=15783	                        |       		min=0, max=11942
avg worker transfer: 46405.34 ops/sec 181.27MB/s	|       	avg worker transfer: 48817.75 ops/sec 190.69MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 8
	95.0th: 15	                                |       		95.0th: 11
	*99.0th: 23	                                |       		*99.0th: 19
	99.5th: 28	                                |       		99.5th: 22
	99.9th: 47	                                |       		99.9th: 37
	min=0, max=8055	                                |       		min=0, max=11790
avg worker transfer: 45256.13 ops/sec 176.78MB/s	|       	avg worker transfer: 51123.28 ops/sec 199.70MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 10
	95.0th: 15	                                |       		95.0th: 13
	*99.0th: 22	                                |       		*99.0th: 20
	99.5th: 27	                                |       		99.5th: 23
	99.9th: 44	                                |       		99.9th: 36
	min=0, max=13367	                        |       		min=0, max=15833
avg worker transfer: 46341.51 ops/sec 181.02MB/s	|       	avg worker transfer: 49675.11 ops/sec 194.04MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 16	                                |       		95.0th: 13
	*99.0th: 24	                                |       		*99.0th: 20
	99.5th: 28	                                |       		99.5th: 23
	99.9th: 48	                                |       		99.9th: 36
	min=0, max=12661	                        |       		min=0, max=9093
avg worker transfer: 46168.65 ops/sec 180.35MB/s	|       	avg worker transfer: 48238.21 ops/sec 188.43MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 11
	95.0th: 15	                                |       		95.0th: 15
	*99.0th: 23	                                |       		*99.0th: 21
	99.5th: 27	                                |       		99.5th: 25
	99.9th: 46	                                |       		99.9th: 38
	min=0, max=12459	                        |       		min=0, max=24429
avg worker transfer: 46382.17 ops/sec 181.18MB/s	|       	avg worker transfer: 48051.97 ops/sec 187.70MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 16	                                |       		95.0th: 14
	*99.0th: 24	                                |       		*99.0th: 20
	99.5th: 29	                                |       		99.5th: 24
	99.9th: 47	                                |       		99.9th: 38
	min=0, max=10409	                        |       		min=0, max=11499
avg worker transfer: 44881.69 ops/sec 175.32MB/s	|       	avg worker transfer: 47815.48 ops/sec 186.78MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 16	                                |       		95.0th: 15
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 28	                                |       		99.5th: 25
	99.9th: 45	                                |       		99.9th: 39
	min=0, max=10438	                        |       		min=0, max=17826
avg worker transfer: 46285.09 ops/sec 180.80MB/s	|       	avg worker transfer: 48455.33 ops/sec 189.28MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 8
	95.0th: 15	                                |       		95.0th: 12
	*99.0th: 21	                                |       		*99.0th: 19
	99.5th: 25	                                |       		99.5th: 22
	99.9th: 42	                                |       		99.9th: 35
	min=0, max=11657	                        |       		min=0, max=10319
avg worker transfer: 44484.32 ops/sec 173.77MB/s	|       	avg worker transfer: 50606.24 ops/sec 197.68MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 8	                                |       		75.0th: 7
	90.0th: 12	                                |       		90.0th: 11
	95.0th: 16	                                |       		95.0th: 14
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 28	                                |       		99.5th: 24
	99.9th: 49	                                |       		99.9th: 38
	min=0, max=13675	                        |       		min=0, max=8355
avg worker transfer: 44980.09 ops/sec 175.70MB/s	|       	avg worker transfer: 48039.78 ops/sec 187.66MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 10
	95.0th: 16	                                |       		95.0th: 13
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 29	                                |       		99.5th: 24
	99.9th: 51	                                |       		99.9th: 38
	min=0, max=9663	                                |       		min=0, max=9687
avg worker transfer: 46749.12 ops/sec 182.61MB/s	|       	avg worker transfer: 49497.56 ops/sec 193.35MB/s
Latency percentiles (usec)	                        |       	Latency percentiles (usec)
	50.0th: 4	                                |       		50.0th: 4
	75.0th: 7	                                |       		75.0th: 7
	90.0th: 11	                                |       		90.0th: 11
	95.0th: 15	                                |       		95.0th: 14
	*99.0th: 23	                                |       		*99.0th: 20
	99.5th: 27	                                |       		99.5th: 23
	99.9th: 47	                                |       		99.9th: 35
	min=0, max=12091	                        |       		min=0, max=9997
avg worker transfer: 46724.76 ops/sec 182.52MB/s	|       	avg worker transfer: 48315.82 ops/sec 188.73MB/s

```

###### Student's T-test (ops/sec) schbench:

```
x vcpu_is_preempted() disabled
+ vcpu_is_preempted() enabled
+--------------------------------------------------------------------------+
|x      x  xxx  x        xxx  xx         + ++++ + ++ +    + +        +    +|
|         |_________A____M___|            |_______M_A__________|           |
+--------------------------------------------------------------------------+
    N           Min           Max        Median           Avg        Stddev
x  13      43763.87      46749.12      46168.65     45632.562     976.74614
+  13      47815.48      51123.28      48658.35     48949.518     1019.7968
Difference at 95.0% confidence
	3316.96 +/- 808.356
	7.26884% +/- 1.77145%
	(Student's t, pooled s = 998.503)
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
Disabled vcpu_is_preempted()	        |       	Enabled vcpu_is_preempted()
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.957 [sec]	        |       	     Total time: 1.002 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.698 [sec]	        |       	     Total time: 18.264 [sec]
      23.698671 usecs/op	        |       	      18.264774 usecs/op
          42196 ops/sec	                |       	          54750 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.981 [sec]	        |       	     Total time: 0.983 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.513 [sec]	        |       	     Total time: 23.981 [sec]
      23.513668 usecs/op	        |       	      23.981687 usecs/op
          42528 ops/sec	                |       	          41698 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 1.003 [sec]	        |       	     Total time: 0.984 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.685 [sec]	        |       	     Total time: 22.672 [sec]
      23.685026 usecs/op	        |       	      22.672092 usecs/op
          42220 ops/sec	                |       	          44107 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.970 [sec]	        |       	     Total time: 0.961 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.515 [sec]	        |       	     Total time: 19.782 [sec]
      23.515603 usecs/op	        |       	      19.782463 usecs/op
          42524 ops/sec	                |       	          50549 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.986 [sec]	        |       	     Total time: 0.984 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.588 [sec]	        |       	     Total time: 22.970 [sec]
      23.588656 usecs/op	        |       	      22.970971 usecs/op
          42393 ops/sec	                |       	          43533 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.970 [sec]	        |       	     Total time: 0.984 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.634 [sec]	        |       	     Total time: 18.061 [sec]
      23.634909 usecs/op	        |       	      18.061399 usecs/op
          42310 ops/sec	                |       	          55366 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.993 [sec]	        |       	     Total time: 0.988 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.645 [sec]	        |       	     Total time: 18.690 [sec]
      23.645483 usecs/op	        |       	      18.690546 usecs/op
          42291 ops/sec	                |       	          53502 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.989 [sec]	        |       	     Total time: 1.001 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.627 [sec]	        |       	     Total time: 18.937 [sec]
      23.627311 usecs/op	        |       	      18.937338 usecs/op
          42323 ops/sec	                |       	          52805 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.982 [sec]	        |       	     Total time: 0.992 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.659 [sec]	        |       	     Total time: 22.789 [sec]
      23.659978 usecs/op	        |       	      22.789283 usecs/op
          42265 ops/sec	                |       	          43880 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.992 [sec]	        |       	     Total time: 0.985 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.554 [sec]	        |       	     Total time: 23.912 [sec]
      23.554444 usecs/op	        |       	      23.912251 usecs/op
          42454 ops/sec	                |       	          41819 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.998 [sec]	        |       	     Total time: 0.989 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.625 [sec]	        |       	     Total time: 23.950 [sec]
      23.625565 usecs/op	        |       	      23.950785 usecs/op
          42327 ops/sec	                |       	          41752 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.984 [sec]	        |       	     Total time: 0.979 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.391 [sec]	        |       	     Total time: 17.404 [sec]
      23.391019 usecs/op	        |       	      17.404603 usecs/op
          42751 ops/sec	                |       	          57456 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.976 [sec]	        |       	     Total time: 0.973 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.611 [sec]	        |       	     Total time: 18.676 [sec]
      23.611126 usecs/op	        |       	      18.676340 usecs/op
          42352 ops/sec	                |       	          53543 ops/sec
# Running sched/messaging benchmark...  |       	# Running sched/messaging benchmark...
     Total time: 0.984 [sec]	        |       	     Total time: 0.989 [sec]
# Running sched/pipe benchmark...	|       	# Running sched/pipe benchmark...
     Total time: 23.604 [sec]	        |       	     Total time: 21.769 [sec]
      23.604037 usecs/op	        |       	      21.769218 usecs/op
          42365 ops/sec	                |       	          45936 ops/sec
```

###### Student’s T-test (ops/sec) perf bench:
  
```
x vcpu_is_preempted() disabled
+ vcpu_is_preempted() enabled
+--------------------------------------------------------------------------+
|+ xxx                                                  +                  |
|++xxxx   +++        +                    +         +   +    +  +         +|
|  |A||__________________________A________M_________________|              |
+--------------------------------------------------------------------------+
    N           Min           Max        Median           Avg        Stddev
x  14         42196         42751         42352       42378.5     146.35665
+  14         41698         57456         50549     48621.143     5868.5519
Difference at 95.0% confidence
	6242.64 +/- 3225.71
	14.7307% +/- 7.61166%
	(Student's t, pooled s = 4150.98)
```
