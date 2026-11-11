# Seed Line Integral Convolution

Also called: Seed LIC, Volume LIC

## Overview
Seed Line Integral Convolution (Seed LIC) is a volume rendering technique for visualizing 3D vector fields that computes Line Integral Convolution (LIC) only along field lines starting from specific seed points, rather than filling the entire volume [S1, S2]. It solves the computational bottleneck of standard Fast LIC [7940-line-integral-convolution-lic.md] by exploiting the sparsity of the input texture, calculating field lines and convolution only from distributed seed points instead of every voxel [S1, S2]. Unlike Fast LIC, which assigns values to neighboring voxels causing smearing, Seed LIC only assigns values to voxels along field lines from seed points, resulting in sparser textures but significantly reduced computational cost [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D vector field $\mathbf{v}$, a scalar field defining a Region of Interest (ROI) or random distribution criteria, and parameters for seed point generation (density factor, threshold, minimum distance).
- **Output:** A 3D scalar texture representing the LIC result, where voxel values indicate the integrated noise along streamlines.

**Data Structures:**
- **Input Texture ($T$):** A 3D grid of voxels. Initially, all voxels are set to 0. Voxels containing seed points are set to 255 (or a non-zero value).
- **Output Texture ($I$):** A 3D grid of voxels, initialized to 0.
- **Field Line State:** Current position $\mathbf{x}$, integration step $m$, and accumulated convolution value.

**Parameters:**
- **Density Factor ($d$):** A value in $[0, 1]$ controlling the sparsity of the input texture [S1, S2].
- **Threshold Value:** A scalar value threshold used to determine if a voxel contributes to the input texture based on the ROI [S2].
- **Minimum Distance:** The minimum distance between seed points to prevent merging. Sources specify discarding points within three voxels of previously inserted seeds [S1, S2].
- **Filter Kernel Length ($2L$):** The length of the convolution kernel. In reported experiments, set to 21 times the grid cell size [S1].
- **Field Line Length:** The total length of the field line integration. In reported experiments, set to 51 times the size of the grid cells [S1].
- **Integration Limit ($M$):** The maximum number of steps for the while loop in the convolution [S2].

**Algorithm Steps:**

1.  **Generate Sparse Input Texture:**
    a. Initialize all voxels in the input texture $T$ to 0 [S1, S2].
    b. Iterate over each voxel $v$ in the grid [S2]:
        i. If the scalar value at $v$ (e.g., vorticity magnitude) is greater than the threshold value [S2]:
            1. Compute a random number $r$ in $[0, 1]$ [S2].
            2. If $r > \text{density factor}$, mark $v$ as a seed point by setting $T(v) = 255$ [S1, S2].
            3. Ensure a minimum distance between seed points. Discard points that fall within three voxels of any previously inserted seed point [S1, S2].
    c. All other voxels remain 0 [S1, S2].

2.  **Initialize Output Texture:**
    a. Set all voxels in the output texture $I$ to 0 [S1, S2].

3.  **Compute LIC from Seed Points:**
    a. Iterate over each voxel $v$ in the grid [S2]:
        i. If $T(v) > 0$ (i.e., $v$ is a seed point):
            1. Let $\mathbf{x}_0$ be the center of voxel $v$ [S2].
            2. Initialize the convolution integral $I(\mathbf{x}_0)$ [S1].
            3. Trace the field line starting from $\mathbf{x}_0$ using a fifth-order adaptive Runge-Kutta method with a fixed sample interval [S1, S2].
            4. Compute the convolution integral along the field line. The intensity $I(\mathbf{x}_0)$ for a voxel located at $\mathbf{x}_0 = \alpha(s_0)$ is given by:
               $$ I(\mathbf{x}_0) = \int_{s_0-L}^{s_0+L} k(s-s_0) T(\alpha(s)) ds $$
               where $k$ is the filter kernel of length $2L$, $T$ is the input texture, and $\alpha(s)$ is the curve parameterized by arc-length $s$ [S1].
            5. Add the result to the voxel $v$ in the output texture [S2].
            6. Propagate the convolution along the field line in both directions (upstream and downstream) for $m = 1$ to $M$ [S2]:
                i. Update the convolution values $I(\mathbf{x}_m)$ and $I(\mathbf{x}_{-m})$ at positions $\mathbf{x}_m$ and $\mathbf{x}_{-m}$ along the field line [S2].
                ii. Add the results to the voxels containing $\mathbf{x}_m$ and $\mathbf{x}_{-m}$ in the output texture [S2].
                iii. Increment $m$ [S2].

4.  **Normalize Output:**
    a. Normalize the intensity of the voxels in the output texture so that values vary from 0 to 255 [S1, S2].
    b. This can be done by normalizing according to the number of hits ($numHits(v)$) per voxel and then scaling so the highest value is 255 [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def seed_lic(vector_field, scalar_field, density_factor, threshold, min_dist_voxels, kernel_length_cells, line_length_cells):
    # 1. Generate Sparse Input Texture
    input_texture = zeros_like(vector_field) # Initialize all voxels to 0
    seed_points = []
    
    # Determine potential seed locations based on ROI and density
    for v in all_voxels:
        if scalar_field[v] > threshold:
            r = random_uniform(0, 1)
            if r > density_factor:
                # Check minimum distance from existing seeds
                if not has_nearby_seed(v, seed_points, min_dist_voxels):
                    input_texture[v] = 255
                    seed_points.append(v)
    
    # 2. Initialize Output Texture
    output_texture = zeros_like(vector_field)
    
    # 3. Compute LIC from Seed Points
    for v in seed_points:
        x0 = center_of_voxel(v)
        
        # Trace field line using 5th-order adaptive Runge-Kutta
        # Returns a list of points [x_-M, ..., x_0, ..., x_M]
        field_line = trace_field_line(x0, vector_field, line_length_cells)
        
        # Compute convolution integral for the seed point
        # I(x0) = integral from s0-L to s0+L of k(s-s0) * T(alpha(s)) ds
        conv_val_0 = compute_convolution_integral(x0, field_line, input_texture, kernel_length_cells)
        
        # Add result to the seed voxel
        output_texture[v] += conv_val_0
        
        # Propagate along the field line
        for m in range(1, M + 1):
            xm = field_line[m]
            x_minus_m = field_line[-m]
            
            # Compute convolution at xm and x_-m
            conv_val_m = compute_convolution_integral(xm, field_line, input_texture, kernel_length_cells)
            conv_val_minus_m = compute_convolution_integral(x_minus_m, field_line, input_texture, kernel_length_cells)
            
            # Add results to the corresponding voxels
            output_texture[voxel_containing(xm)] += conv_val_m
            output_texture[voxel_containing(x_minus_m)] += conv_val_minus_m
            
    # 4. Normalize Output
    max_val = max(output_texture)
    if max_val > 0:
        output_texture = (output_texture / max_val) * 255
        
    return output_texture

def has_nearby_seed(v, seed_points, min_dist_voxels):
    for seed in seed_points:
        if distance(v, seed) <= min_dist_voxels:
            return True
    return False

def trace_field_line(x0, vector_field, length_cells):
    # Implementation of 5th-order adaptive Runge-Kutta
    # Returns points spaced by arc-length s
    pass

def compute_convolution_integral(x, field_line, input_texture, kernel_length_cells):
    # Discrete approximation of the integral
    # Sum k(s-s0) * T(alpha(s)) over the kernel length
    pass
```

## Subtopics

### Sparse Volume LIC
A variant that adjusts the size of the input texture to be larger than the vector field (detail enlargement) to produce smoother and more accurate strokes [S2]. To avoid aliasing during this enlargement, the step size between sample points when computing convolution integrals is reduced inversely proportional to the texture size increase [S2].

### Aliasing Reduction via 3x3x3 Convolution
A post-processing step to reduce aliasing in Seed LIC textures, which are more affected by aliasing than Fast LIC due to fewer calculated voxels [S1, S2]. The output texture is convolved with a 3x3x3 filter where the center entry $a_{222} = 1$ and the rest of the entries are set to 0.25 [S2]. This smears field lines to create smoother strokes but leads to thicker lines; to preserve thickness, the resolution of input and output textures should be increased by a factor of three prior to convolution [S2].

## Operating characteristics
- **Time Complexity:** Significantly faster than Fast LIC for sparse input textures. For a 398x298x798 texture, Fast LIC took about 25 hours, while Seed LIC took 26.86 seconds [S1, S2]. For a 128x128x128 texture, Fast LIC took 752 seconds while Seed LIC took 4.7 seconds [S2]. For a 256x256x256 texture with 17341 seed points, Seed LIC took 70.9 seconds, compared to 752 seconds for Fast LIC [S2].
- **Memory Complexity:** Requires storage for the input texture (sparse), output texture, and field line data. The sparsity of the input texture reduces the number of field lines computed, lowering memory access patterns compared to Fast LIC.
- **Strengths:** Drastically reduces computational cost for large 3D datasets; allows interactive exploration; exploits sparsity to avoid unnecessary computations in areas where field lines do not coincide with seed points [S1, S2].
- **Limitations:** More affected by aliasing than Fast LIC, resulting in a less smooth appearance [S1, S2]. The output is dependent on the placement of seed points; if seed points are too close, field lines may merge into bundles [S1, S2]. Rendering solid LIC textures as 2D slices via direct volume rendering loses inner details [S2].
- **When to use:** Large 3D datasets where computational efficiency is critical and the vector field has sparse regions of interest.
- **When not to use:** When a smooth, continuous texture is required without post-processing, or when the vector field is dense and uniform, making the sparsity advantage negligible.

## Background
Seed LIC was proposed by Helgeland and Andreassen in 2004 [S1]. It builds on the Fast LIC method by Stalling and Hege [S1, S2]. The method was developed to address the high computational cost of Fast LIC for large 3D datasets, such as those encountered in fluid dynamics simulations (e.g., shock waves, vorticity fields) [S1, S2]. It is used in scientific visualization to provide effective volume visualization of three-dimensional vector fields, often combined with techniques like limb darkening to enhance depth perception [S1, S2].

## Sources
[S1] Visualization of Vector Fields Using Seed LIC and Volume Rendering, https://www.cs.unc.edu/~taylorr/Comp715/papers/01333665.pdf, fetched 2026-10-27
[S2] Lecture notes: Visualization I Visualization of vector fields using ..., https://www.sci.utah.edu/~pascucci/classes/2009_fall/files/lec07-notes.pdf, fetched 2026-10-27
