# Radon Transform
Also called: Forward Radon Transform, Parallel-Beam Radon Transform, Sinogram Generation

## Overview
The Radon transform generates a sinogram by computing the line integrals of a 2D image's contrast along parallel rays at various angles [S2, unofficial]. It serves as the forward projection operator in tomographic reconstruction, converting spatial domain data into projection domain data [S2, unofficial]. Unlike [Filtered Back-Projection](11308-filtered-back-projection-fbp.md), which is an approximate inverse method, the Radon transform is the direct linear mapping used to acquire the measurements required for reconstruction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 2D image array $I$ of size $H \times W$.
*   Output: A sinogram array $P$ of size $N_d \times N_\theta$, where $N_d$ is the number of detectors and $N_\theta$ is the number of projection angles.

**Data Structures:**
*   `image`: 2D numpy array or tensor.
*   `theta`: 1D array of projection angles in radians.
*   `rs`: 1D array of detector positions.
*   `sinogram`: 2D array where columns correspond to angles and rows to detector positions.

**Parameters:**
*   `theta`: Projection angles. Typically calculated as `np.linspace(0.0, 180.0, max(image.shape), endpoint=False)` to provide approximately as many projections as pixels across the object [S2, unofficial].
*   `rs`: Linearly spaced detector positions. Typically from -1 to 1, with length equal to the image height (or diagonal, depending on implementation) [S1, unofficial].
*   `dr`: Detector spacing, calculated as the difference between consecutive detector positions `rs` [S1, unofficial].

**Steps:**
1.  Generate an array of rotation angles `thetas` from 0 to 180 degrees with a 1-degree step, converted to radians [S1, unofficial].
2.  For each angle in `thetas`:
    a. Rotate the input image by the current angle. The rotation is typically performed using bilinear interpolation to handle non-integer pixel coordinates [S1, unofficial].
    b. Sum the pixel values along the vertical axis (axis 0) of the rotated image to obtain the projection profile for that angle [S1, unofficial].
3.  Multiply each projection profile by the detector spacing `dr` to approximate the integral [S1, unofficial].
4.  Transpose the resulting array of projection profiles to form the sinogram $P$, such that dimensions are (detectors $\times$ angles) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import rotate

def radon_transform(image, theta_degrees=None, rs=None):
    """
    Compute the forward Radon transform (sinogram) of a 2D image.
    
    Args:
        image: 2D numpy array representing the object.
        theta_degrees: Array of angles in degrees. If None, defaults to 0-180 with 1-degree step.
        rs: Array of detector positions. If None, defaults to linearly spaced from -1 to 1 with length equal to image height.
        
    Returns:
        sinogram: 2D numpy array of shape (num_detectors, num_angles).
    """
    # Default parameters if not provided
    if theta_degrees is None:
        theta_degrees = np.arange(0, 180, 1)
    
    if rs is None:
        # Detector positions from -1 to 1, length equal to image height
        rs = np.linspace(-1, 1, image.shape[0])
        
    # Convert angles to radians
    thetas_rad = np.deg2rad(theta_degrees)
    
    num_angles = len(thetas_rad)
    num_detectors = len(rs)
    
    # Calculate detector spacing for integration approximation
    dr = rs[1] - rs[0] if num_detectors > 1 else 1.0
    
    sinogram = np.zeros((num_detectors, num_angles))
    
    for i, angle in enumerate(thetas_rad):
        # Rotate the image by the current angle
        # Note: scipy.ndimage.rotate rotates counter-clockwise. 
        # The sign of the angle may need adjustment depending on coordinate system conventions.
        rotated_image = rotate(image, angle * 180 / np.pi, reshape=False, order=1)
        
        # Sum along the vertical axis (axis 0) to get projection profile
        # This corresponds to integrating along rays perpendicular to the detector array
        projection = np.sum(rotated_image, axis=0)
        
        # Multiply by detector spacing to approximate the integral
        projection *= dr
        
        # Store in sinogram
        sinogram[:, i] = projection
        
    return sinogram
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The rotation step may introduce artifacts at the image boundaries if not padded or handled correctly. `reshape=False` in rotation preserves the original image size, potentially clipping data at corners [S1, unofficial].
*   **Precision:** Summing pixel values approximates the continuous line integral. Finer detector spacing (`dr`) improves accuracy but increases memory usage [S1, unofficial].
*   **Degenerate Input:** If the image is empty or has zero height/width, the detector spacing calculation may fail. Guard against division by zero or empty arrays [G].

## Subtopics

### [Filtered Back-Projection (FBP)](11308-filtered-back-projection-fbp.md)
An approximate algorithm for the inverse Radon transform. It is among the fastest methods for inverse Radon transform and relies on the Fourier slice theorem. Choose this when speed is critical and noise levels are low.

## Operating characteristics
*   **Time Complexity:** $O(N_\theta \cdot N_d \cdot N_p)$, where $N_\theta$ is the number of angles, $N_d$ is the number of detectors, and $N_p$ is the number of pixels in the image (for the summation step). The rotation step adds overhead proportional to the image size.
*   **Memory Complexity:** $O(N_d \cdot N_\theta)$ for the sinogram storage.
*   **Strengths:** Directly models the physical acquisition process in parallel-beam CT. Differentiable, allowing gradient-based optimization in differentiable rendering pipelines [Parent].
*   **Limitations:** Computationally expensive for high-resolution images and many angles. Sensitive to noise in the projection data, which propagates to reconstruction errors.
*   **When to use:** When implementing the forward model for differentiable tomographic reconstruction or when generating synthetic sinograms for training data.
*   **When not to use:** When real-time performance is required for large volumes, or when the acquisition geometry is not parallel-beam (e.g., cone-beam or fan-beam), which require different projection models.

## Background
The Radon transform was introduced by Johann Radon in 1917 [S2, unofficial]. It forms the mathematical foundation of computed tomography (CT), where the goal is to reconstruct a 2D image from a set of projections [S2, unofficial]. The method is widely used in medical imaging, materials science, and computer vision for tomographic reconstruction [S2, unofficial]. In the context of differentiable rendering, it is implemented as a differentiable operation to enable gradient-based optimization of scene parameters [Parent].

## Sources
[S1] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
[S2] plot_radon_transform.rst.txt, https://scikit-image.org/docs/stable/_sources/auto_examples/transform/plot_radon_transform.rst.txt, fetched 2026-10-30 (unofficial)
