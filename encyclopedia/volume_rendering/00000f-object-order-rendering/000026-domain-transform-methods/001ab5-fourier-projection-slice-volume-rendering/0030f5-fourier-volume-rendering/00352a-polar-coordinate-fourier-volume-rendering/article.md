# Polar Coordinate Fourier Volume Rendering

Also called: Fourier volume rendering by polar coordinate data representation, Polar FVR

## Overview
Polar Coordinate Fourier Volume Rendering (Polar FVR) is a domain-transform volume rendering method that stores and processes frequency domain data in polar coordinates to reduce computational complexity from $O(n^2 \log n)$ to $O(n^2)$ and improve memory access efficiency [S1]. It sets itself apart from its parent [Fourier Volume Rendering](6050-fourier-volume-rendering.md) by exploiting data regularity and high density near the origin in polar space, allowing slice extraction via bilinear or nearest-neighbor interpolation instead of trilinear interpolation, and shifting the majority of computational cost to a one-time preprocessing stage [S1].

## How it works
**Inputs:** 3D spatial domain volume data $V(x,y,z)$ [S1].
**Outputs:** 2D projection image $P(u,v)$ [S1].

**Data Structures:**
*   **Rectilinear Frequency Grid:** 3D array of Hartley coefficients [S1].
*   **Polar Frequency Grid:** 3D array storing data along the radial direction first, ensuring that data accessed during slice extraction are in adjacent memory locations [S1].
*   **Slice Data:** 2D array representing the extracted slice in polar coordinates, which is then converted to rectilinear coordinates for the inverse transform [S1].

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
    # Step 1: Generate rectilinear coordinate frequency domain data [S1]
    freq_rect = hartley_transform_3d(volume_data)
    
    # Step 2: Convert rectilinear coordinate data to polar coordinate [S1]
    freq_polar = rectilinear_to_polar(freq_rect)
    
    # Step 3: Extract a slice from the polar coordinate data [S1]
    slice_polar = extract_slice_polar(freq_polar)
    
    # Step 4: Convert the slice from polar coordinate back to rectilinear coordinate [S1]
    slice_rect = polar_to_rectilinear(slice_polar)
    
    # Step 5: Apply 2D inverse Hartley transform [S1]
    projection = inverse_hartley_transform_2d(slice_rect)
    
    return projection

def hartley_transform_3d(data):
    # Standard 3D Hartley transform: H(u,v,w) = sum_{x,y,z} data(x,y,z) * (cos(2pi(ux+vy+wz)) + sin(2pi(ux+vy+wz)))
    # Computed via standard fast algorithm.
    return standard_hartley_3d(data)

def rectilinear_to_polar(rect_grid):
    # Map rectilinear (u,v,w) to polar (r, theta, phi)
    # r = sqrt(u^2 + v^2 + w^2)
    # theta = atan2(v, u)
    # phi = atan2(w, sqrt(u^2 + v^2))
    # Interpolate rect_grid onto polar grid using bilinear/bicubic kernel [S1]
    return interpolate_to_polar_grid(rect_grid)

def extract_slice_polar(polar_grid):
    # Extract slice at specified viewing angle using nearest-neighbor or bilinear interpolation
    # between adjacent angles [S1]
    return interpolate_slice(polar_grid)

def polar_to_rectilinear(polar_slice):
    # Convert polar slice back to rectilinear coordinates using bilinear interpolation [S1]
    return interpolate_to_rectilinear_grid(polar_slice)

def inverse_hartley_transform_2d(slice_data):
    # Standard 2D inverse Hartley transform to obtain spatial projection [S1]
    return standard_inverse_hartley_2d(slice_data)
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

## Background
Polar Coordinate Fourier Volume Rendering was proposed to address the high computational complexity of traditional Fourier Volume Rendering (FVR) [S1]. Traditional FVR requires one trilinear interpolation for each pixel to obtain a slice, resulting in $O(n^2 \log n)$ complexity [S1]. The method leverages the Fourier projection-slice theorem, which is the foundation of [Fourier Volume Rendering](6050-fourier-volume-rendering.md) [S1]. The approach was introduced in a paper titled "Accelerating Fourier volume rendering by polar coordinate data representation" [S1].

## Sources
[S1] Accelerating Fourier volume rendering by polar coordinate data representation, https://doi.org/10.1016/j.cmpb.2012.05.010, fetched 2026-10-27
