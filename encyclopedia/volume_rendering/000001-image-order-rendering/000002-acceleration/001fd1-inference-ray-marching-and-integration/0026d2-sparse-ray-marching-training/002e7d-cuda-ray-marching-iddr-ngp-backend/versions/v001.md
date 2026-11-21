# CUDA Ray Marching (IDDR-NGP Backend)

## Overview
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation [S1, unofficial]. It serves as the core computational unit for the IDDR-NGP backend, enabling differentiable rendering by computing gradients alongside radiance accumulation [S1, unofficial]. It distinguishes itself by organizing computation per-warp for spherical harmonic (SH) evaluation and using shared memory for ray specification caching to minimize global memory traffic [S1, unofficial].

## How it works
**Inputs and Outputs:**
The kernel takes ray origins $\mathbf{rays\_o} \in \mathbb{R}^{N \times 3}$, ray directions $\mathbf{rays\_d} \in \mathbb{R}^{N \times 3}$, near planes $\mathbf{near} \in \mathbb{R}^{N}$, and far planes $\mathbf{far} \in \mathbb{R}^{N}$ [S1, unofficial]. It outputs sampled positions $\mathbf{xyzs} \in \mathbb{R}^{N \times n\_step \times 3}$, directions $\mathbf{dirs} \in \mathbb{R}^{N \times n\_step \times 3}$, and deltas $\mathbf{deltas} \in \mathbb{R}^{N \times n\_step \times 2}$, where the first delta channel stores RGB contribution weights and the second stores depth contributions [S1, unofficial]. The density grid is accessed via a `density_bitfield` tensor of type `uint8` [S1, unofficial].

**Data Structures:**
The hashed density grid is stored in a `uint8` tensor. The exact hashing function (e.g., hash table size, collision resolution) is not specified in the provided sources [S1, unofficial]. The grid resolution is defined by `resolution` (a 3D vector) and a bounding box defined by `bound` [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** For each ray $i \in [0, N)$, initialize the current position $\mathbf{p} = \mathbf{rays\_o}[i]$ and accumulated transmittance $T = 1.0$. Initialize step counter $k = 0$. [G: Standard ray marching initialization].
2.  **Ray Bounds:** Determine the intersection interval $[t_{near}, t_{far}]$ using $\mathbf{near}[i]$ and $\mathbf{far}[i]$. [G: Clip ray segment to bounding box if necessary].
3.  **Marching Loop:** While $t < t_{far}$ and $T > \epsilon$ (where $\epsilon$ is a small threshold, e.g., $10^{-4}$):
    a.  **Sampling:** Compute the next sample position $\mathbf{x} = \mathbf{p} + t \cdot \mathbf{rays\_d}[i]$. [G: Apply perturbation if enabled, as noted in definition].
    b.  **Density Lookup:** Query the hashed density grid at $\mathbf{x}$ using `density_bitfield`. Let the retrieved density be $\sigma$. [S1, unofficial].
    c.  **Adaptive Step:** Calculate step size $\Delta t$. The sources mention "adaptive step sizes" but do not provide the specific formula (e.g., based on density threshold or fixed grid spacing) [S1, unofficial]. [G: Typically $\Delta t$ is derived from the grid resolution or density magnitude].
    d.  **Accumulation:** Compute opacity $\alpha = 1 - \exp(-\sigma \cdot \Delta t)$. Update transmittance $T \leftarrow T \cdot (1 - \alpha)$. Store $\Delta t$ in $\mathbf{deltas}[i, k, 0]$ and $\Delta t$ in $\mathbf{deltas}[i, k, 1]$ [S1, unofficial]. Store $\mathbf{x}$ in $\mathbf{xyzs}[i, k]$ and $\mathbf{rays\_d}[i]$ in $\mathbf{dirs}[i, k]$ [S1, unofficial].
    e.  **Advance:** $t \leftarrow t + \Delta t$, $k \leftarrow k + 1$.
4.  **Termination:** If $k \ge n\_step$, stop. [G: $n\_step$ is a maximum step count parameter].

**Parameters:**
-   `density_bitfield`: `uint8` tensor for hashed density [S1, unofficial].
-   `bound`: Bounding box size [S1, unofficial].
-   `density_thresh`: Density threshold for early termination or step sizing [S1, unofficial].
-   `min_near`: Minimum near plane distance [S1, unofficial].
-   `bg_radius`: Background radius [S1, unofficial].

**Edge Cases:**
-   **Precision:** Floating-point precision in density lookup and transmittance accumulation can cause artifacts. [G: Use `float32` for accumulation].
-   **Boundaries:** Rays outside the `bound` should be handled or clipped. [G: Check ray-box intersection].
-   **Degenerate Input:** Zero-length rays or invalid directions should be skipped or handled with a default value. [G: Check for zero direction vector].

## Subtopics

### [Ray Tracing Kernel](5968-ray-tracing-kernel.md)
The Ray Tracing Kernel is a CUDA kernel implementation for forward and backward ray tracing through a sparse grid volume, handling spherical harmonic (SH) evaluation and ray bounds. It serves as the core computational unit for the IDDR-NGP backend, enabling differentiable rendering by computing gradients alongside radiance accumulation. It distinguishes itself by organizing computation per-warp for SH evaluation and using shared memory for ray specification caching to minimize global memory traffic.

## Operating characteristics
**Time and Memory Complexity:**
Time complexity is $O(N \cdot n\_step)$, where $N$ is the number of rays and $n\_step$ is the number of samples per ray [S1, unofficial]. Memory complexity is dominated by the hashed density grid and the output tensors $\mathbf{xyzs}$, $\mathbf{dirs}$, and $\mathbf{deltas}$ [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
The method is designed for GPU acceleration, leveraging CUDA kernels for parallel processing [S1, unofficial].

**Strengths and Limitations:**
Strengths include adaptive step sizes and perturbation support, which can improve rendering quality and training stability [S1, unofficial]. Limitations include the lack of detailed specification for the hashing function and adaptive step size formula, which may hinder implementation [S1, unofficial].

**When to Use:**
Use this method when implementing the IDDR-NGP backend for differentiable volume rendering with sparse hashed grids [S1, unofficial].

## Background
Origin: The method is part of the IDDR-NGP project, an unofficial implementation based on the NeRF-Factory repository [S1, unofficial]. It builds on [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md).

## Sources
[S1] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
