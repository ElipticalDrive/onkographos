# Breakable Integrals

Also called: Distributable Integral Functions

## Overview
Breakable integrals are a family of integral functions defined over a domain $[t_1, t_{N+1}]$ that can be decomposed into a sum of products of integrals over individual segments $[t_k, t_{k+1}]$ [S1]. This decomposition enables the parallel or distributed computation of the total integral by allowing partial results from independent segments to be combined [S1]. Unlike standard parallel primitives that distribute data elements, this family distributes the mathematical structure of the integral itself, making it suitable for multi-GPU implementations where segments are processed independently and aggregated [S1].

## Shared principles
The core mechanism of this family is the mapping of computational tasks (pixels, rays, or data elements) to parallel execution units (threads, warps, or cores) [G]. Common principles include:
1. **Thread Mapping:** Assigning one or more threads to process specific data elements (e.g., one thread per pixel or one thread per voxel sample) [G].
2. **Memory Hierarchy Utilization:** Leveraging global memory for large datasets, shared/local memory for intermediate results (reductions), and registers for per-thread state [G].
3. **Synchronization:** Managing concurrency conflicts using atomic operations, barriers, or spatial partitioning (tiles) to ensure data consistency without excessive serialization [G].
4. **Data Parallelism:** Performing identical operations on different data elements simultaneously, often requiring coalesced memory access patterns to maximize bandwidth [G].

## Subtopics

### Transmittance
The transmittance function $T(t_1 \rightarrow t_{N+1}) = \exp \left( - \int_{t_1}^{t_{N+1}} \sigma(t) dt \right)$ is a breakable integral because the exponential of an integral over a union of segments equals the product of the exponentials (transmittances) over each segment [S1]. Specifically, $T(t_1 \rightarrow t_{N+1}) = \prod_{i=1}^N T(t_i \rightarrow t_{i+1})$ [S1]. This property allows transmittance to be computed segment-wise and multiplied across GPUs [S1].

### Volume Rendering
The volume rendering equation $C(t_1 \rightarrow t_{N+1}) = \int_{t_1}^{t_{N+1}} T(t_1 \rightarrow t) \sigma(t) c(t) dt$ is a breakable integral because it is an integral of the product of transmittance $T(t_1 \rightarrow t)$ (which is breakable) and the source term $\sigma(t)c(t)$ [S1]. It fits the general form of the family by leveraging the breakability of transmittance [S1].

### Distortion Loss
The distortion loss $\mathcal{L}_{dist}(t_1 \rightarrow t_{N+1}) = 2 \int_{t_1}^{t_{N+1}} T(t_1 \rightarrow u) \sigma(u) S(t_1 \rightarrow u) du$ is a breakable integral [S1]. The inner component $S(t_1 \rightarrow u) = \int_{t_1}^u T(t_1 \rightarrow v) \sigma(v) (u - v) dv$ is breakable [S1]. The product $T(t_1 \rightarrow u)S(t_1 \rightarrow u)$ is breakable due to the closure properties of the family [S1], and the outer integral preserves this property [S1].

## Background
The concept of breakable integrals was introduced in the context of scaling Neural Radiance Fields (NeRFs) across multiple GPUs [S1]. The primary motivation was to enable efficient distributed training and inference by partitioning the ray integration domain [S1]. The family is characterized by its ability to decompose complex integrals into segment-wise operations that can be aggregated, facilitating parallel processing in distributed systems [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
