# Point-Cloud Based CGH Calculation
Also called: Point-Cloud Approach, Point-based CGH, Summation of Point Contributions

## Overview
The Point-Cloud Based CGH Calculation method generates holograms by discretizing a 3D scene into a set of points, treating each as a light-emitting primitive, and summing their individual wavefield contributions onto the hologram plane [S1]. It is distinguished from sibling methods by its direct geometric summation of point sources, offering simplicity and the ability to process real-world scanned data, but suffers from high computational costs for large scenes compared to convolution-based or layer-based approaches [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $M$ 3D points $\mathbf{p}_i = (x_i, y_i, z_i)$ representing the scene, where each point has an associated amplitude $A_i$ and initial phase $\phi_{0,i}$. The hologram plane is defined by resolution $N_x \times N_y$ and pixel pitch $\Delta$.
*   **Output:** A complex-valued hologram $H(x,y)$ or its phase-quantized version for display.

**Data Structures:**
*   **Point Cloud:** An array of structures containing coordinates $(x_i, y_i, z_i)$, amplitude $A_i$, and phase $\phi_{0,i}$.
*   **Hologram Plane:** A 2D complex array of size $N_x \times N_y$.

**Mathematical Model:**
The complex amplitude $U_h(x,y)$ at a hologram pixel $(x,y)$ is the superposition of waves from all $M$ points [S1, S3]:
$$ U_h(x,y) = \sum_{i=1}^{M} A_i \exp\left(i \phi_i(x,y)\right) $$
The phase $\phi_i(x,y)$ for point $i$ at pixel $(x,y)$ is determined by the distance $r_i$ [S3]:
$$ \phi_i(x,y) = k r_i + \phi_{0,i} $$
$$ r_i = \sqrt{(x - x_i)^2 + (y - y_i)^2 + z_i^2} $$
where $k = 2\pi/\lambda$ is the wavenumber and $\lambda$ is the wavelength [S3].

**Algorithm Steps:**
1.  Initialize the hologram plane $H(x,y)$ to zero complex values.
2.  For each point $i$ from $1$ to $M$:
    a. Calculate the distance $r_i(x,y)$ from point $i$ to every pixel $(x,y)$ on the hologram plane.
    b. Compute the phase $\phi_i(x,y) = k r_i(x,y) + \phi_{0,i}$.
    c. Add the contribution to the hologram: $H(x,y) \leftarrow H(x,y) + A_i \exp(i \phi_i(x,y))$.
3.  (Optional) Quantize the phase of $H(x,y)$ to $[0, 2\pi]$ for phase-only SLMs [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_point_cloud_cgh(points, hologram_width, hologram_height, wavelength, pixel_pitch):
    """
    Calculate CGH using point-cloud summation.
    
    Args:
        points: List of tuples (x, y, z, amplitude, initial_phase)
        hologram_width: int, number of pixels in x
        hologram_height: int, number of pixels in y
        wavelength: float, wavelength of light in same units as coordinates
        pixel_pitch: float, size of one hologram pixel
        
    Returns:
        hologram: 2D numpy array of complex values
    """
    import numpy as np
    
    # Initialize hologram plane
    hologram = np.zeros((hologram_height, hologram_width), dtype=np.complex128)
    
    # Wavenumber
    k = 2.0 * np.pi / wavelength
    
    # Create coordinate grids for the hologram plane
    # Assuming hologram center is at (0,0) in object space for simplicity
    # In practice, coordinate offsets must be handled based on scene/hologram alignment
    x_coords = np.arange(hologram_width) * pixel_pitch
    y_coords = np.arange(hologram_height) * pixel_pitch
    X, Y = np.meshgrid(x_coords, y_coords)
    
    for point in points:
        px, py, pz, amp, phi0 = point
        
        # Calculate distance from point to each hologram pixel
        # r = sqrt((x - px)^2 + (y - py)^2 + (z - pz)^2)
        # Note: z is depth. If hologram is at z=0, then distance is sqrt(x^2 + y^2 + pz^2)
        # relative to the point's z-coordinate.
        # Assuming hologram plane is at z=0 and point is at z=pz:
        dx = X - px
        dy = Y - py
        dz = -pz  # Distance along z-axis from point to hologram plane
        R = np.sqrt(dx**2 + dy**2 + dz**2)
        
        # Calculate phase
        phase = k * R + phi0
        
        # Add contribution
        # Using complex exponential: exp(i*phase)
        contribution = amp * np.exp(1j * phase)
        hologram += contribution
        
    return hologram
```

**Parameters:**
*   **Wavelength ($\lambda$):** Must match the reconstruction light source [S3].
*   **Sub-hologram Size:** In tiled approaches, computation time scales quadratically with sub-hologram size [S2].

**Edge Cases and Pitfalls:**
*   **Occlusion:** The basic summation method does not account for occlusion or reflection, leading to incorrect intensity if these effects are significant [S1].
*   **Precision:** Large numbers of points can lead to numerical accumulation errors; use double precision for amplitude/phase accumulation [G].
*   **Boundary Conditions:** Points far from the hologram plane may require paraxial approximations if the exact spherical wave formula is too costly, but this reduces accuracy [G].

## Subtopics

### Look-Up-Table (LUT) Acceleration
Pre-calculates point contributions to reduce runtime. [LUT Acceleration](link-to-lut.md)

### Wavefront Recording Plane (WRP)
Uses an intermediate plane to speed up propagation. [WRP Method](link-to-wrp.md)

### Convolution-Based CGH
Replaces point-wise summation with FFT-based convolution for speed. [Convolution CGH](link-to-convolution-cgh.md)

## Operating characteristics
*   **Time Complexity:** $O(M \cdot N)$, where $M$ is the number of points and $N$ is the number of hologram pixels. If tiled, it scales quadratically with sub-hologram size [S2].
*   **Memory Complexity:** $O(N)$ for the hologram plane, plus $O(M)$ for the point cloud.
*   **Performance:** On an NVIDIA GTX 980 Ti, calculating a hologram for 2 million points takes one to tens of seconds, depending on sub-hologram size [S2]. This is significantly slower than convolution methods, which run in milliseconds for similar resolutions [S2].
*   **Strengths:** Simple implementation; supports real-world scanned point clouds; no need for scene segmentation into layers or polygons [S1].
*   **Limitations:** Unacceptably high computational cost for large scenes with occlusion/reflection; not suitable for real-time applications without acceleration techniques like LUTs or WRP [S1].

## Background
The point-cloud approach is one of the earliest and most fundamental methods for CGH generation, rooted in the physical principle of wave superposition from discrete sources [S1]. It was developed as a direct numerical implementation of Huygens' principle for computer rendering [S1]. It is widely used in research for validating holographic algorithms and for processing real-world 3D scans [S1].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
[S2] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
