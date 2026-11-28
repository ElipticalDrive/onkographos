# Generalized Joseph Projector with Linear Interpolation

Also called: Branchless Generalized Joseph Projector (GJP), GJPlin, GJPspl, axis-aligned grid traversal with bilinear/spline interpolation, linear interpolating GJPlin algorithm

## Overview
The Generalized Joseph Projector with Linear Interpolation (GJPlin) is a branchless, ray-driven forward projection algorithm designed for efficient GPU execution in cone-beam CT reconstruction [S1, unofficial]. It solves the problem of accurately sampling a 3D voxel volume along arbitrary ray directions while maintaining high memory bandwidth efficiency and avoiding the artifacts associated with nearest-neighbor sampling [S1, unofficial]. Unlike Siddon’s DDA, which degenerates to nearest-neighbor sampling when rays align with grid axes, GJPlin uses driving-axis-aligned sampling to ensure an optimal, non-redundant number of samples, combined with 2D bilinear interpolation in the remaining dimensions to model the integrating nature of detector pixels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Source point $\tilde{s}$ (vector), detector pixel coordinates $\tilde{d}$ (vector), volume grid dimensions $V_{dim}$, and volume data array $V$.
*   **Output:** Accumulated projection value $a$ (scalar).

**Data Structures:**
*   Volume data $V$ is accessed via integer indices $(i, j, k)$.
*   Vectors are 3D Cartesian coordinates.

**Algorithm Steps:**

1.  **Ray Definition:** Calculate the ray direction vector $\tilde{r} = \tilde{d} - \tilde{s}$ [S1, unofficial].
2.  **Driving Axis Identification:** Identify the driving axis $m$ as the index with the largest absolute component in $\tilde{r}$:
    $$ m \leftarrow \text{argmax}_i(|r_i|) $$
    [S1, unofficial]
3.  **Sampling Increment and Offset:**
    *   Calculate the sampling increment vector $\tilde{r}$ (normalized to unit steps along the driving axis):
        $$ \tilde{r} \leftarrow \tilde{r} / r_m $$
        [S1, unofficial]
    *   Calculate the sampling offset $o$ to ensure the first sampling plane intersects the driving axis at an integer coordinate:
        $$ o \leftarrow -s_m $$
        [S1, unofficial]
    *   This ensures that for any integer $i$, the driving axis component of the sampling point $\tilde{p}^{(i)}$ is integer: $[\tilde{s} + o\tilde{r}]_m = 0 \implies p^{(i)}_m = i + \text{const}$ [S1, unofficial].
4.  **Loop Bounds:** Determine the number of sampling points $i_{max}$ based on the volume extent along the driving axis $m$:
    $$ i_{max} \leftarrow V_{dim}[m] $$
    [S1, unofficial]
