# Radon Transform

Also called: Parallel-Beam Radon Transform, Sinogram Generation

## Overview
The Radon Transform maps a 2D image into its projection profiles (sinogram) by integrating pixel values along parallel lines at various angles [S1, unofficial]. It serves as the forward operator in tomographic reconstruction, converting spatial domain data into the measurement domain required for inverse rendering and CT reconstruction [S1, unofficial]. Unlike direct back-projection, which is a reconstruction method, the Radon Transform is the projection step that generates the sinogram data structure [S1, unofficial].

## How it works
**Inputs:** A 2D input image $I$ of size $H \times W$.
**Outputs:** A sinogram $p$ of size $N_r \times N_\theta$, where $N_r$ is the number of detector positions and $N_\theta$ is the number of angles; the array of rotation angles `thetas`; and the array of detector positions `rs`.

**Data Structures:**
- `thetas`: 1D array of angles in radians.
- `rs`: 1D array of detector coordinates.
- `rotations`: 3D array of rotated images, shape $(N_\theta, H, W)$.
- `p`: 2D sinogram array, shape $(N_r, N_\theta)$.

**Parameters:**
- `thetas`: Angles from $0$ to $180^\circ$. Typical step is $1^\circ$ [S1, unofficial].
- `rs`: Linearly spaced detector positions from $-1$ to $1$, with length equal to the image height $H$ [S1, unofficial].
- `dr`: Detector spacing, calculated as the difference between consecutive detector positions [S1, unofficial].

**Steps:**
1. Generate an array of rotation angles `thetas` from $0$ to $180$ degrees with a specified step (e.g., $1^\circ$), converted to radians:
   $$ \text{thetas} = \left[ \frac{k \cdot \pi}{180} \right]_{k=0}^{179} $$
2. Define detector positions `rs` as linearly spaced values from $-1$ to $1$ with length equal to the image height $H$:
   $$ \text{rs} = \text{linspace}(-1, 1, H) $$
3. Calculate detector spacing `dr`:
   $$ dr = \text{rs}[1] - \text{rs}[0] $$
4. For each angle $\theta$ in `thetas`, rotate the input image $I$ by $\theta \cdot \frac{180}{\pi}$ degrees. Store these as `rotations` [S1, unofficial].
5. For each rotated image in `rotations`, sum the pixel values along the vertical axis (axis 0) to obtain a projection profile [S1, unofficial].
6. Multiply each projection profile by `dr` to approximate the integral [S1, unofficial].
7. Transpose the resulting array of profiles to form the sinogram $p$ with dimensions (detectors $\times$ angles) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from skimage.transform import rotate

def radon_transform(image):
    """
    Perform the Radon Transform on a 2D image.
    
    Args:
        image: 2D numpy array representing the input image.
        
    Returns:
        thetas: Array of angles in radians.
        rs: Array of detector positions.
        p: Sinogram array (detectors x angles).
        dr: Detector spacing.
    """
    # Step 1: Generate angles from 0 to 180 degrees with 1-degree step
    thetas_deg = np.arange(0, 180, 1)
    thetas = thetas_deg * np.pi / 180
    
    # Step 2: Define detector positions from -1 to 1, length = image height
    H = image.shape[0]
    rs = np.linspace(-1, 1, H)
    
    # Step 3: Calculate detector spacing
    dr = np.diff(rs)[0]
    
    # Step 4: Rotate image for each angle
    # Note: skimage.transform.rotate rotates counter-clockwise.
    # The source code uses theta * 180 / np.pi for the angle argument.
    rotations = np.array([
        rotate(image, theta * 180 / np.pi, reshape=False) 
        for theta in thetas
    ])
    
    # Step 5 & 6: Sum along vertical axis (axis 0) and multiply by dr
    # Summing along axis 0 collapses the height dimension, leaving width.
    # However, the source code transposes later, implying the sum result 
    # has shape (angles, width). The sinogram is defined as (detectors, angles).
    # The source code uses image.shape[0] for rs length, but sums along axis 0.
    # If image is HxW, sum(axis=0) results in shape (W,).
    # The source code creates p with shape (N_angles, N_detectors) before transpose?
    # Let's follow the source code exactly:
    # p = np.array([rotation.sum(axis=0) * dr for rotation in rotations]).T
    # rotation.sum(axis=0) -> shape (W,)
    # List comprehension -> shape (N_angles, W)
    # .T -> shape (W, N_angles)
    # rs has length H. If H != W, there is a dimension mismatch in the source logic 
    # unless W is treated as the detector count or H=W.
    # The source notes say "dimensions: detectors x angles".
    # If rs length is H, and p.T is (W, N_angles), then W must be the number of detectors.
    # We assume square images or that W is the detector dimension for this implementation.
    
    profiles = np.array([
        rotation.sum(axis=0) * dr 
        for rotation in rotations
    ])
    
    # Step 7: Transpose to get sinogram (detectors x angles)
    p = profiles.T
    
    return thetas, rs, p, dr
```

**Edge Cases and Pitfalls:**
- **Image Dimensions:** The source code defines `rs` length based on image height ($H$) but sums along axis 0, resulting in profiles of length $W$. The sinogram dimensions depend on $W$ being the number of detectors. If $H \neq W$, the detector positions `rs` and sinogram width may mismatch [S1, unofficial].
- **Rotation Interpolation:** Using `skimage.transform.rotate` introduces interpolation artifacts. For high-precision requirements, analytical projection methods are preferred over discrete rotation and summation [S1, unofficial].
- **Boundary Conditions:** The rotation is performed with `reshape=False` in the pseudocode to maintain image size, but the source code does not explicitly pass this. Default behavior may change image size, affecting alignment [S1, unofficial].

## Subtopics

### Filtered Back-Projection (FBP)
A reconstruction method that uses the Radon Transform's output (sinogram) to recover the original image. It applies a ramp filter in the frequency domain to the sinogram before back-projecting [S1, unofficial]. Unlike the forward Radon Transform which generates projections, FBP is the inverse operation. It is chosen when a fast, analytical reconstruction is needed from a sinogram generated by `skimage.transform.radon` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_\theta \cdot H \cdot W)$ due to rotating the image for each angle and summing pixels. Rotation is the dominant cost [S1, unofficial].
- **Memory Complexity:** $O(N_\theta \cdot H \cdot W)$ to store the rotated images array, plus $O(N_r \cdot N_\theta)$ for the sinogram [S1, unofficial].
- **Performance:** Using a $1^\circ$ step angle provides better reconstruction quality but is slower than larger steps (e.g., $5^\circ$) [S1, unofficial].
- **Strengths:** Simple to implement using standard image processing libraries; provides a direct approximation of the parallel-beam Radon Transform [S1, unofficial].
- **Limitations:** Discrete rotation and summation are approximations of the continuous integral; interpolation errors from rotation affect accuracy; not suitable for large-scale or real-time applications compared to analytical FFT-based methods [S1, unofficial].

## Background
The Radon Transform is a fundamental concept in tomography, named after Johann Radon. In the context of the provided source, it is implemented as a practical tool for CT reconstruction using Python libraries like `scikit-image` and `numpy` [S1, unofficial]. The specific implementation described is part of an educational or open-source project for tomographic reconstruction [S1, unofficial].

## Sources
[S1] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
