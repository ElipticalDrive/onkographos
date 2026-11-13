# Polar Coordinate Fourier Volume Rendering

Also called: Fourier volume rendering by polar coordinate data representation, Polar FVR

## Overview
Polar Coordinate Fourier Volume Rendering (Polar FVR) is a domain-transform volume rendering method that stores and processes frequency domain data in polar coordinates to reduce computational complexity from $O(n^2 \log n)$ to $O(n^2)$ and improve memory access efficiency [S1]. It sets itself apart from its parent [Fourier Volume Rendering](6050-fourier-volume-rendering.md) by exploiting data regularity and high density near the origin in polar space, allowing slice extraction via bilinear or nearest-neighbor interpolation instead of trilinear interpolation, and shifting the majority of computational cost to a one-time preprocessing stage [S1].

## How it works
**Inputs:** 3D spatial domain volume data $V(x,y,z)$.
**Outputs:** 2D projection image $P(u,v)$.

**Data Structures:**
*   **Rectilinear Frequency Grid:** 3D array of Hartley coefficients.
*   **Polar Frequency Grid:** 3D array storing data along the radial direction first. The layout ensures that data accessed during slice extraction are in adjacent memory locations [S1].
*   **Slice Data:** 2D array representing the extracted slice in polar coordinates, which is then converted to rectilinear coordinates for the inverse transform.

**Parameters:**
*   **Interpolation for Slice Extraction:** Nearest-neighbor interpolation is used in the interactive rendering stage to achieve speed independent of interpolation kernel size [S1].
*   **Interpolation for Coordinate Conversion:** A better interpolation kernel with larger support can be used during the one-time preprocessing conversion from rectilinear to polar coordinates [S1].

**Steps:**
1.  **Generate Rectilinear Frequency Data:** Apply the 3D Hartley transform to the spatial domain data to obtain rectilinear coordinate frequency domain data [S1].
2.  **Convert to Polar Coordinates:** Convert the rectilinear coordinate frequency data to polar coordinates. This is a one-time preprocessing step [S1].
3.  **Extract Slice:** Extract a slice from the polar coordinate data. Due to the high data density near the origin, this can be done using nearest-neighbor interpolation without sacrificing image quality, or bilinear interpolation between adjacent angles [S1].
4.  **Convert Slice to Rectilinear:** Convert the extracted slice from polar coordinates back to rectilinear coordinates. This conversion is performed using bilinear interpolation [S1].
5.  **Inverse Transform:** Apply the 2D inverse Hartley transform to the rectilinear slice to obtain the projection [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polar_fvr(volume_data):
    """
    volume_data: 3D numpy array of spatial domain data.
    Returns: 2D numpy array of the projection image.
    """
    # Step 1: Generate rectilinear coordinate frequency domain data
    # Apply 3D Hartley transform
    freq_rect = hartley_transform_3d(volume_data)
    
    # Step 2: Convert rectilinear coordinate data to polar coordinate
    # One-time preprocessing
    freq_polar = rectilinear_to_polar(freq_rect)
    
    # Step 3: Extract a slice from the polar coordinate data
    # For interactive rendering, use nearest-neighbor for speed
    # Or bilinear interpolation between adjacent angles
    slice_polar = extract_slice_polar(freq_polar)
    
    # Step 4: Convert the slice from polar coordinate back to rectilinear coordinate
    # Use bilinear interpolation for this conversion
    slice_rect = polar_to_rectilinear(slice_polar)
    
    # Step 5: Apply 2D inverse Hartley transform
    projection = inverse_hartley_transform_2d(slice_rect)
    
    return projection
```

**Edge Cases and Pitfalls:**
*   **Coordinate Conversion Necessity:** Extracted slices are in polar coordinates and must be converted back to rectilinear coordinates before applying the inverse Fourier transform [S1].
*   **Memory Access:** Storing polar coordinate data along the radial direction first is critical for efficient memory access during slice extraction [S1].

## Subtopics

### [Bilinear Interpolation in Fourier Space](14350-bilinear-interpolation-in-fourier-space.md)
Bilinear Interpolation in Fourier Space (F RBiLin) is a method to reconstruct a Cartesian Fourier grid from polar samples by interpolating the four nearest polar points surrounding a target pixel. It solves the problem of mapping frequency domain data, acquired in polar coordinates (typically from projection slices), onto a regular Cartesian grid required for the Inverse Fast Fourier Transform (IFFT). It sets itself apart from nearest-neighbor approaches by using weighted averaging based on Euclidean distance, and from Radial-Angular variants by using Cartesian Euclidean distance rather than arc/radial distance.

## Operating characteristics
**Time and Memory Complexity:**
*   Computational complexity is reduced to $O(n^2)$ from the $O(n^2 \log n)$ complexity of traditional rectilinear FVR [S1].
*   The interactive rendering stage is extremely fast, with speed independent of the size of the interpolation kernel [S1].

**Measured Performance:**
*   Tested on a desktop computer with a Pentium 4 3.2 GHz processor and 1 GB of RAM [S1].
*   Dataset: CT Head dataset, originally 256x256x113 samples, zero-filled and duplicated to 256x256x256 samples for testing [S1].

**Strengths and Limitations:**
*   **Strengths:** Reduced computational complexity, efficient memory access due to data regularity, ability to trade off between image quality and memory storage by reducing sampling density at high spatial frequencies [S1].
*   **Limitations:** Requires a one-time preprocessing step for coordinate conversion, which may be significant for very large datasets [S1].

**When to Use:**
*   When real-time or interactive visualization is required, and the preprocessing cost can be amortized [S1].
*   When memory access efficiency is a concern [S1].

**When Not to Use:**
*   When the preprocessing time is prohibitive and interactive performance is not critical [S1].

## Background
Polar Coordinate Fourier Volume Rendering was proposed to address the high computational complexity of traditional Fourier Volume Rendering (FVR) [S1]. Traditional FVR requires one trilinear interpolation for each pixel to obtain a slice, resulting in $O(n^2 \log n)$ complexity [S1]. The method leverages the Fourier projection-slice theorem, which is the foundation of [Fourier Volume Rendering](6050-fourier-volume-rendering.md) [S1]. The approach was introduced in a paper titled "Accelerating Fourier volume rendering by polar coordinate data representation" [S1].

## Sources
[S1] Accelerating Fourier volume rendering by polar coordinate data representation, https://doi.org/10.1016/j.cmpb.2012.05.010, fetched 2026-10-27
