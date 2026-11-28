# Three-Dimensional LIC

Also called: 3D LIC, Volume Line Integral Convolution, Volumetric LIC

## Overview
Three-Dimensional Line Integral Convolution (3D LIC) generalizes the 2D LIC algorithm to three dimensions, applying a noise texture along streamlines within a 3D vector field to produce a 3D scalar output texture where voxel values are locally correlated with the flow direction [S1][S2, unofficial]. It differs from its siblings by operating on volumetric data rather than surfaces or 2D slices, allowing for the visualization of internal flow structures, though it suffers from higher computational cost and perceptual difficulties in conveying directionality from 2D projections [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input Vector Field:** A 3D vector field $\mathbf{V}(x,y,z)$.
*   **Input Texture:** A 3D scalar noise texture $T(x,y,z)$.
*   **Output:** A 3D scalar field $O(x,y,z)$.

**Data Structures:**
The algorithm processes the volume on a per-cell basis. Each cell is defined by its spatial coordinates and the vector field value at that location. The noise texture is sampled along the streamline passing through the cell center.

**Steps:**
1.  **Initialize Output:** Create a 3D output scalar field $O$ of the same dimensions as the input vector field and texture.
2.  **Iterate Cells:** For each cell $(i,j,k)$ in the 3D grid:
    a.  **Sample Vector:** Retrieve the vector $\mathbf{v} = \mathbf{V}(i,j,k)$.
    b.  **Check Magnitude Threshold:** If $|\mathbf{v}|$ is outside the volume renderer's min/max threshold window, set $O(i,j,k)$ to transparent/zero and skip further processing for this cell [S1].
    c.  **Integrate Streamline:** Compute the streamline passing through the center of the cell $(i,j,k)$ by integrating the vector field $\mathbf{V}$ (e.g., using Euler or Runge-Kutta methods) to generate a path $\mathbf{r}(t)$.
    d.  **Convolve Noise:** Sample the input noise texture $T$ along the streamline $\mathbf{r}(t)$ and convolve these samples with a kernel (typically a box filter or Gaussian) to compute the output value $O(i,j,k)$ [S1].
3.  **Render:** Render the resulting 3D scalar field $O$ using volume rendering techniques, such as ray casting with opacity transfer functions [S1].

**Formulas:**
The 3D algorithm extends the 1D LIC equations trivially. For a cell at position $\mathbf{x}$, the output is:
$$ O(\mathbf{x}) = \int_{-\infty}^{\infty} T(\mathbf{r}(t)) \cdot h(t) \, dt $$
where $\mathbf{r}(t)$ is the streamline position at parameter $t$, and $h(t)$ is the convolution kernel [S1].

**Parameters:**
*   **Convolution Length:** Not specified in sources.
*   **Integration Step Size:** Not specified in sources.
*   **Threshold Window:** Defined by the volume renderer's min/max magnitude thresholds [S1].

**Edge Cases and Pitfalls:**
*   **Perceptual Difficulty:** Dense 3D displays suffer from difficulties in clearly conveying directional information and mental reconstruction of 3D flow from 2D slices [S2, unofficial].
*   **Performance:** The algorithm is computationally expensive. Without thresholding, performance drops significantly compared to 2D LIC [S1].
*   **Boundary Conditions:** Not specified in sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_d_lic(vector_field, noise_texture, output_volume, threshold_min, threshold_max):
    """
    Computes 3D Line Integral Convolution.
    
    Args:
        vector_field: 3D array of vectors V(x,y,z).
        noise_texture: 3D array of scalar noise T(x,y,z).
        output_volume: 3D array to store results O(x,y,z).
        threshold_min: Minimum vector magnitude to process.
        threshold_max: Maximum vector magnitude to process.
    """
    # Iterate over all cells in the volume
    for x, y, z in grid_coordinates(vector_field):
        # Get vector at current cell
        v = vector_field[x, y, z]
        magnitude = length(v)
        
        # Check magnitude threshold
        if magnitude < threshold_min or magnitude > threshold_max:
            output_volume[x, y, z] = 0.0
            continue
            
        # Integrate streamline through the cell center
        # Note: Integration method (Euler, RK4) and step size are implementation details
        # not specified in the source. We assume a function that returns a path.
        path = integrate_streamline(vector_field, start_point=(x, y, z))
        
        # Sample noise along the path and convolve
        # Note: The specific kernel and sampling strategy are not detailed in the source.
        # We assume a standard convolution operation.
        samples = []
        for point in path:
            samples.append(noise_texture[point.x, point.y, point.z])
            
        output_volume[x, y, z] = convolve(samples)

    return output_volume
```

## Subtopics

### Seed Line Integral Convolution
Generates LIC using seeds rather than dense sampling, reducing computational cost. [Link](8864-seed-line-integral-convolution.md)

### Image Space Fabric Visualization
Visualizes flow using fabric-like structures in image space, avoiding volume rendering. [Link](10569-image-space-fabric-visualization.md)

### Variable Convolution Length for Dye
Adjusts convolution length based on flow characteristics to improve visual clarity. [Link](12158-variable-convolution-length-for-dye.md)

### Variable Step Euler LIC
Uses adaptive step sizes during integration to improve accuracy and performance. [Link](12778-variable-step-euler-lic.md)

### Haloed Line Integral Convolution
Adds halos to LIC patterns to enhance depth perception. [Link](20040-haloed-line-integral-convolution.md)

### Filtering and Aliasing Reduction
Techniques to reduce aliasing artifacts in LIC results. [Link](20171-filtering-and-aliasing-reduction.md)

## Operating characteristics
*   **Time Complexity:** The algorithm remains one-dimensional in nature, so the cost per cell increases by a factor of three due to dimensionality [S1].
*   **Memory Complexity:** Requires storage for the input vector field, input noise texture, and output scalar field [S1][S2, unofficial].
*   **Performance:** On an IBM 550 RISC 6000, 3D LIC processes approximately 1,200 cells per second (CPS) without optimization [S1]. Using magnitude thresholding to skip cells outside the volume renderer's min/max window, performance exceeds 30,000 CPS [S1]. This is roughly an order of magnitude slower than the DDA method (30,000 CPS) in 2D, but comparable when thresholding is applied in 3D [S1].
*   **Strengths:** Allows visualization of internal flow structures within a volume [S2, unofficial].
*   **Limitations:** High computational cost; perceptual difficulties in conveying directionality from 2D slices; dense 3D displays are hard to interpret [S2, unofficial].
*   **When to use:** When internal flow structures need to be visualized and computational resources allow for volume rendering [S2, unofficial].
*   **When not to use:** When real-time performance is required or when the flow is primarily surface-based [S2, unofficial].

## Background
3D LIC was introduced as a generalization of the 2D LIC algorithm by [S1]. It was developed to address the need for visualizing 3D flow fields within volumes, extending the concept of texture correlation along streamlines to three dimensions [S1][S2, unofficial]. The method has been used in applications such as rendering electrostatic fields [S1].

## Sources
[S1] Imaging vector fields using Line Integral Convolution, https://digital.library.unt.edu/ark:/67531/metadc1399414/?q=vector field volume rendering, fetched 2026-10-28
[S2] Visualizing 3D Flow - College of Science and Engineering, https://www-users.cse.umn.edu/~interran/papers/cga98.pdf, fetched 2026-10-29 (unofficial)
