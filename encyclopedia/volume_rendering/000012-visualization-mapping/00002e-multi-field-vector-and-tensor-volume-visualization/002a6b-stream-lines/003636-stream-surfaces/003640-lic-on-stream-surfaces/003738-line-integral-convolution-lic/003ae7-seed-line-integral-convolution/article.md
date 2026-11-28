# Seed Line Integral Convolution

Also called: Seed LIC, Sparse Input Texture LIC, Sparse LIC, Streamline LIC, Volume LIC

## Overview
Seed Line Integral Convolution (Seed LIC) is a volume rendering method for visualizing 3D vector fields that computes Line Integral Convolution only along field lines originating from a sparse set of seed points, rather than covering every voxel in the output texture [S1, S2]. It solves the problem of high computational cost associated with Fast LIC when applied to large 3D datasets by exploiting the sparsity of the input texture [S1, S2]. Unlike its sibling [Image Space Fabric Visualization](10569-image-space-fabric-visualization.md) which renders explicit geometric primitives, Seed LIC generates a sparse texture of field lines, setting it apart from [Line Integral Convolution (LIC)](7940-line-integral-convolution-lic.md) by avoiding unnecessary computations in areas where field lines do not coincide with cells containing seed points [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D vector field $\mathbf{V}$, a scalar field for Region Of Interest (ROI) definition (e.g., vorticity magnitude), and a density factor $\delta \in [0,1]$ controlling seed point sparsity [S1, S2].
*   **Output:** A 3D scalar texture where voxel values represent the convolved intensity of field lines, normalized to the range $[0, 255]$ [S1, S2].

**Data Structures:**
*   **Input Texture ($T$):** A 3D grid of the same resolution as the vector field (or an enlarged resolution for detail enlargement) [S1, S2]. Voxels are initialized to 0, and selected seed voxels are set to 255 [S1, S2].
*   **Output Texture ($I$):** A 3D grid of the same resolution as the input texture, initialized to 0, storing the convolution result [S1, S2].
*   **Hit Count Map ($H$):** A 3D grid storing the number of field lines passing through each voxel, used for normalization [S2].

**Parameters:**
*   **Filter Kernel Length ($2L$):** Typically set to 21 times the grid cell size [S1].
*   **Field Line Length:** Typically set to 51 times the grid cell size [S1].
*   **Minimum Seed Distance:** Points within 3 voxels of an existing seed are discarded to prevent merging [S1, S2].
*   **Threshold Value:** Scalar value threshold for ROI inclusion [S2].
*   **Density Factor ($\delta$):** Probability threshold for seed placement [S2].

**Steps:**

1.  **Generate Sparse Input Texture:**
    *   Initialize all voxels in $T$ to 0 [S1, S2].
    *   For each voxel $v$:
        *   If scalar value($v$) > threshold value:
            *   Generate random number $r \in [0,1]$ [S2].
            *   If $r > \delta$:
                *   Set $T(v) = 255$ [S1, S2].
    *   Enforce minimum distance: Discard any seed points that fall within 3 voxels of a previously inserted seed point [S1, S2].

2.  **Initialize Output:**
    *   Initialize all voxels in $I$ to 0 [S1, S2].
    *   Initialize all voxels in $H$ to 0 [S2].

3.  **Trace Field Lines and Compute Convolution:**
    *   For each voxel $v$ in the grid:
        *   If $T(v) > 0$:
            *   Set starting position $x_0$ to the center of voxel $v$ [S2].
            *   Trace field line forward and backward from $x_0$ using a fifth-order adaptive Runge-Kutta method with a fixed sample interval [S1, S2].
            *   Compute convolution integral for $x_0$:
                $$I(x_0) = \int_{s_0-L}^{s_0+L} k(s-s_0) T(\alpha(s)) ds$$
                where $\alpha(s)$ is the curve parameterized by arc-length $s$, and $k$ is the filter kernel of length $2L$ [S1].
            *   Add $I(x_0)$ to $I(v)$ [S2].
            *   Increment $H(v)$ by 1 [G: Standard implementation detail required for the normalization step described in the evidence].
            *   Iterate along the field line for $m = 1$ to $M$ (where $M$ corresponds to the field line length limit):
                *   Update positions $x_m$ and $x_{-m}$ along the field line [S2].
                *   Compute convolution $I(x_m)$ and $I(x_{-m})$ using the same integral formula [S2].
                *   Add $I(x_m)$ to the voxel containing $x_m$ and increment its hit count [S2].
                *   Add $I(x_{-m})$ to the voxel containing $x_{-m}$ and increment its hit count [S2].

4.  **Normalize Output:**
    *   For each voxel $v$:
        *   Normalize intensity according to $H(v)$ [S2].
    *   Scale all voxel values in $I$ so that the maximum value is 255 [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def seed_lic(vector_field, scalar_field, density_factor, threshold, kernel_length, line_length):
    # 1. Generate Sparse Input Texture
    input_texture = zeros_like(vector_field)
    seed_points = []
    
    # Determine potential seeds based on ROI and density
    for v in all_voxels(vector_field):
        if scalar_field[v] > threshold:
            if random() > density_factor:
                seed_points.append(v)
    
    # Enforce minimum distance between seeds
    valid_seeds = []
    for v in seed_points:
        if not any(distance(v, existing) < 3 for existing in valid_seeds):
            valid_seeds.append(v)
            input_texture[v] = 255
            
    # 2. Initialize Output and Hit Count
    output_texture = zeros_like(vector_field)
    hit_count = zeros_like(vector_field)
    
    # 3. Trace Field Lines and Compute Convolution
    for v in valid_seeds:
        x0 = center_of_voxel(v)
        
        # Trace forward and backward using fixed-step integrator
        forward_line = trace_field_line(vector_field, x0, direction=1, length=line_length)
        backward_line = trace_field_line(vector_field, x0, direction=-1, length=line_length)
        
        # Compute convolution for start point
        conv_start = compute_convolution(forward_line, backward_line, input_texture, kernel_length)
        output_texture[v] += conv_start
        hit_count[v] += 1
        
        # Compute convolution for intermediate points
        for m in range(1, line_length + 1):
            xm = forward_line[m]
            x_minus_m = backward_line[m]
            
            conv_m = compute_convolution(forward_line, backward_line, input_texture, kernel_length, start_index=m)
            
            voxel_xm = voxel_containing_point(xm)
            voxel_x_minus_m = voxel_containing_point(x_minus_m)
            
            if voxel_xm is not None:
                output_texture[voxel_xm] += conv_m
                hit_count[voxel_xm] += 1
                
            if voxel_x_minus_m is not None:
                output_texture[voxel_x_minus_m] += conv_m
                hit_count[voxel_x_minus_m] += 1

    # 4. Normalize Output
    max_val = 0
    for v in all_voxels(vector_field):
        if hit_count[v] > 0:
            output_texture[v] /= hit_count[v]
            if output_texture[v] > max_val:
                max_val = output_texture[v]
                
    if max_val > 0:
        output_texture /= max_val
        output_texture *= 255
        
    return output_texture

def compute_convolution(forward_line, backward_line, input_texture, kernel_length, start_index=0):
    total = 0.0
    for s in range(-kernel_length, kernel_length + 1):
        idx = start_index + s
        if 0 <= idx < len(forward_line):
            pos = forward_line[idx]
            val = trilinear_interpolate(input_texture, pos)
            total += val * kernel_weight(s)
        elif -kernel_length <= -idx <= kernel_length:
            # Backward line indexing mirrors forward line around start_index
            b_idx = -idx
            if 0 <= b_idx < len(backward_line):
                pos = backward_line[b_idx]
                val = trilinear_interpolate(input_texture, pos)
                total += val * kernel_weight(s)
    return total

def trace_field_line(vector_field, start_pos, direction, length):
    points = [start_pos]
    current_pos = start_pos
    step_size = 1.0 / length
    for _ in range(length):
        # Simplified fifth-order adaptive Runge-Kutta with fixed sample interval
        k1 = vector_field_at(current_pos)
        k2 = vector_field_at(current_pos + 0.5 * step_size * k1)
        k3 = vector_field_at(current_pos + 0.5 * step_size * k2)
        k4 = vector_field_at(current_pos + step_size * k3)
        current_pos = current_pos + step_size * direction * (k1 + 2*k2 + 2*k3 + k4) / 6
        points.append(current_pos)
    return points

def vector_field_at(pos):
    return trilinear_interpolate(vector_field, pos)

def trilinear_interpolate(grid, pos):
    # Standard trilinear interpolation assuming pos is in grid coordinates
    x, y, z = pos
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    def val(i, j, k):
        return grid[min(i, grid.shape[0]-1), min(j, grid.shape[1]-1), min(k, grid.shape[2]-1)]
    return (val(ix,iy,iz)*(1-fx)*(1-fy)*(1-fz) + val(ix+1,iy,iz)*fx*(1-fy)*(1-fz) +
            val(ix,iy+1,iz)*(1-fx)*fy*(1-fz) + val(ix+1,iy+1,iz)*fx*fy*(1-fz) +
            val(ix,iy,iz+1)*(1-fx)*(1-fy)*fz + val(ix+1,iy,iz+1)*fx*(1-fy)*fz +
            val(ix,iy+1,iz+1)*(1-fx)*fy*fz + val(ix+1,iy+1,iz+1)*fx*fy*fz)

def center_of_voxel(v):
    return [v[0] + 0.5, v[1] + 0.5, v[2] + 0.5]

def voxel_containing_point(pos):
    return [int(pos[0]), int(pos[1]), int(pos[2])]

def zeros_like(grid):
    return [[0 for _ in range(grid.shape[1])] for _ in range(grid.shape[0])]

def all_voxels(grid):
    return [(i, j, k) for i in range(grid.shape[0]) for j in range(grid.shape[1]) for k in range(grid.shape[2])]

def distance(a, b):
    return ((a[0]-b[0])**2 + (a[1]-b[1])**2 + (a[2]-b[2])**2)**0.5

def random():
    import random as rnd
    return rnd.random()

def kernel_weight(s):
    # Box filter kernel of length 2L+1
    return 1.0 / (2 * kernel_length + 1)
```

## Subtopics

### [Direct Volume Rendering of LIC Textures](16714-direct-volume-rendering-of-lic-textures.md)
Direct Volume Rendering of LIC Textures is a visualization method that renders nerve fibre structures and other tissue types simultaneously by combining a Line Integral Convolution (LIC) texture with a probability map. It solves the problem of efficiently visualizing complex directional data in Diffusion Tensor Imaging (DTI) without the computational overhead of ensuring uniform coverage across the entire volume. Unlike streamline-based techniques which render explicit geometric primitives, this method uses volume rendering to represent finer details via noise texture resolution and allows for semi-transparent or nearly opaque fibre tracts. It sets itself apart from its sibling [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md) by utilizing the generated LIC texture and hit-count probability map as dual inputs for transfer function definition, rather than just rendering the texture itself.

### Sparse Volume LIC
Sparse Volume LIC is a variant of Seed LIC that emphasizes detail enlargement and aliasing reduction through specific parameter adjustments [S2]. It differs from standard Seed LIC by explicitly managing the relationship between input texture resolution and vector field resolution to maintain stroke accuracy [S2]. It is chosen when higher visual fidelity is required at the cost of increased computation due to smaller step sizes [S2].
*   **Step:** For each voxel $v$, if scalar value($v$) > threshold, compute random number in $[0,1]$; if random number > density factor, set input texture value($v$) = 255 [S2]. To avoid aliasing during detail enlargement, reduce the step size between sample points when computing convolution integrals inversely proportional to the texture size increase [S2].
*   **Parameter:** Threshold value: scalar value threshold used to determine if a voxel contributes to the input texture [S2]. Density factor: a value in $[0,1]$ controlling the sparsity of the input texture points [S2].
*   **Comparison:** Detail enlargement uses a smaller step size to ensure most voxels are covered by the field line when the input texture is larger than the vector field [S2].

### Aliasing Reduction via 3x3x3 Convolution
Aliasing Reduction via 3x3x3 Convolution is a post-processing step for Seed LIC that reduces aliasing artifacts by smoothing the output texture [S2]. It sets itself apart by explicitly addressing the "stair-stepped" appearance of voxel-based field lines through isotropic filtering [S2]. It is chosen when the visual smoothness of field lines is prioritized over strict geometric accuracy [S2].
*   **Definition:** Reducing aliasing in Seed LIC textures by convolving the output texture with a 3x3x3 filter to smear field lines and create smoother strokes [S2].
*   **Step:** Convolve the output texture with the 3x3x3 filter to smear field lines [S2].
*   **Parameter:** Convolution filter A: a 3x3x3 filter where the center entry $a_{222} = 1$ and the rest of the entries are set to 0.25 [S2].
*   **Pitfall:** Convolution leads to thicker strokes; to preserve thickness, increase the resolution of input and output texture by a factor of three prior to convolution [S2].
*   **Number:** Convolution of the LIC texture took 206.19 seconds [S2].

### Smart Seed Tracing
Smart Seed Tracing is a line search method that follows the vector field $\mathbf{k}$ in positive and negative directions from a starting point on mesh $M$ until it hits a 'smart seed' (Core or Halo region) [S4]. Unlike LIC, it does not integrate color along the line but stops at the first seed occurrence to determine color based on distance and seed region [S4]. It sets itself apart by using a line search approach with dampening factors instead of continuous integration [S4]. It is chosen when seamless line illustration with good contrast and smooth transitions between core and halo regions is required [S4].
*   **Formula:** Final color $R$ interpolation: $R = \frac{R_{-1} i_1 + R_1 i_{-1}}{i_{-1} + i_1}$, where $i_1$ and $i_{-1}$ are the iteration counts for positive and negative directions respectively [S4].
*   **Parameter:** Step size $\lambda$: The step size of the iteration [S4]. Dampening factor $\tau_w$: Controls the impact of the dampening factor $\tau$ [S4]. Direction $d$: Defines positive or negative direction of iteration, $d \in [-1, 1]$ [S4].
*   **Layout:** Input Textures: $T_{EXv}$ contains view-projected vector field $\mathbf{k}$. $T_{EXp}$ contains seed region properties ($R$ for stroke color, $G$ for permeability) [S4].
*   **Code:** Algorithm 1: Line search algorithm
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
id = 0; x0 = x; tau = 1;
while x0 not in S and x0 != contour:
    id += 1;
    x0 = x0 + d * lambda * T_EXv(x0);
    tau = tau * (1 - tau_w * (1 - T_EXp(x0).G));
Rd = tau * T_EXp(x0).R + (1 - tau)
```
*   **Pitfall:** Requires careful tuning of step size and dampening factors to avoid overshooting seed regions or producing discontinuous strokes [S4].

## Operating characteristics
*   **Time Complexity:** Significantly faster than Fast LIC for sparse textures. For a 256x256x256 texture with 17,341 seed points, Seed LIC took 70.9 seconds, compared to 752 seconds for Fast LIC [S2]. For a 128x128x128 texture, Fast LIC took 752 seconds while Seed LIC took 4.7 seconds [S2]. For a 398x298x798 texture, Fast LIC took about 25 hours, while Seed LIC took 26.86 seconds [S1, S2].
*   **Memory Complexity:** Requires storage for the input texture, output texture, and hit count map. The input texture is sparse, but stored as a dense grid [S1, S2].
*   **Measured Performance:** On an SGI ONYX 3800 using a single 500 MHz R14000 processor, Seed LIC showed immense computational cost reduction compared to Fast LIC on synthetic vector fields [S1].
*   **Strengths:** Reduces computational cost immensely for sparse input textures; allows interactive exploration; ROI preprocessing aligns boundaries with flow direction [S1, S2].
*   **Limitations:** More affected by aliasing than Fast LIC due to fewer calculated voxels, resulting in a less smooth appearance [S1, S2]. Convolution can cause field lines to merge into bundles if seed points are too close [S1, S2]. Rendering solid LIC textures as 2D slices via direct volume rendering loses inner details [S2].
*   **When to Use:** Large 3D datasets where Fast LIC is too slow; when sparse representation of flow is acceptable or desired; when ROI-based visualization is needed [S1, S2].
*   **When Not to Use:** When smooth, continuous texture coverage is required without aliasing artifacts; when computational resources are not a constraint and Fast LIC is feasible [S1, S2].

## Background
Seed LIC was proposed by Helgeland and Andreassen in 2004 as a method for effective volume visualization of three-dimensional vector fields [S1]. It builds on the Line Integral Convolution (LIC) algorithm by Stalling and Hege (Fast LIC) but optimizes it for sparsity [S1, S2]. The method was developed to address the perceptual difficulties of dense 3D displays and the high computational cost of applying LIC to large volumes [S1, S2]. It is used in scientific visualization, particularly for visualizing vorticity fields and nerve fibre structures in Diffusion Tensor Imaging (DTI) [S1, S2].

## Sources
[S1] Visualization of Vector Fields Using Seed LIC and Volume Rendering, https://www.cs.unc.edu/~taylorr/Comp715/papers/01333665.pdf, fetched 2026-10-27
[S2] Lecture notes: Visualization I Visualization of vector fields using ..., https://www.sci.utah.edu/~pascucci/classes/2009_fall/files/lec07-notes.pdf, fetched 2026-10-27
[S3] Visualizing 3D Flow - College of Science and Engineering, https://www-users.cse.umn.edu/~interran/papers/cga98.pdf, fetched 2026-10-29 (unofficial)
[S4] Sline: Seamless Line Illustration for Interactive Biomedical Visualization, https://doi.org/10.2312/vcbm.20161281, fetched 2026-10-29