5.  **Initialization:** Initialize accumulator $a \leftarrow 0$.
6.  **Sampling Loop:** Iterate $i$ from $0$ to $i_{max} - 1$:
    a.  **Calculate Sampling Point:**
        $$ \tilde{p} \leftarrow \tilde{s} + (o + i) \cdot \tilde{r} $$
        [S1, unofficial]
    b.  **Boundary Check:** If $\tilde{p}$ is outside the volume bounds, skip to the next iteration [S1, unofficial].
    c.  **Grid Indices:** Calculate floor and ceiling indices for the sampling point:
        $$ \tilde{p}_{fl} \leftarrow \text{floor}(\tilde{p}) $$
        $$ \tilde{p}_{cl} \leftarrow \text{ceil}(\tilde{p}) $$
        [S1, unofficial]
    d.  **Assert Driving Axis Integer Property:** Verify that the driving axis component is integer:
        $$ p_{fl,m} = p_{cl,m} = p_m $$
        [S1, unofficial]
    e.  **Calculate Distances:** Compute the fractional distances from the floor indices:
        $$ \tilde{d}_k \leftarrow p_k - p_{fl,k} \quad \text{for } k \in \{1, 2, 3\} $$
        [S1, unofficial]
    f.  **Calculate Interpolation Weights:**
        *   Compute floor weights using the linear kernel $w_{lin}(d) = 1 - d$:
            $$ w_{fl,k} \leftarrow 1 - \tilde{d}_k $$
            [S1, unofficial]
        *   Compute complementary ceiling weights:
            $$ w_{cl,k} \leftarrow 1 - w_{fl,k} $$
            [S1, unofficial]
        *   Set driving axis weights to 1 (since the driving axis is integer-aligned):
            $$ w_{fl,m} \leftarrow 1, \quad w_{cl,m} \leftarrow 1 $$
            [S1, unofficial]
    g.  **Accumulate Contributions:** Add the weighted contributions from the four nearest neighbor voxels in the plane perpendicular to the driving axis. The indices for the four voxels are combinations of floor/ceiling indices for the non-driving axes, while the driving axis index is fixed at $p_{fl,m}$ (which equals $p_{cl,m}$):
        $$ a \leftarrow a + V[p_{fl,1}, p_{fl,2}, p_{fl,3}] \cdot \|\tilde{r}\| \cdot w_{fl,1} \cdot w_{fl,2} \cdot w_{fl,3} $$
        $$ a \leftarrow a + V[p_{fl,1}, p_{cl,2}, p_{cl,3}] \cdot \|\tilde{r}\| \cdot w_{fl,1} \cdot w_{cl,2} \cdot w_{cl,3} $$
        $$ a \leftarrow a + V[p_{cl,1}, p_{fl,2}, p_{cl,3}] \cdot \|\tilde{r}\| \cdot w_{cl,1} \cdot w_{fl,2} \cdot w_{cl,3} $$
        $$ a \leftarrow a + V[p_{cl,1}, p_{cl,2}, p_{fl,3}] \cdot \|\tilde{r}\| \cdot w_{cl,1} \cdot w_{cl,2} \cdot w_{fl,3} $$
        [S1, unofficial]
        *Note: The term $\|\tilde{r}\|$ represents the scaling factor for the ray length per step. In the normalized formulation where $\tilde{r}$ is divided by $r_m$, this term accounts for the physical length of the voxel traversal along the ray direction.*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generalized_joseph_projector_linear(s, d, volume, volume_dims):
    """
    s: Source point (vec3)
    d: Detector pixel center (vec3)
    volume: 3D array of voxel values
    volume_dims: Tuple (dim_x, dim_y, dim_z)
    """
    # 1. Ray definition
    r_vec = d - s
    
    # 2. Driving axis identification
    # m is the index (0, 1, or 2) with the largest absolute component
    abs_r = abs(r_vec)
    m = argmax(abs_r)
    
    # 3. Sampling increment and offset
    # Normalize r_vec by its component along the driving axis m
    r_hat = r_vec / r_vec[m]
    
    # Sampling offset ensures the first sample plane intersects the driving axis at an integer
    o = -s[m]
    
    # 4. Loop bounds
    i_max = volume_dims[m]
    
    # Norm of the original ray direction vector (used for scaling)
    # Note: The algorithm description uses ||r|| in the accumulation.
    # In the normalized step formulation, the physical length per step is ||r_hat||.
    # However, standard Joseph projector implementations often scale by the voxel size
    # or ray segment length. Here we follow the source formula strictly:
    # a += volume[...] * ||r|| * weights
    # But wait, r_hat is unit step in driving axis. The physical length of one step
    # along the ray is ||r_hat||. The source formula says ||r||.
    # Let's look at Eq 17-20: "volume[...] * ||r|| * weights".
    # And r_hat = r / r_m.
    # If we use r_hat in p = s + (o+i)*r_hat, then the step size is ||r_hat||.
    # The source text says: "The increment vector r~ ... will be chosen such that ...
    # p(i) = (s + o r~) + i r~". And "r~ = r / r_m".
    # So the step vector is r_hat. The length of this step is ||r_hat||.
    # The formula in the notes says "||r||". This is likely a notation inconsistency
    # in the notes vs the paper's Eq 17-20 where r~ is used.
    # Given "r~" is the sampling increment, the length should be ||r~||.
    # We will use ||r_hat|| as the scaling factor per step.
    step_length = norm(r_hat)
    
    accumulator = 0.0
    
    # 6. Sampling Loop
    for i in range(i_max):
        # a. Calculate sampling point
        p = s + (o + i) * r_hat
        
        # b. Boundary check
        # Check if p is within volume bounds [0, dims-1] for all axes
        if not (0 <= p[0] < volume_dims[0] and
                0 <= p[1] < volume_dims[1] and
                0 <= p[2] < volume_dims[2]):
            continue
            
        # c. Grid indices
        p_fl = floor(p)
        p_cl = ceil(p)
        
        # d. Assert driving axis integer property (implicit in logic)
        # p_fl[m] == p_cl[m] == integer part of p[m]
        
        # e. Calculate distances
        # d_k = p_k - p_fl_k
        dist = p - p_fl
        
        # f. Calculate Interpolation Weights
        # Linear kernel: w(d) = 1 - d
        w_fl = 1.0 - dist
        w_cl = 1.0 - w_fl  # Equivalent to dist
        
        # Set driving axis weights to 1
        w_fl[m] = 1.0
        w_cl[m] = 1.0
        
        # g. Accumulate Contributions
        # Voxel 1: fl, fl, fl
        idx1 = (int(p_fl[0]), int(p_fl[1]), int(p_fl[2]))
        val1 = volume[idx1[0], idx1[1], idx1[2]]
        weight1 = w_fl[0] * w_fl[1] * w_fl[2]
        accumulator += val1 * step_length * weight1
        
        # Voxel 2: fl, cl, cl
        idx2 = (int(p_fl[0]), int(p_cl[1]), int(p_cl[2]))
        val2 = volume[idx2[0], idx2[1], idx2[2]]
        weight2 = w_fl[0] * w_cl[1] * w_cl[2]
        accumulator += val2 * step_length * weight2
        
        # Voxel 3: cl, fl, cl
        idx3 = (int(p_cl[0]), int(p_fl[1]), int(p_cl[2]))
        val3 = volume[idx3[0], idx3[1], idx3[2]]
        weight3 = w_cl[0] * w_fl[1] * w_cl[2]
        accumulator += val3 * step_length * weight3
        
        # Voxel 4: cl, cl, fl
        idx4 = (int(p_cl[0]), int(p_cl[1]), int(p_fl[2]))
        val4 = volume[idx4[0], idx4[1], idx4[2]]
        weight4 = w_cl[0] * w_cl[1] * w_fl[2]
        accumulator += val4 * step_length * weight4
        
    return accumulator
