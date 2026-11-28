# Dye-Advection LIC
Also called: 3D Dye-Advection LIC, Dye Advection, Dye Injection with Backward LIC, Dye Smearing

## Overview
Dye-Advection LIC is a 3D extension of Line Integral Convolution that simulates the advection of colored dye within a vector field by exploiting the "smearing" nature of the LIC algorithm [S1, S2, unofficial]. It solves the problem of visualizing local flow features, such as wavefronts or specific material paths, which are difficult to observe in standard LIC due to the lack of directional correspondence between neighboring streamlines [S2, unofficial]. Unlike standard LIC, which produces symmetric textures, this method allows users to dynamically inject dye at arbitrary locations, ensuring the color propagates only in the forward flow direction [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D vector field $\mathbf{V}$; a base white noise texture $N(x,y,z)$; a dye injection mask $M(x,y,z)$ defining the spatial location and color of the injected dye; convolution length $L$ [S2, unofficial].
*   **Output:** A final rendered image $Final_{out}(x,y)$ combining the base flow structure with the advected dye [S2, unofficial].

**Data Structures:**
*   **White Noise Texture:** A 3D volume of random values, typically isotropic Gaussian or uniform noise, sampled along streamlines [S2, unofficial]. Exact layout and bit packing are not given in the sources [G].
*   **Dye Mask:** A 3D volume where non-zero values indicate the presence of dye. The value represents the color/intensity of the dye at that location [S2, unofficial]. Exact layout and bit packing are not given in the sources [G].
*   **Streamline Data:** Precomputed or on-the-fly streamline paths for each voxel, defined by their start points and integration directions [G]. Exact layout and bit packing are not given in the sources [G].

**Steps:**
1.  **Compute Base LIC:** Compute a standard LIC image $F_{out}(x,y)$ for the current view [S2, unofficial]. This is done by convolving the white noise texture $N$ along both positive and negative streamline directions for all cells [S2, unofficial].
2.  **Identify Affected Cells:** Determine the set of cells $C_{affected}$ that will receive color contributions from the dye [S2, unofficial]. A cell $c$ is in $C_{affected}$ if its backward streamline (negative direction) passes through a region where the dye mask $M$ is non-zero, within the convolution length $L$ [S2, unofficial].
    *   *Note:* Efficient identification of $C_{affected}$ is critical for performance. See [Fast Searching Algorithm for Dye Advection](12136-fast-searching-algorithm-for-dye-advection.md) for an optimized approach using a sparse directed graph of backward neighbors [S1, unofficial].
3.  **Compute Dye LIC:** For each cell $c \in C_{affected}$, compute a dye-specific LIC value $D_{out}(c)$ [S2, unofficial].
    *   Sample the dye mask $M$ along the **backward** streamline direction from cell $c$ for length $L$ [S2, unofficial].
    *   Convolve the sampled values using the standard LIC filter kernel [G].
    *   Cells not in $C_{affected}$ are assigned $D_{out}(c) = 0$ (or transparent) [S2, unofficial].
4.  **Composite:** Combine the base LIC and the dye LIC. The final output is computed as:
$$Final_{out}(x,y) = D_{out}(x,y) \circledcirc F_{out}(x,y)$$
where the operator $\circledcirc$ overwrites the pixel values of $F_{out}$ with $D_{out}$ where $D_{out}$ is non-zero [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dye_advection_lic(vector_field, noise_texture, dye_mask, convolution_length, camera):
    """
    Computes Dye-Advection LIC for a 3D vector field.
    
    Args:
        vector_field: 3D array of vectors V(x,y,z) with a sample(x,y,z) method.
        noise_texture: 3D array of noise values N(x,y,z) with a sample(x,y,z) method.
        dye_mask: 3D array of dye colors M(x,y,z) with a sample(x,y,z) method.
        convolution_length: L, the integration length for LIC.
        camera: Object with height, width, and get_ray(x, y) returning (origin, direction).
        
    Returns:
        final_image: 2D array with base LIC and advected dye.
    """
    height, width = camera.height, camera.width
    F_out = [[0.0] * width for _ in range(height)]
    D_out = [[0.0] * width for _ in range(height)]
    num_samples = 100  # Integration discretization [G]

    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = camera.get_ray(x, y)

            # 1. Compute base LIC F_out(x,y) along both directions
            samples_fwd = integrate_streamline(vector_field, ray_origin, ray_direction, convolution_length, num_samples)
            samples_bwd = integrate_streamline(vector_field, ray_origin, [-d for d in ray_direction], convolution_length, num_samples)
            # Standard LIC averages samples from both directions
            all_samples = [(s_f + s_b) / 2.0 for s_f, s_b in zip(samples_fwd, samples_bwd)]
            F_out[y][x] = convolve_1d(all_samples, convolution_length)

            # 2. Check if backward streamline passes through dye
            is_affected = any(dye_mask.sample(s[0], s[1], s[2]) != 0.0 for s in samples_bwd)

            # 3. Compute dye LIC D_out(x,y) only for affected cells, backward only
            if is_affected:
                dye_samples = [dye_mask.sample(s[0], s[1], s[2]) for s in samples_bwd]
                D_out[y][x] = convolve_1d(dye_samples, convolution_length)
            else:
                D_out[y][x] = 0.0

    # 4. Composite: overwrite F_out with D_out where D_out is non-zero
    final_image = [[0.0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            if D_out[y][x] != 0.0:
                final_image[y][x] = D_out[y][x]
            else:
                final_image[y][x] = F_out[y][x]
    return final_image

def integrate_streamline(field, origin, direction, length, num_samples):
    """
    Integrates along a streamline, sampling the field.
    """
    samples = []
    pos = list(origin)
    step = length / num_samples
    dir_len = (direction[0]**2 + direction[1]**2 + direction[2]**2)**0.5
    norm_dir = [d / dir_len for d in direction]
    
    for _ in range(num_samples):
        samples.append(field.sample(pos[0], pos[1], pos[2]))
        pos = [pos[i] + norm_dir[i] * step for i in range(3)]
    return samples

def convolve_1d(data, length):
    """
    1D convolution with a box filter kernel of size 2*L+1.
    """
    kernel_size = int(2 * length) + 1
    kernel = [1.0 / kernel_size] * kernel_size
    n = len(data)
    result = 0.0
    for i in range(n):
        for j in range(kernel_size):
            idx = i - (kernel_size // 2) + j
            if 0 <= idx < n:
                result += data[idx] * kernel[j]
    return result
```

**Parameters:**
*   **Convolution Length ($L$):** Determines the extent of the smearing effect. Typical values depend on the resolution and flow velocity, often set to a fraction of the domain size or based on the local streamline length [S2, unofficial].
*   **Dye Mask ($M$):** Defines the spatial extent and color of the injected dye [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Upstream Coloring:** If standard LIC (convolving in both directions) is applied directly to the dyed texture, the upstream of the dyed area will be incorrectly colored because positive streamlines in that area pass through the dyed area [S2, unofficial]. The solution is to convolve only in the backward direction for affected cells [S2, unofficial].
*   **Performance:** Naive identification of affected cells by scanning the entire volume is computationally expensive. Efficient algorithms are required for interactive use [S1, unofficial].

## Subtopics

### [Fast Searching Algorithm for Dye Advection](12136-fast-searching-algorithm-for-dye-advection.md)
This algorithm efficiently identifies the set of grid cells affected by a new dye injection in Dye-Advection LIC without scanning the entire volume or storing exhaustive backward streamline histories. It differs from brute-force neighbor lists by constructing a sparse directed graph of "direct flow-back neighbors" and using Breadth First Search (BFS) to locate the affected region, significantly reducing memory overhead while maintaining visual accuracy.

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the streamline integration and convolution. For each frame, standard LIC is computed for all pixels ($O(N)$ where $N$ is the number of pixels) [S1, S2, unofficial]. The dye advection step requires identifying affected cells and computing LIC only for them [S2, unofficial]. With the fast searching algorithm, the number of affected cells is proportional to the dye volume and flow velocity, making it significantly faster than recomputing LIC for the entire volume [S1, unofficial].
*   **Memory Complexity:** Requires storage for the vector field, noise texture, dye mask, and streamline data [G]. The fast searching algorithm reduces memory overhead by avoiding exhaustive neighbor lists [S1, unofficial].
*   **Strengths:** Allows dynamic, interactive injection of dye to highlight specific flow features; preserves the visual quality of standard LIC; provides clear directional information for local flow structures [S1, S2, unofficial].
*   **Limitations:** Computationally more intensive than standard LIC due to the need for two convolution passes (base and dye) and the search for affected cells [G].
*   **When to use:** When visualizing local flow features, wavefronts, or material advection in 3D vector fields is required, and interactivity is desired [S1, S2, unofficial].
*   **When not to use:** When performance is critical and dynamic dye injection is not needed; standard LIC or particle tracing may be more appropriate [S1, unofficial].

## Background
Dye-Advection LIC was proposed by Shen et al. as a 3D extension of the Line Integral Convolution method [S1, unofficial]. The method leverages the "smearing" nature of LIC to simulate the advection of dye within a flow field, allowing for the visualization of both regular LIC textures and 3D particle traces [S1, unofficial]. The technique was introduced to address the difficulty of observing correspondence between neighboring streamlines in standard LIC, which is crucial for tracking local flow features [S2, unofficial]. The method was presented in the context of flow visualization techniques for CFD using volume rendering [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
