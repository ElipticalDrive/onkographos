# Gradient-based Occlusion Information Computation

Also called: Boundary-preserving occlusion, Perona-Malik inspired occlusion metric

## Overview
This method computes a view-independent occlusion metric for volume data that preserves sharp boundaries between structures, addressing the isotropic blurring inherent in simple spherical mean ambient occlusion [S1, unofficial]. It differs from sibling methods like [Voxel Ambient Occlusion](6646-voxel-ambient-occlusion.md) by modulating the neighborhood mean with a gradient-based weight derived from Perona-Malik diffusion principles, ensuring that edges are not smoothed out during the occlusion calculation [S1, unofficial].

## How it works
**Inputs:**
- Volume data $I$, a 3D grid of voxel intensities.
- Radius $R$, defining the neighborhood size.
- Parameter $\eta$, controlling boundary preservation strength.

**Outputs:**
- Occlusion volume $m$, where each voxel $m_x$ contains the computed occlusion information.

**Data Structures:**
- The input volume $I$ is stored as a 3D array.
- The gradient $\nabla I_x$ is computed per voxel.
- The output $m$ is stored in a 3D array of the same dimensions as $I$.

**Steps:**
1.  **Compute Gradient Magnitude:** For each voxel $x$, calculate the gradient magnitude $\|\nabla I_x\|$. This is typically done using central differences or Sobel operators on the input volume $I$.
2.  **Compute Gradient Weight:** Calculate the weight $g_x$ for each voxel using the formula:
    $$ g_x = \frac{\eta^2}{\|\nabla I_x\|^2 + \eta^2} $$
    Here, $\eta$ is a dataset-dependent parameter. Small $\eta$ (e.g., $0.001$ to $0.01$) preserves boundaries by reducing the weight in high-gradient regions. Large $\eta$ (e.g., $\ge 1$) acts as a box filter, suitable for noisy data [S1, unofficial].
3.  **Approximate Spherical Convolution:** The exact occlusion metric is defined as a spherical mean modulated by $g_x$:
    $$ m_x = g_x \cdot (B_R * I_x) $$
    where $B_R$ is a spherical filter of radius $R$ and $*$ denotes convolution [S1, unofficial]. To avoid the high complexity $O(mn)$ where $m = \frac{4}{3}\pi R^3 + 1$, the spherical filter $B_R$ is approximated by a box filter of width $2R$ [S1, unofficial].
4.  **Separable Convolution:** The 3D convolution with the box filter is separated into three consecutive 1D box filter convolutions of width $2R+1$:
    $$ m_x = g_x \cdot [b_{2R+1} * [b_{2R+1} * (b_{2R+1} * I_x)]] $$
    This separability allows for efficient GPU implementation [S1, unofficial].
5.  **Modulation:** Multiply the result of the separable convolution by the precomputed gradient weight $g_x$ to obtain the final occlusion value $m_x$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_based_occlusion(volume: Array3D, R: float, eta: float) -> Array3D:
    """
    Computes gradient-based occlusion information for a volume.
    
    Args:
        volume: 3D array of voxel intensities.
        R: Radius of the neighborhood.
        eta: Boundary preservation parameter.
        
    Returns:
        3D array of occlusion values.
    """
    # Step 1: Compute gradient magnitude
    # Using central differences for interior voxels, clamping at boundaries
    grad_x = np.zeros_like(volume)
    grad_y = np.zeros_like(volume)
    grad_z = np.zeros_like(volume)
    
    # Central differences
    grad_x[1:-1, :, :] = (volume[2:, :, :] - volume[:-2, :, :]) / 2.0
    grad_y[:, 1:-1, :] = (volume[:, 2:, :] - volume[:, :-2, :]) / 2.0
    grad_z[:, :, 1:-1] = (volume[:, :, 2:] - volume[:, :, :-2]) / 2.0
    
    # Handle boundaries with forward/backward differences or zero padding
    grad_x[0, :, :] = volume[1, :, :] - volume[0, :, :]
    grad_x[-1, :, :] = volume[-1, :, :] - volume[-2, :, :]
    grad_y[:, 0, :] = volume[:, 1, :] - volume[:, 0, :]
    grad_y[:, -1, :] = volume[:, -1, :] - volume[:, -2, :]
    grad_z[:, :, 0] = volume[:, :, 1] - volume[:, :, 0]
    grad_z[:, :, -1] = volume[:, :, -1] - volume[:, :, -2]
    
    grad_magnitude_sq = grad_x**2 + grad_y**2 + grad_z**2
    
    # Step 2: Compute gradient weight g_x
    # g_x = eta^2 / (||grad||^2 + eta^2)
    g = (eta**2) / (grad_magnitude_sq + eta**2)
    
    # Step 3 & 4: Separable box filter convolution
    # Approximate spherical filter with box filter of width 2R+1
    # Note: R is typically an integer or rounded to nearest integer for filter width
    filter_width = int(2 * R) + 1
    box_filter = np.ones(filter_width) / filter_width
    
    # Apply 1D box filter along X
    # Using convolution mode 'same' to maintain dimensions
    temp_x = convolve1d(volume, box_filter, axis=0)
    
    # Apply 1D box filter along Y
    temp_y = convolve1d(temp_x, box_filter, axis=1)
    
    # Apply 1D box filter along Z
    temp_z = convolve1d(temp_y, box_filter, axis=2)
    
    # Step 5: Modulate by gradient weight
    occlusion = g * temp_z
    
    return occlusion

