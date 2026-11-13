# Boundary-structure-aware transfer function


## Overview
Boundary-structure-aware transfer functions classify volume data by combining boundary-based and structure-based advantages, specifically addressing the limitation where standard ambient occlusion fails to distinguish objects with identical average intensities but different global structural variations [S1]. Unlike sibling methods that rely on local gradients or simple occlusion sums, this method uses the **standard deviation of ambient occlusion** ($O_{std}$) computed over a global field to separate materials that share similar intensity profiles but differ in their spatial arrangement or internal continuity [S1]. It constructs a 2D classification space mapping intensity (or second derivatives) against $O_{std}$, enabling robust extraction of structures with continuous intensity changes or imperfect homogeneity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume field $f(\mathbf{x})$.
*   **Output:** Opacity $\alpha \in [0, 1]$ and color $C$ for each voxel, derived from a 2D histogram classification space.

**Data Structures:**
1.  **Input Field:** The original scalar volume $f(\mathbf{x})$.
2.  **Processed Field ($\vec{d}$):** A derived field used for occlusion calculation. Typically $\vec{d} = f''(\mathbf{x})$ (the divergence of the gradient, i.e., Laplacian) to highlight boundaries and suppress interiors, or $f(\mathbf{x})$ for simple cases [S1].
3.  **Directional Occlusion Field:** A temporary structure storing $A(\mathbf{p}, \hat{\omega}_i, \vec{d})$ for each direction $i$ [S1].
4.  **2D Histogram:** A binning structure with axes $X$ (intensity $f(\mathbf{x})$ or $f''(\mathbf{x})$) and $Y$ ($O_{std}(\vec{d})$) [S1].
5.  **Transfer Function Mappings:** Three 1D mappings (or a 2D lookup table) defining opacity and color based on regions in the 2D histogram [S1].

**Parameters:**
*   $T$: Number of samples along each direction $\hat{\omega}$ for computing directional occlusion [S1].
*   $H$: Number of different directions sampled to compute the standard deviation [S1].
*   $M(\mathbf{p}, \vec{d})$: Visibility mapping function. Application-specific; e.g., binary (1 if $\mathbf{p}$ is part of the isosurface, 0 otherwise), Gaussian weighted, or linear ramp [S1].

**Algorithm Steps:**

1.  **Preprocessing (Optional but Recommended):**
    Compute the second derivative (Laplacian) of the input field to suppress continuous intensity variations and highlight boundaries.
    $$ \vec{d}(\mathbf{x}) = f''(\mathbf{x}) = \nabla \cdot \nabla f(\mathbf{x}) $$
    [S1]
    *Note: If the volume consists of simple objects with constant intensities, use $\vec{d}(\mathbf{x}) = f(\mathbf{x})$ instead [S1].*

2.  **Compute Directional Occlusion:**
    For each voxel $\mathbf{p}$ and each of the $H$ discrete directions $\hat{\omega}_i$ (where $i = 1 \dots H$), compute the directional occlusion $A(\mathbf{p}, \hat{\omega}_i, \vec{d})$. This is approximated by summing the visibility mapping along the ray:
    $$ A(\mathbf{p}, \hat{\omega}_i, \vec{d}) \approx \sum_{t=0}^{T} M(\mathbf{p} + t \hat{\omega}_i, \vec{d}) $$
    [S1]
    Here, $T$ is the number of samples along the direction. The visibility function $M$ determines which parts of the volume contribute to occlusion (e.g., $M=1$ for opaque voxels, $M=0$ for transparent).

3.  **Compute Standard Deviation of Ambient Occlusion:**
    For each voxel $\mathbf{p}$, calculate the standard deviation of the $H$ directional occlusion values computed in Step 2. This quantifies the variation in structural surroundings:
    $$ O_{std}(\mathbf{p}, \vec{d}) = \sqrt{ \frac{1}{H} \sum_{i=1}^{H} \left( A(\mathbf{p}, \hat{\omega}_i, \vec{d}) \right)^2 - \left( \frac{1}{H} \sum_{i=1}^{H} A(\mathbf{p}, \hat{\omega}_i, \vec{d}) \right)^2 } $$
    [S1]
    This value $O_{std}$ captures global structure information. Objects with the same intensity but different locations or shapes will have different directional occlusion patterns, resulting in distinct $O_{std}$ values, whereas standard ambient occlusion (the mean) would be identical [S1].

4.  **Construct 2D Classification Space:**
    Create a 2D histogram where:
    *   Horizontal axis: Intensity $f(\mathbf{x})$ or processed intensity $f''(\mathbf{x})$.
    *   Vertical axis: Standard deviation of ambient occlusion $O_{std}(\vec{d})$.
    [S1]

