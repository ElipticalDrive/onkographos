# Line Integral Convolution (LIC)

Also called: 线积分卷积

## Overview
Line Integral Convolution (LIC) is a texture-based flow visualization method that maps a scalar noise field onto a vector field by integrating the noise along streamlines, producing a streak-like texture that reveals global flow direction [S1, S2, S3, unofficial]. It differs from streamline-based methods by densely displaying direction information across the entire field without requiring user-defined seed points, though it does not inherently encode flow direction (positive/negative) or speed magnitude [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input 1:** A vector field $\mathbf{V}(x,y)$ defined on a 2D grid.
*   **Input 2:** A scalar noise texture $F_{in}$ (typically white noise) with the same resolution as the vector field [S2, S3, unofficial].
*   **Output:** A scalar field $F_{out}$ representing the convolved texture [S1, S2, unofficial].

**Data Structures:**
*   **Vector Field:** Stored as a 2D array of 2D vectors $(u, v)$ corresponding to grid points.
*   **Noise Texture:** Stored as a 2D array of scalar values.
*   **Streamline Integration:** Requires a numerical integrator (e.g., Euler or Runge-Kutta) to trace paths through the vector field.

**Algorithm Steps:**
1.  **Initialize Output:** Create an output buffer $F_{out}$ of the same dimensions as the input fields.
2.  **Iterate Pixels:** For each pixel $(x,y)$ in the output grid:
    a.  **Trace Streamlines:** Compute two streamlines starting from $(x,y)$: one in the positive direction of the vector field $\mathbf{V}$ and one in the negative direction.
    b.  **Sample Noise:** Along each streamline, sample the input noise texture $F_{in}$ at discrete points $P_i$ (positive direction) and $P'_i$ (negative direction).
    c.  **Compute Weights:** Calculate the weighting factor $h_i$ for each sample point based on the arc length $\Delta S_i$ and the convolution kernel $k(w)$.
    d.  **Aggregate:** Sum the weighted noise values and divide by the sum of weights to determine $F_{out}(x,y)$.
3.  **Post-Processing (Optional):** Apply aliasing reduction techniques such as trilinear interpolation during rendering to smooth the "stair-stepped" appearance of the field lines [S4].

**Formula:**
The output value $F_{out}(x,y)$ is calculated as [S2, unofficial]:

$$
F_{out}(x,y) = \frac{\sum_{i=0}^{n} h_i F_{in}(P_i) + \sum_{i=0}^{n'} h'_i F_{in}(P'_i)}{\sum_{i=0}^{n} h_i + \sum_{i=0}^{n'} h'_i}
$$

Where:
*   $P_0 = (x,y)$ is the starting pixel.
*   $P_i$ represents the $i$-th cell the streamline steps in the positive direction.
*   $P'_i$ represents a step in the negative direction.
*   $h_i$ and $h'_i$ are weighting variables computed from the exact integral of the convolution kernel $k(w)$ over the arc length segment $\Delta S_i$ between points $s_i$ and $s_{i+1}$ along the streamline:
    $$
    h_i = \int_{s_i}^{s_{i+1}} k(w) dw
    $$
*   $k(w)$ is the low pass filter kernel. A common choice is a Hanning ripple function with a period of $2\pi$ [S2, unofficial].
*   $s_0 = 0$.

**Parameters:**
*   **Convolution Length:** The distance along the streamline over which the integration occurs. This determines the "streak" length. [Not given in the sources.]
*   **Kernel:** The low pass filter $k(w)$. Typically a Hanning ripple function [S2, unofficial].
*   **Integration Step Size:** The step size used for tracing streamlines. [Not given in the sources.]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_integral_convolution(vector_field, noise_texture, kernel_func, max_steps):
    """
    Computes Line Integral Convolution for a 2D vector field.
    
    Args:
        vector_field: 2D array of (u, v) vectors.
        noise_texture: 2D array of scalar noise values.
        kernel_func: Function k(w) defining the low pass filter.
        max_steps: Maximum number of integration steps in each direction.
        
    Returns:
        output_texture: 2D array of convolved values.
    """
    height, width = noise_texture.shape
    output_texture = np.zeros((height, width))
    
    # Precompute kernel integrals if possible, or compute on the fly
    # For simplicity, we assume a constant step size and approximate integral
    
    for y in range(height):
        for x in range(width):
            # Start streamline at (x, y)
            pos_path = []
            neg_path = []
            
            curr_x, curr_y = float(x), float(y)
            
            # Trace positive direction
            for step in range(max_steps):
                pos_path.append((curr_x, curr_y))
                u, v = vector_field[int(curr_y), int(curr_x)]
                # Normalize vector to avoid speed dependency in path shape, 
                # though speed affects arc length weighting.
                # For standard LIC, we integrate along the flow.
                # Simple Euler integration:
                next_x = curr_x + u
                next_y = curr_y + v
                curr_x, curr_y = next_x, next_y
                if not (0 <= curr_x < width and 0 <= curr_y < height):
                    break
            
            # Trace negative direction
            curr_x, curr_y = float(x), float(y)
            for step in range(max_steps):
                neg_path.append((curr_x, curr_y))
                u, v = vector_field[int(curr_y), int(curr_x)]
                next_x = curr_x - u
                next_y = curr_y - v
                curr_x, curr_y = next_x, next_y
                if not (0 <= curr_x < width and 0 <= curr_y < height):
                    break
            
            # Compute weighted sum
            numerator = 0.0
            denominator = 0.0
            
            # Process positive path
            for i in range(len(pos_path) - 1):
                p1 = pos_path[i]
                p2 = pos_path[i+1]
                
                # Sample noise at p1 (or interpolate)
                # Using nearest neighbor for simplicity; trilinear interpolation is better [S4]
                noise_val = sample_noise(noise_texture, p1)
                
                # Calculate arc length delta_s
                dx = p2[0] - p1[0]
                dy = p2[1] - p1[1]
                delta_s = np.sqrt(dx*dx + dy*dy)
                
                # Compute weight h_i = integral of k(w) over delta_s
                # Approximation: k(0) * delta_s if kernel is centered at 0
                # More accurately, integrate k(w) from 0 to delta_s
                # Assuming k(w) is symmetric and we integrate from center:
                # h_i approx k(0) * delta_s
                weight = kernel_func(0) * delta_s 
                
                numerator += weight * noise_val
                denominator += weight
                
            # Process negative path
            for i in range(len(neg_path) - 1):
                p1 = neg_path[i]
                p2 = neg_path[i+1]
                
                noise_val = sample_noise(noise_texture, p1)
                
                dx = p2[0] - p1[0]
                dy = p2[1] - p1[1]
                delta_s = np.sqrt(dx*dx + dy*dy)
                
                weight = kernel_func(0) * delta_s
                
                numerator += weight * noise_val
                denominator += weight
            
            if denominator > 0:
                output_texture[y, x] = numerator / denominator
            else:
                output_texture[y, x] = noise_texture[y, x] # Fallback
                
    return output_texture

def sample_noise(texture, coords):
    """Simple nearest neighbor sampling. Replace with trilinear for aliasing reduction [S4]."""
    x, y = coords
    ix, iy = int(round(x)), int(round(y))
    if 0 <= iy < texture.shape[0] and 0 <= ix < texture.shape[1]:
        return texture[iy, ix]
    return 0.0
```

**Edge Cases and Pitfalls:**
*   **Direction Ambiguity:** LIC does not distinguish between positive and negative flow directions; the texture looks the same in both directions [S3, unofficial].
*   **Speed Magnitude:** LIC does not display flow speed magnitude [S3, unofficial].
*   **Aliasing:** Voxel-based field lines can exhibit a "stair-stepped" aliasing appearance. Trilinear interpolation during rendering can reduce this and create a halo effect for better line separation [S4].
*   **Periodic Motion:** To generate periodic motion effects (e.g., for animation), the phase of the convolution filter function must be shifted during convolution [S2, unofficial].

## Subtopics

### [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md)
Computes LIC only along field lines from specific seed points, reducing computational cost compared to Fast LIC by exploiting sparsity, though it is more susceptible to aliasing.

### [Image Space Fabric Visualization](10569-image-space-fabric-visualization.md)
Projects tensor data into image space and advects noise along eigenvectors on GPU FBOs for real-time fabric-like visualization of symmetric tensor fields.

### [Box Filter Dye Advection](12157-box-filter-dye-advection.md)
Replaces the Hanning filter with a box filter and introduces phase-shifting to animate dye advection along streamlines over time.

### [Variable Convolution Length for Dye](12158-variable-convolution-length-for-dye.md)
Decouples the convolution length of the dye from the global LIC texture, allowing long dye trails without degrading the underlying LIC texture's contrast or efficiency.

### [Variable Step Euler LIC](12778-variable-step-euler-lic.md)
Uses variable step Euler's method to compute curved field lines for each pixel, capturing small-scale structures like eddies better than straight-line approximations.

### Aliasing Reduction via Trilinear Interpolation
Using trilinear interpolation during rendering to reduce the 'stair-stepped' aliasing appearance of voxel-based field lines in LIC textures. This technique smooths field lines and creates a halo effect for better separation of lines, which is particularly important for sparse textures like those in Seed LIC [S4].

## Operating characteristics
*   **Time Complexity:** Standard LIC is computationally expensive, as it requires integrating streamlines for every pixel in the field [S1, S3, unofficial]. Optimized versions (e.g., Fast LIC) reduce this by reusing calculations for adjacent pixels along the same streamline [S1, unofficial].
*   **Memory Complexity:** Requires storage for the vector field, noise texture, and output texture. [Not given in the sources.]
*   **Strengths:** Densely displays global flow direction information across the entire field without requiring seed points [S3, unofficial]. Intuitively shows flow field direction with streak-like textures [S3, unofficial].
*   **Limitations:** Does not display flow direction (positive/negative) or speed magnitude [S3, unofficial]. Computationally expensive compared to optimized variants [S1, unofficial].
*   **When to Use:** When a global overview of flow direction is needed without user interaction for seed points.
*   **When Not to Use:** When flow direction (sign) or speed magnitude is critical, or when real-time performance is required without optimization techniques.

## Background
Line Integral Convolution was introduced by Cabral and Leedom [S2, unofficial]. It is a global imaging technique that creates a scalar field by convolving a white noise texture with a one-dimensional low pass filter along the streamlines of a vector field [S2, unofficial]. It has been one of the most visible flow visualization algorithms [S1, unofficial]. Improvements and extensions were proposed by Forssell and Cohen, and by Stalling and Hege [S2, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
[S3] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S4] PDF Lecture notes: Visualization I Visualization of vector fields using ..., https://www.sci.utah.edu/~pascucci/classes/2009_fall/files/lec07-notes.pdf, fetched 2026-10-27