def convolve1d(data: Array3D, kernel: Array1D, axis: int) -> Array3D:
    """
    Applies a 1D convolution along a specific axis.
    """
    # Pad data to handle boundaries
    pad_width = len(kernel) // 2
    padded_data = np.pad(data, ((pad_width, pad_width), (pad_width, pad_width), (pad_width, pad_width)), mode='edge')
    
    # Perform convolution
    result = np.zeros_like(data)
    if axis == 0:
        for i in range(len(kernel)):
            result += kernel[i] * padded_data[i:i+data.shape[0], :, :]
    elif axis == 1:
        for i in range(len(kernel)):
            result += kernel[i] * padded_data[:, i:i+data.shape[1], :]
    elif axis == 2:
        for i in range(len(kernel)):
            result += kernel[i] * padded_data[:, :, i:i+data.shape[2]]
            
    return result
```

**Parameters:**
- $\eta$: Dataset-dependent parameter. Typical values: $\eta \in [0.001, 0.01]$ for preserving boundaries in CT/MRI data; $\eta = 1.0$ for noisy synthetic data [S1, unofficial]. If $\eta \ge 1$, the filter behaves like a box filter [S1, unofficial].
- $R$: Radius of the spherical neighborhood. Typical value: $R=40$ for all datasets in the source results [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Isotropic Blurring:** Simple ambient occlusion (weighted sample mean of spherical neighborhood) results in isotropic blurring that does not preserve boundaries [S1, unofficial]. This method mitigates that by using $g_x$.
- **Complexity:** The naive spherical convolution has complexity $O(mn)$ where $m = \frac{4}{3}\pi R^3 + 1$, which is infeasible for large $R$ [S1, unofficial]. The separable box filter approximation is necessary for performance.
- **Boundary Conditions:** Gradient computation at volume boundaries requires careful handling (e.g., zero-padding, replication, or clamping) to avoid artifacts.

## Operating characteristics
- **Time Complexity:** The naive spherical convolution is $O(mn)$ where $m = \frac{4}{3}\pi R^3 + 1$, which is very costly [S1, unofficial]. The separable box filter approximation reduces this significantly, allowing computation within seconds on modern GPUs [S1, unofficial].
- **Memory Complexity:** Requires storage for the input volume, gradient volume, weight volume, and intermediate convolution results. Typically $O(N)$ where $N$ is the number of voxels.
- **Performance:** The method is view-independent and can be precomputed and stored, not affecting visualization speed [S1, unofficial].
- **Strengths:** Preserves boundaries between structures, avoiding the isotropic blurring of simple ambient occlusion [S1, unofficial]. Efficient via separable convolution.
- **Limitations:** Requires tuning of $\eta$ and $R$. The box filter approximation may introduce slight artifacts compared to a true spherical filter, but this is negligible for large $R$ [S1, unofficial].

## Background
This method was introduced in the context of multivariate transfer function design for volume visualization, specifically to improve the statistical feature space construction by providing a boundary-preserving occlusion metric [S1, unofficial]. It is inspired by the Perona-Malik anisotropic diffusion equation, which is known for edge-preserving smoothing [S1, unofficial]. The method is used in conjunction with statistical transfer function spaces to classify volume data, such as tumors in MRI datasets [S1, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
