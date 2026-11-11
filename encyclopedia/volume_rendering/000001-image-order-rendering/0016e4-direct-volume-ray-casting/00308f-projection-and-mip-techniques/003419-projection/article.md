# Projection

Also called: Direct Volume Rendering via Projection, Orthographic Projection (Summation), Summation Projection, Summed Reprojection, Summed projection, Total Intensity Projection, Voxel Projection

## Overview
Projection (specifically Summation Projection) is a volume rendering technique that projects a 3D volume onto a 2D plane by summing the voxel values along the projection axis [S1, unofficial]. It visualizes the total accumulated intensity along rays, effectively creating a 2D map of the volume's total density or signal [S1, S3, unofficial]. Unlike [Maximum Intensity Projection](902-maximum-intensity-projection.md) which highlights the brightest structures, Projection aggregates all contributions, which decreases image noise but significantly reduces image contrast due to the cumulative effect of background activity [S3, unofficial]. It is distinct from [Direct volume ray casting](19-direct-volume-ray-casting.md) by using simple addition as the aggregation operator instead of front-to-back compositing with opacity [Parent].

## How it works
The method follows the shared principles of the family [Parent], with specific aggregation and initialization details.

**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $V$ of dimensions $D_x \times D_y \times D_z$. In medical imaging contexts, this is often a NIfTI file where dimension 0 is the X-axis (sagittal slices), dimension 1 is the Y-axis (coronal slices), and dimension 2 is the Z-axis (axial slices) [S1, unofficial].
*   **Output:** A 2D image array $I$ of dimensions $D_y \times D_z$ (assuming projection along the X-axis) [S1, unofficial].

**Data Structures:**
*   **Volume Array:** A 3D array where $V[x, y, z]$ stores the scalar value at voxel $(x, y, z)$.
*   **Output Array:** A 2D array initialized to zero.

**Steps:**
1.  **Initialize:** Create a 2D output array $I$ with dimensions matching the Y and Z axes of the input volume, initialized to zero [S1, unofficial].
    $$ I[y, z] = 0 \quad \forall y \in [0, D_y), z \in [0, D_z) $$
2.  **Iterate and Sum:** For each voxel (or slice) along the projection axis (e.g., X-axis), add its value to the corresponding pixel in the output array [S1, unofficial].
    $$ I[y, z] \leftarrow I[y, z] + V[x, y, z] $$
    This is typically implemented by iterating through the slices along the projection axis and accumulating them [S1, unofficial].
3.  **Classification:** Map the final accumulated value $A = I[y, z]$ to a color $\mathbf{c}$ using a transfer function $T(A) \rightarrow \mathbf{c}$ [Parent]. Opacity is often set to $1.0$ or derived from the transfer function, but depth is not accumulated [Parent].

**Pseudocode:**
The following Python-like pseudocode implements the summation projection along the first dimension (X-axis) of a 3D numpy array `img_np` [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def projection_summation(img_np):
    """
    Computes the summation projection of a 3D volume along the first axis (X).
    
    Args:
        img_np: A 3D numpy array of shape (D_x, D_y, D_z).
        
    Returns:
        A 2D numpy array of shape (D_y, D_z) containing the summed values.
    """
    # Initialize output array with zeros, matching Y and Z dimensions
    # img_np.shape[0] is D_x, shape[1] is D_y, shape[2] is D_z
    vr = np.zeros((img_np.shape[1], img_np.shape[2]))
    
    # Iterate through each slice index i along the X-axis
    for i in range(img_np.shape[0]):
        # Extract the 2D slice at index i
        # img is of shape (D_y, D_z)
        img = img_np[i, :, :]
        
        # Add the slice to the accumulator
        vr += img
        
    return vr
```

**Parameters:**
*   **Projection Axis:** The axis along which summation occurs. In the example above, it is the X-axis (dimension 0) [S1, unofficial].
*   **Accumulator Initialization:** Zero ($0$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Contrast Reduction:** The method produces images with inferior contrast compared to slice display and Maximum Intensity Projection because background activity accumulates [S3, unofficial].
*   **Noise:** While it decreases noise compared to raw projection, it may still be noisier than Maximum Activity Projection in some contexts [S3, unofficial].
*   **Axis Orientation:** The implementation must correctly map the volume's internal axis ordering (e.g., medical NIfTI axes) to the output image dimensions [S1, unofficial].

## Subtopics

### Cylinder Projection
A viewpoint-independent projection technique designed for producing film loops where the viewpoint is rotated about the z-axis [S4]. The 3D vector is projected onto the surface in the direction of a vector pointing away from the rotational axis [S4]. It preserves the z component of the vector field, providing a more intuitive visualization than standard normal projection, though uniformity may still not be readily apparent [S4]. It is only suitable for simple surfaces which are centered on the rotational axis [S4].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the volume, as each voxel is visited exactly once [S1, unofficial].
*   **Memory Complexity:** $O(D_y \times D_z)$ for the output image, plus $O(N)$ for the input volume [S1, unofficial].
*   **Performance:** Very fast, as it involves simple addition without interpolation or compositing [S1, unofficial].
*   **Strengths:** Reduces image noise; simple to implement; fast execution [S3, unofficial].
*   **Limitations:** Significantly reduces image contrast due to cumulative background activity; inferior contrast compared to slice display and Maximum Intensity Projection [S3, unofficial].
*   **When to use:** When noise reduction is prioritized over contrast, or for quick visualization of total intensity [S3, unofficial].
*   **When not to use:** When highlighting specific high-intensity structures (use [Maximum Intensity Projection](902-maximum-intensity-projection.md)) or when depth discrimination is needed [S3, unofficial].

## Background
Projection (Summation) is a fundamental technique in direct volume rendering, often used in medical imaging (e.g., CT, SPECT) to visualize total density or signal [S1, S3, unofficial]. It is a variant of the broader class of projection techniques that includes MIP and Minimum Intensity Projection [Parent]. The technique is particularly relevant in nuclear medicine (SPECT) where it is referred to as "summed reprojection" [S3, unofficial].

## Sources
[S1] mneela/Volume-Rendering, https://github.com/mneela/Volume-Rendering, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S3] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
[S4] Visualizing 3D velocity fields near contour surfaces, https://doi.org/10.5555/951087.951133, fetched 2026-10-26