```

## Subtopics

### Generalized Joseph Projector with Spline Interpolation
A variant of GJP that replaces the linear interpolation kernel with a spline kernel $w_{spl}(d) = 1 - 3d + 2d^2$ [S1, unofficial]. This kernel is smooth and differentiable at grid points, reducing high-frequency artifacts compared to the kinked linear kernel, though it may have comparable residuals to oversampled DDA [S1, unofficial]. It uses the same driving-axis-aligned sampling and branchless structure as GJPlin [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The algorithm performs a fixed number of operations per sampling point, proportional to the volume extent along the driving axis $i_{max}$ [S1, unofficial].
*   Memory access pattern is coherent along the driving axis, ensuring high cache hit rates for parallel threads [S1, unofficial].

**Measured Performance:**
*   On a 512³ Shepp Logan phantom projected onto a 512² detector with a 10° cone angle:
    *   **GTX 970:** GJPlin takes 4.97 ms with 334 GB/s memory access rate [S1, unofficial].
    *   **GTX 1080:** GJPlin takes 2.40 ms with 686 GB/s memory access rate [S1, unofficial].
*   GJPlin outperforms regular DDA by a factor of up to 2, despite accessing about twice as much memory on average [S1, unofficial].
*   GJPspl (Spline variant) shows similar performance characteristics to GJPlin on the same hardware [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Branchless execution makes it highly efficient on GPUs. It avoids the artifacts of DDA when rays are parallel to grid axes. It achieves the best reconstruction quality among tested methods (GJP, DDA, DDA 2x) [S1, unofficial].
*   **Limitations:** Accesses more memory than DDA. Performance is optimal when the driving axis does not coincide with the fastest-changing memory index (though the source notes this generally, the branchless nature mitigates some overhead) [S1, unofficial].

**When to Use:**
*   Use for calculating individual projections within read-and-write memory on GPUs, specifically for cone beam CT reconstruction [S1, unofficial].
*   Choose over DDA when ray alignment artifacts are a concern and higher accuracy is needed without the cost of oversampling.
*   Choose GJPlin for best overall quality/speed trade-off; choose GJPspl if high-frequency artifact reduction is prioritized and the slight increase in computational complexity (spline evaluation) is acceptable.

## Background
The Generalized Joseph Projector was introduced by Jonas Graetz (Dittmann) in 2016/2020 [S1, unofficial]. It builds on the Joseph method [10586-joseph-s-method.md], which uses bilinear interpolation on intersection points. The "Generalized" aspect refers to the branchless formulation that unifies the handling of different driving axes, allowing for efficient GPU implementation. The method addresses the conflict between sampling quality and processing speed in GPU-based CT reconstruction, leveraging the coherence of sampling patterns to maximize memory bandwidth utilization [S1, unofficial].

## Sources
[S1] High performance volume ray casting: A branchless generalized Joseph projector, https://arxiv.org/abs/1609.00958v2, fetched 2026-10-24 (unofficial)
