# NeRF Ray Marching Optimizations

Also called: Accelerated NeRF Ray Marching, Occupancy Grids for Empty Space Skipping

## Overview
A set of three techniques to optimize NeRF ray marching performance with imperceptible error: exponential stepping, empty space skipping, and sample compaction [S1]. It builds on [Uniform Ray Marching](4810-uniform-ray-marching.md) by replacing fixed uniform intervals with adaptive step sizes and skipping empty regions via multiscale occupancy grids [S1].

## How it works
**Inputs:** Ray origin $o$, ray direction $d$, scene bounding box size $s$ (largest axis), NeRF model $f_{\theta}$, current training step count [S1].
**Outputs:** Sample positions along the ray, transmittance, accumulated color [S1].

**Data Structures:**
1. **Occupancy Grids:** A cascade of $K$ grids. Each grid has resolution $128^3$. Cells are laid out in Morton (z-curve) order. Each cell stores occupancy as a single bit [S1].
2. **Density Grids:** A second set of $K$ grids with the same layout, storing full-precision floating-point density values [S1].

**Parameters:**
* Fixed step size for unit cube scenes: $\Delta t = \sqrt{3}/1024$ [S1].
* Transmittance stopping threshold: $10^{-4}$ [S1].
* Number of grids $K$: 1 for synthetic NeRF scenes; $K \in [1, 5]$ for larger real-world scenes [S1].
* Grid resolution: $128^3$ [S1].
* Update frequency: Every 16 training iterations [S1].
* Density decay factor: 0.95 [S1].
* Occupancy threshold $t$: $0.01 \cdot 1024/\sqrt{3}$ [S1].
* Number of candidate cells $M$: $K \cdot 128^3$ for first 256 steps; $K \cdot 128^3 / 2$ for subsequent steps [S1].

**Steps:**

1. **Step Size Selection:**
   * If the scene is bounded to the unit cube $[0,1]^3$, set $\Delta t = \sqrt{3}/1024$ [S1].
   * Otherwise, set $\Delta t = t/256$, where $t$ is the current distance along the ray. Clamp $\Delta t$ to $[-\frac{\sqrt{3}}{1024} \cdot s, \frac{\sqrt{3}}{1024} \cdot s]$, where $s$ is the size of the largest axis of the scene's bounding box [S1].

2. **Grid Query:**
   * For a sample position $x$, determine which grid to query. Among the grids covering $x$, select the finest one with cell side-length larger than the current step size $\Delta t$ [S1].
   * The domain for grid $k$ is $[-2^{k-1}+0.5, 2^{k-1}+0.5]^3$, centered around $(0.5, 0.5, 0.5)$ [S1].

3. **Empty Space Skipping:**
   * Query the occupancy bit of the cell containing $x$ in the selected grid.
   * If the bit is low (0), skip the sample [S1].

4. **Ray Marching Loop:**
   * Initialize transmittance $T = 1.0$.
   * While $T > 10^{-4}$:
     * Calculate next sample position $x = o + t \cdot d$.
     * Check occupancy. If occupied, query NeRF model $f_{\theta}(x)$ to get density $\sigma$ and color $C$.
     * Update transmittance: $T \leftarrow T \cdot \exp(-\sigma \cdot \Delta t)$.
     * Accumulate color: $C_{out} \leftarrow C_{out} + T \cdot (1 - \exp(-\sigma \cdot \Delta t)) \cdot C$.
     * Update distance: $t \leftarrow t + \Delta t$.
     * Recalculate $\Delta t$ based on new $t$ if using exponential stepping [S1].

5. **Grid Update (Every 16 iterations):**
   * Decay density values in all cells by a factor of 0.95 [S1].
   * Sample $M$ candidate cells:
     * If training step $< 256$: Sample $M = K \cdot 128^3$ cells uniformly without repetition [S1].
     * If training step $\ge 256$: Set $M = K \cdot 128^3 / 2$. Partition into two sets:
       * First $M/2$ cells sampled uniformly among all cells [S1].
       * Remaining $M/2$ cells sampled via rejection sampling restricted to currently occupied cells [S1].
   * For each candidate cell, set its density value to $\max(\text{current}, \text{NeRF density at random location in cell})$ [S1].
   * Update occupancy bits by thresholding density with $t = 0.01 \cdot 1024/\sqrt{3}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def morton_3d(x, y, z):
    code = 0
    for i in range(7):
        code |= (x & (1 << i)) << (2 * i)
        code |= (y & (1 << i)) << (2 * i + 1)
        code |= (z & (1 << i)) << (2 * i + 2)
    return code

def index_to_coords(idx, resolution=128):
    z = idx % resolution
    idx //= resolution
    y = idx % resolution
    x = idx // resolution
    return x, y, z