5.  **Tag Regions and Assign Properties:**
    Identify regions in the 2D histogram corresponding to features of interest. Map these regions to opacity and color. The method supports three 1D mappings for fine control:
    *   Mapping based on $f(\mathbf{x})$ or $f''(\mathbf{x})$.
    *   Mapping based on $O_{std}(\vec{d})$.
    *   Combined mapping: $f(\mathbf{x}) \times O_{std}(f''(\mathbf{x})) \mapsto [0, 1]$ or $f''(\mathbf{x}) \times O_{std}(f''(\mathbf{x})) \mapsto [0, 1]$ for opacity/color [S1].
    *   *Implementation Note:* Users can "brush" regions in the 2D histogram to select specific structures. Color can be assigned based on the divergence $f''(\mathbf{x})$ to help distinguish boundaries [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_boundary_structure_aware_tf(volume_f, visibility_func, num_directions=12, samples_per_dir=10):
    """
    Computes the boundary-structure-aware transfer function values for a volume.
    
    Args:
        volume_f: 3D numpy array of scalar values f(x).
        visibility_func: Function M(p, d) that returns 1 if voxel p contributes to occlusion, 0 otherwise.
                         Note: In practice, this is often a threshold or Gaussian filter applied to d.
        num_directions: H, number of directions to sample.
        samples_per_dir: T, number of samples along each ray.
    
    Returns:
        opacity_map: 3D array of opacity values.
        color_map: 3D array of RGB color values.
    """
    
    # Step 1: Preprocessing
    # Compute Laplacian f''(x) to highlight boundaries.
    # Using a simple finite difference approximation for Laplacian.
    f_double_prime = compute_laplacian(volume_f)
    
    # Decide input field d. 
    # For complex structures with continuous changes, use f''. 
    # For simple constant-intensity objects, use volume_f.
    # Here we assume the general case using f'' as per the primary description.
    d = f_double_prime
    
    # Step 2 & 3: Compute Standard Deviation of Ambient Occlusion
    # Initialize arrays
    H = num_directions
    T = samples_per_dir
    
    # Generate H directions (e.g., Fibonacci sphere or uniform sampling)
    directions = generate_directions(H)
    
    # Array to store directional occlusions for each voxel
    # Shape: (X, Y, Z, H)
    directional_occlusions = np.zeros((*volume_f.shape, H))
    
    for i, omega in enumerate(directions):
        # Compute directional occlusion for all voxels along direction omega
        # A(p, omega, d) = sum_{t=0}^{T} M(p + t*omega, d)
        # We iterate through voxels and sum visibility along the ray
        
        # Optimization: Vectorized ray casting or iterative accumulation
        # For clarity, we show the logic per voxel
        for x in range(volume_f.shape[0]):
            for y in range(volume_f.shape[1]):
                for z in range(volume_f.shape[2]):
                    occlusion_sum = 0.0
                    current_pos = np.array([x, y, z])
                    for t in range(T):
                        # Sample position
                        sample_pos = current_pos + t * omega
                        # Check bounds
                        if is_inside_volume(sample_pos, volume_f.shape):
                            # Get value d at sample position
                            d_val = get_value_at(d, sample_pos)
                            # Apply visibility function M
                            # M is application specific. Example: binary threshold on d
                            m_val = visibility_func(sample_pos, d_val)
                            occlusion_sum += m_val
                    directional_occlusions[x, y, z, i] = occlusion_sum

    # Compute Mean and Mean of Squares for Standard Deviation
    mean_occlusion = np.mean(directional_occlusions, axis=3)
    mean_sq_occlusion = np.mean(np.square(directional_occlusions), axis=3)
    
    # O_std = sqrt(E[A^2] - (E[A])^2)
    o_std = np.sqrt(mean_sq_occlusion - np.square(mean_occlusion))
    
    # Step 4 & 5: Classification and Mapping
    # Construct 2D histogram bins (conceptually)
    # Map (f''(x), o_std) to opacity and color
    
    # Example mapping logic:
    # Define thresholds or regions in the 2D space of (f_double_prime, o_std)
    # This part is user-defined via "brushing" in the original paper.
    # Here we provide a generic implementation of a region-based assignment.
    
    opacity_map = np.zeros(volume_f.shape)
    color_map = np.zeros((*volume_f.shape, 3))
    
    # Example: Select regions based on specific criteria
    # Region 1: High boundary strength (high |f''|) and specific structural variation
    mask1 = (np.abs(f_double_prime) > threshold_boundary) & (o_std > threshold_struct)
    
    # Assign properties based on the mask
    # Opacity can be a function of f'' and o_std
    opacity_map[mask1] = 1.0 
    color_map[mask1] = [1.0, 0.0, 0.0] # Red
    
    # Region 2: Low boundary strength, low structural variation (interiors)
    mask2 = (np.abs(f_double_prime) < threshold_interior) & (o_std < threshold_interior_struct)
    opacity_map[mask2] = 0.5
    color_map[mask2] = [0.0, 1.0, 0.0] # Green
    
    return opacity_map, color_map

def compute_laplacian(volume):
    """Simple 3D Laplacian kernel convolution."""
    kernel = np.zeros((3, 3, 3))
    kernel[1, 1, 1] = -6
    kernel[0, 1, 1] = 1
    kernel[2, 1, 1] = 1
    kernel[1, 0, 1] = 1
    kernel[1, 2, 1] = 1
    kernel[1, 1, 0] = 1
    kernel[1, 1, 2] = 1
    return np.convolve(volume, kernel, mode='same') # Placeholder for actual convolution

def generate_directions(n):
    """Generate n unit vectors uniformly distributed on a sphere."""
    # Fibonacci sphere distribution
    indices = np.arange(0, n, dtype=float) + 0.5
    Y = 1 - (indices / n) * 2
    Radius = np.sqrt(1 - Y * Y)
    Theta = np.pi * (1 + np.sqrt(5)) * indices
    return np.vstack((np.cos(Theta) * Radius, Y, np.sin(Theta) * Radius)).T

def is_inside_volume(pos, shape):
    return (0 <= pos[0] < shape[0] and 
            0 <= pos[1] < shape[1] and 
            0 <= pos[2] < shape[2])

def get_value_at(array, pos):
    # Bilinear/Bicubic interpolation would go here for non-integer positions
    x, y, z = pos
    ix, iy, iz = int(x), int(y), int(z)
    # Simple nearest neighbor for brevity
    return array[ix, iy, iz]
```

**Edge Cases and Pitfalls:**
*   **Global vs. Local Filter Size:** Standard ambient occlusion (Correa et al.) uses a local filter and fails to distinguish objects with the same intensity and local surroundings but different global locations (e.g., two balls in a box) [S1]. This method uses the whole domain (global filter) to compute $O_{std}$, solving this issue [S1].
*   **Continuous Intensity Changes:** Kindlmann et al.'s method fails on non-Gaussian boundaries or varying intensities [S1]. By using $f''(\mathbf{x})$ to suppress continuous changes into a constant value before computing $O_{std}$, this method handles varying intensities robustly [S1].
*   **Simple Volumes:** If the volume contains only a few objects with constant intensities, using $f''(\mathbf{x})$ may result in most values being near zero, losing information. In such cases, using $f(\mathbf{x})$ as the input field $\vec{d}$ is recommended [S1].
*   **Visibility Function $M$:** The choice of $M$ is critical and application-specific. A binary function isolates isosurfaces, while Gaussian weights preserve soft boundaries [S1].

## Operating characteristics
*   **Complexity:** The method requires computing directional occlusion for $H$ directions and $T$ samples per voxel. Complexity is $O(N \cdot H \cdot T)$ where $N$ is the number of voxels. This is significantly more expensive than standard gradient-based methods.
*   **Memory:** Requires storing the volume, the Laplacian field, and the directional occlusion array ($N \times H$). For large $H$, memory usage is high.
*   **Performance:** Not given in the sources. However, the global nature of the occlusion calculation (summing over the whole domain or large radii) suggests high computational cost compared to local filters.
*   **Strengths:**
    *   Separates objects with identical intensities and local surroundings but different global structures or locations [S1].
    *   Handles structures with continuous intensity changes (non-homogeneous interiors) better than boundary-only methods [S1].
    *   Robust against overlapping occlusion spectra that confuse standard ambient occlusion [S1].
*   **Limitations:**
    *   Computationally intensive due to global ray casting for occlusion.
    *   Requires careful tuning of the visibility function $M$ and the choice between using $f(\mathbf{x})$ or $f''(\mathbf{x})$ as the input field [S1].
    *   Sensitive to the number of directions $H$ and samples $T$; insufficient sampling may lead to inaccurate $O_{std}$ estimates [S1].

## Background
*   **Origin:** Proposed by Lina Yu and Hongfeng Yu from the University of Nebraska-Lincoln [S1].
*   **Publication:** Presented at the Symposium on Visualization (SA '17) in Bangkok, Thailand, November 27-30, 2017 [S1].
*   **Context:** The method addresses limitations in existing boundary-based methods (like Kindlmann et al.) and structure-based methods (like Correa et al.). Correa et al.'s method uses the sum of ambient occlusion $O(f(\mathbf{x}))$, which fails to distinguish objects with similar intensities but different surroundings or locations. Kindlmann et al.'s method uses derivatives but fails on non-Gaussian boundaries. This method bridges these gaps by using the standard deviation of ambient occlusion on a processed field [S1].
*   **Applications:** Demonstrated on MRI scans of feet (separating marrow from muscle/bone) and CT scans of engine data (separating stick-shape holes) and combustion simulations (extracting holes on flame isosurfaces) [S1].

## Sources

[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
