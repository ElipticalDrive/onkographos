# Projection

Also called: Direct Volume Rendering via Projection, Orthographic Projection (Summation), Summation Projection, Summed Reprojection, Summed projection, Total Intensity Projection, Voxel Projection

## Overview
Projection is a volume rendering technique that visualizes the total accumulated intensity of a 3D volume along a specific axis by summing voxel values [S1, unofficial]. It solves the problem of compressing volumetric data into a 2D representation while preserving the total signal, which effectively decreases image noise compared to raw slice displays [S3, unofficial]. Unlike its siblings [Maximum Intensity Projection](902-maximum-intensity-projection.md) and [Minimum Intensity Projection](5299-minimum-intensity-projection.md), which highlight extreme values, Projection emphasizes cumulative density, making it suitable for structures where total mass or integrated signal is the primary diagnostic feature, though it suffers from reduced contrast due to the superposition of background activity [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume array $V$ of dimensions $D_x \times D_y \times D_z$. The axes correspond to spatial dimensions (e.g., X, Y, Z).
*   **Output:** A 2D projection array $P$ of dimensions $D_{proj1} \times D_{proj2}$, where the projection axis is collapsed.

**Data Structures:**
*   $V$: A dense 3D array (e.g., `numpy.ndarray` or `float32`/`float64` matrix).
*   $P$: A 2D array initialized to zeros, matching the dimensions of the volume perpendicular to the projection axis.

**Algorithm:**
The method performs an orthographic projection along one of the principal axes (typically the first dimension, X).

1.  Initialize the output array $P$ with zeros. The shape of $P$ is $(D_y, D_z)$ if projecting along the X-axis.
    $$ P[y, z] = 0 \quad \forall y \in [0, D_y), z \in [0, D_z) $$
2.  Iterate through each slice index $i$ along the projection axis (X-axis, $i \in [0, D_x)$).
3.  Extract the 2D slice $S_i$ at index $i$:
    $$ S_i[y, z] = V[i, y, z] $$
4.  Accumulate the slice values into the corresponding pixel in $P$:
    $$ P[y, z] \leftarrow P[y, z] + S_i[y, z] $$
5.  Repeat steps 2–4 for all $i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projection(volume):
    """
    Computes the summation projection of a 3D volume along the first axis (X).
    
    Args:
        volume: A 3D numpy array of shape (Dx, Dy, Dz).
        
    Returns:
        A 2D numpy array of shape (Dy, Dz) containing the summed intensities.
    """
    # Initialize output array with zeros, matching the dimensions of the 
    # axes perpendicular to the projection axis (Y and Z).
    projection_map = np.zeros((volume.shape[1], volume.shape[2]))
    
    # Iterate through each slice along the first axis (X)
    for i in range(volume.shape[0]):
        # Extract the 2D slice at index i
        slice_data = volume[i, :, :]
        
        # Add the slice values to the projection map
        projection_map += slice_data
        
    return projection_map
```

**Parameters:**
*   **Projection Axis:** The axis along which summation occurs. In the standard implementation described in [S1, unofficial], this is the first dimension (X).
*   **Data Type:** Floating-point arithmetic is recommended to prevent overflow during summation, especially for large volumes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Contrast Reduction:** Because background structures accumulate along the ray, they can obscure foreground structures with similar total intensity, leading to inferior contrast compared to slice displays or Maximum Intensity Projection [S3, unofficial].
*   **Noise vs. Signal:** While summation reduces high-frequency noise (averaging effect), it also reduces the relative contrast of small lesions or thin structures against a noisy background [S3, unofficial].
*   **Axis Orientation:** The interpretation of the output depends on the coordinate system of the input volume. In medical imaging (NIfTI), the axes often correspond to Sagittal (X), Coronal (Y), and Axial (Z) planes [S1, unofficial].

## Subtopics

### [Cylinder Projection](12798-cylinder-projection.md)
A viewpoint-independent technique for visualizing 3D vector fields on surfaces, projecting vectors radially away from a rotational axis to preserve the z-component. It is distinct from scalar volume summation and is used for film loops of rotating data.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the volume. Each voxel is visited exactly once.
*   **Memory Complexity:** $O(D_y \times D_z)$ for the output array, plus $O(N)$ for the input volume.
*   **Performance:** Extremely fast as it involves only addition operations without interpolation, transfer function evaluation, or alpha blending.
*   **Strengths:** Noise reduction; simple implementation; preserves total signal intensity; order-independent.
*   **Limitations:** Poor contrast for small or low-density structures embedded in high-density backgrounds; loss of depth information; "blooming" effect where overlapping structures merge indistinguishably [S3, unofficial].
*   **When to use:** When the total integrated signal is the metric of interest (e.g., total bone mass, total tracer uptake) and noise reduction is prioritized over local contrast.
*   **When not to use:** When distinguishing overlapping structures of similar intensity is required, or when high local contrast is needed for small lesions [S3, unofficial].

## Background
The method originates from early direct volume rendering techniques that sought to compress 3D data into 2D displays for interpretation [S2, unofficial]. It is widely used in medical imaging, particularly in SPECT and CT, as a preprocessing or alternative visualization step to [Maximum Intensity Projection](902-maximum-intensity-projection.md) [S3, unofficial]. The technique is often referred to as "Summed Reprojection" in nuclear medicine literature, where it is compared against raw projections and maximum activity projections for lesion detectability [S3, unofficial].

## Sources
[S1] mneela/Volume-Rendering, https://github.com/mneela/Volume-Rendering, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S3] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
