# Subsampled Reconstruction

Also called: Partial Decoding for Fast Interaction, Subsampled Time Step Decoding

## Overview
Subsampled Reconstruction is a technique for time-varying volume rendering that enables fluent user interaction by decoding only a subsampled portion of the compressed data rather than the full time step [S1, unofficial]. It addresses the latency bottleneck in real-time exploration of large 4D datasets, where full reconstruction times can exceed 100ms [S1, unofficial]. Unlike full reconstruction or sibling methods like DCT-compressed time-runs rendering, it prioritizes interaction speed over spatial fidelity by decoding a fraction of the data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed bitstream of a time-varying volume dataset and a subsampling factor (e.g., 1:8) [S1, unofficial].
*   **Output:** A reconstructed 3D volume slice at the current time step containing only the subsampled voxel values [S1, unofficial].

**Data Structures:**
The method operates on the compressed dataset and a target subsampling factor [S1, unofficial]. Exact layouts are Not given in the sources [S1, unofficial].

**Steps:**
1.  **Input:** Load the compressed dataset and target subsampling factor [S1, unofficial].
2.  **Subsampled Decoding:** Decode only the coefficients corresponding to the subsampled spatial locations, skipping the rest of the data [S1, unofficial].
3.  **Output:** Return the reconstructed 3D volume slice at the current time step [S1, unofficial].

**Parameters:**
*   **Subsampling Factor:** The ratio of skipped voxels (e.g., 1:8) [S1, unofficial].
*   **Compression Ratio:** The underlying compression of the dataset (e.g., 1:128) [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subsampled_reconstruction(compressed_data, subsampling_factor):
    """Reconstructs a volume from compressed data using subsampled decoding."""
    full_dims = compressed_data.get_spatial_dimensions()
    k = subsampling_factor
    sub_dims = [d // k for d in full_dims]
    reconstructed_volume = [[[0.0 for _ in range(sub_dims[2])] for _ in range(sub_dims[1])] for _ in range(sub_dims[0])]
    
    for x in range(0, full_dims[0], k):
        for y in range(0, full_dims[1], k):
            for z in range(0, full_dims[2], k):
                # The sources do not specify the exact selective decoding or inverse transform routine.
                value = 0.0
                reconstructed_volume[x // k][y // k][z // k] = value
    return reconstructed_volume
```

## Subtopics

### [Multiresolution Volume Sampling](14183-multiresolution-volume-sampling.md)
An adaptive spatial subsampling scheme that progressively reconstructs a 3D time-slice from a 4D base tensor. It differs by refining to full resolution when interaction slows and reusing previously computed values.

## Operating characteristics
*   **Time Complexity:** Significantly faster than full reconstruction [S1, unofficial]. For a 1:8 subsampled time step from a 1:128 compressed dataset, reconstruction takes approximately 20.8ms [S1, unofficial].
*   **Memory Complexity:** Reduced decoding load, as only a subsampled portion of the data is processed [S1, unofficial].
*   **Strengths:** Enables fluent interaction for large datasets where full reconstruction would cause lag (>100ms) [S1, unofficial].
*   **Limitations:** Lower spatial resolution due to subsampling [S1, unofficial].

## Background
The method was proposed in the context of Tensor-Train-based 4D time-dependent volume rendering (TT4D) to address the high reconstruction times associated with large datasets [S1, unofficial].

## Sources
[S1] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