def sample_point_in_cell(grid_idx, cell_idx, resolution=128):
    x, y, z = index_to_coords(cell_idx, resolution)
    k = grid_idx + 1
    min_val = -2**(k-1) + 0.5
    width = 2**k - 1
    cell_size = width / resolution
    return (min_val + x * cell_size + random.random() * cell_size,
            min_val + y * cell_size + random.random() * cell_size,
            min_val + z * cell_size + random.random() * cell_size)

def query_occupancy_grid(x, grids, step_size):
    best_k = -1
    for k_idx in range(len(grids)):
        k = k_idx + 1
        min_val = -2**(k-1) + 0.5
        width = 2**k - 1
        if not (min_val <= x[0] <= min_val + width and
                min_val <= x[1] <= min_val + width and
                min_val <= x[2] <= min_val + width):
            continue
        cell_side = width / 128.0
        if cell_side > step_size:
            best_k = k_idx
            break
    if best_k == -1:
        for k_idx in range(len(grids)):
            k = k_idx + 1
            min_val = -2**(k-1) + 0.5
            width = 2**k - 1
            if (min_val <= x[0] <= min_val + width and
                min_val <= x[1] <= min_val + width and
                min_val <= x[2] <= min_val + width):
                best_k = k_idx
                break
    if best_k == -1:
        return 0
    k = best_k + 1
    min_val = -2**(k-1) + 0.5
    width = 2**k - 1
    cell_side = width / 128.0
    u = int((x[0] - min_val) / cell_side)
    v = int((x[1] - min_val) / cell_side)
    w = int((x[2] - min_val) / cell_side)
    u = max(0, min(127, u))
    v = max(0, min(127, v))
    w = max(0, min(127, w))
    idx = morton_3d(u, v, w)
    return grids[best_k][idx]

def update_grids(grids, density_grids, step, k_count):
    if step % 16 != 0:
        return
    for i in range(k_count):
        for j in range(len(density_grids[i])):
            density_grids[i][j] *= 0.95
    if step < 256:
        M = k_count * 128**3
        all_indices = list(range(128**3))
        random.shuffle(all_indices)
        selected_indices = all_indices[:M]
    else:
        M = k_count * 128**3 // 2
        all_indices = list(range(128**3))
        random.shuffle(all_indices)
        selected_indices = all_indices[:M//2]
        occupied_indices = [i for i, bit in enumerate(grids[0]) if bit == 1]
        for _ in range(M//2):
            selected_indices.append(random.choice(occupied_indices))
    threshold = 0.01 * 1024 / math.sqrt(3)
    for idx in selected_indices:
        grid_idx = idx // (128**3)
        cell_idx = idx % (128**3)
        point = sample_point_in_cell(grid_idx, cell_idx)
        sigma = nerf_model(point)
        density_grids[grid_idx][cell_idx] = max(density_grids[grid_idx][cell_idx], sigma)
        if density_grids[grid_idx][cell_idx] > threshold:
            grids[grid_idx][cell_idx] = 1
        else:
            grids[grid_idx][cell_idx] = 0

def nerf_ray_marching(o, d, s, nerf_model, grids, density_grids, step):
    t = 0.0
    T = 1.0
    C_out = [0.0, 0.0, 0.0]
    if s <= 1.0:
        delta_t = math.sqrt(3) / 1024.0
    else:
        delta_t = math.sqrt(3) / 1024.0 * s
    while T > 1e-4:
        x = (o[0] + t * d[0], o[1] + t * d[1], o[2] + t * d[2])
        occ = query_occupancy_grid(x, grids, delta_t)
        if occ == 1:
            sigma, C = nerf_model(x)
            alpha = 1.0 - math.exp(-sigma * delta_t)
            T = T * math.exp(-sigma * delta_t)
            C_out[0] += T * alpha * C[0]
            C_out[1] += T * alpha * C[1]
            C_out[2] += T * alpha * C[2]
        t += delta_t
        if s > 1.0:
            delta_t = t / 256.0
            max_delta_t = math.sqrt(3) / 1024.0 * s
            delta_t = max(1e-6, min(max_delta_t, delta_t))
    return C_out
```

## Operating characteristics
* **Time Complexity:** Computation cost grows only logarithmically in scene diameter due to exponential step size growth [S1].
* **Memory Complexity:** $K \times 128^3$ bits for occupancy grids and $K \times 128^3$ floats for density grids [S1].
* **Performance:** One order of magnitude faster than using triangle BVH for rayshape intersections on RTX 3090 GPU [S1].
* **Strengths:** Imperceptible error, significant speedup for large scenes [S1].
* **Limitations:** Requires maintaining and updating grids, which adds complexity [S1].

## Background
Originated in "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding" by Müller et al. (2022) [S1]. Used in Instant-NGP for fast NeRF training and rendering.

## Sources
[S1] Instant Neural Graphics Primitives with a Multiresolution Hash Encoding, https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
