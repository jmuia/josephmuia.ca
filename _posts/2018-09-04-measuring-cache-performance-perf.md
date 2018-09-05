---
layout: post
title: Measuring cache performance + perf
---

Allen Downey's _[Think OS](http://greenteapress.com/thinkos/)_ discusses cache performance and provides some [code](https://github.com/AllenDowney/ThinkOS/tree/master/code/cache) to experiment with on your own computer.

I'm going to run through the experiment and supplement the data with L1 cache events collected by `perf`.

* TOC
{:toc}

# Method
The program iterates through an array, measuring the average time to read and write an element. We vary the size of the array as well as the stride/step of the iteration to infer information about the cache.

Downey provides a more thorough explanation of the particulars of the code in Section 7.4 Measuring cache performance.

# Theory
_Think OS_ presents some theory about cache performance, which I will reference throughout this post. Numbering and emphasis mine.

> 1. The program reads through the array many times, so it has plenty of **temporal locality**. If the entire array fits in cache, we expect the average miss penalty to be near 0.
> 2. When the stride is 4 bytes, we read every element of the array, so the program has plenty of **spatial locality**. If the block size is big enough to contain 64 elements, for example, the hit rate would be 63/64, even if the array does not fit in cache.
> 3. If the stride is equal to the block size (or greater), the **spatial locality** is effectively zero, because each time we read a block, we only access one element. In that case we expect to see the maximum miss penalty.

> In summary, we expect good cache performance if the array is smaller than the cache size or if the stride is smaller than the block size. Performance only degrades if the array is bigger than the cache and the stride is large.

# Results
Let's see the theory in practice.

![Chart comparing cache performance (access time in ns) for various array sizes and iteration strides](/img/cache-performance.svg)

## L1 cache

In the chart above we can see cache performance is good, regardless of stride, for arrays less than 2<sup>15</sup> bytes.

One might infer based on (1) that the size of my L1 cache is 2<sup>15</sup> bytes. I can use `lshw -C memory` to view information about my hardware. There are two L1 caches, an instruction cache and data cache, both 32 KiB (2<sup>15</sup> B).

## Block size

At 2<sup>16</sup> bytes, just beyond the L1 cache, there is an apparent divide between two groups of strides. 64 B and greater strides and have a much higher access time. Based on (2) and (3) we might suspect the block size is 64 B.

According to `/proc/cpuinfo` and/or `getconf -a | grep -i cache` I can see the line size (block size) for the caches as 64 B. Neat.

## L2, L3, L4 caches
`lshw` has information about other caches too: L2 is 256 KiB (2<sup>18</sup> B), L3 is 6 MiB (2<sup>22.6</sup> B), and L4 is 128 MiB (2<sup>27</sup> B).

I suspect the rest of the data is a bit more nuanced as the L2-4 caches are unified, but there appear to be inflection points near the limits of these caches.

## perf

We can use `perf` to collect info about caches misses.

`perf stat -e L1-dcache-loads,L1-dcache-load-misses ./cache` will give us the loads and misses, and it'll compute the cache miss rate.

### Fits in L1 dcache

If the array fits in L1 dcache, regardless of the stride size, the performance cache miss rate is low at 0.01% due to temporal locality (1).

Stride 4 bytes:
```
1,270,419,490      L1-dcache-loads
      125,823      L1-dcache-load-misses     #    0.01% of all L1-dcache hits
```

Stride 64 bytes (the block size):
```
1,223,318,610      L1-dcache-loads 
      145,630      L1-dcache-load-misses     #    0.01% of all L1-dcache hits
```

### Does not fit in L1 dcache

If the array does not fit in L1 dcache, temporal locality is poor or non-existent and we can see the miss rate get progressively worse (doubling) as the stride increases (doubles). Spatial locality degrades until we reach the block size (64 B), where it's effectively zero due to (2) and (3).

Stride 4 bytes:
```
1,304,443,689      L1-dcache-loads 
   81,658,703      L1-dcache-load-misses     #    6.26% of all L1-dcache hits
```

Stride 8 bytes:
```
1,294,548,933      L1-dcache-loads
  161,528,587      L1-dcache-load-misses     #   12.48% of all L1-dcache hits
```

Stride 16 bytes:
```
1,256,901,626      L1-dcache-loads 
  312,769,928      L1-dcache-load-misses     #   24.88% of all L1-dcache hits
```

Stride 32 bytes:
```
1,206,040,883      L1-dcache-loads
  599,584,387      L1-dcache-load-misses     #   49.72% of all L1-dcache hits
```

Stride 64 bytes (as bad as it gets):
```
  618,049,217      L1-dcache-loads
  613,581,569      L1-dcache-load-misses     #   99.28% of all L1-dcache hits
```


Stride 128 bytes (can't get any worse):
```
  620,190,895      L1-dcache-loads
  615,688,933      L1-dcache-load-misses     #   99.27% of all L1-dcache hits
```

# Parting thoughts
It's cool to see the experiment data align with the theory about the relationship between cache size, block size, and cache performance. It's _more_ cool that we can check the chart results against our actual hardware information and data collected with `perf`.


