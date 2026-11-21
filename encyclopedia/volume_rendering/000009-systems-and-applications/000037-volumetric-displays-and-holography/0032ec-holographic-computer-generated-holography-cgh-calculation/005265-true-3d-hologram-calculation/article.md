# True 3D Hologram Calculation

Also called: Fresnel holography with phase correction

## Overview
True 3D Hologram Calculation generates a hologram where scene elements are positioned at their actual 3D coordinates, providing accurate depth cues and parallax for all viewing angles. It differs from planar hologram methods by explicitly correcting the object phase to maintain consistency across the hologram plane, preventing the noise and artifacts that arise when using a constant object phase for objects at varying depths [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A point cloud of object points $\mathbf{o}$ derived from color and depth buffers, with occluded points culled from the viewer's eye position. The wavelength $\lambda$ of the reconstruction light. The position of the hologram plane.
*   **Outputs:** A complex amplitude distribution or phase-only hologram pattern ready for upload to a Spatial Light Modulator (SLM) [S1].

**Data Structures:**
*   **Point Cloud:** A list of 3D coordinates $\mathbf{o} = (x_o, y_o, z_o)$ and associated color/intensity values.
*   **Hologram Plane:** A 2D grid of pixels $(x, y)$ at a specific depth $z_h$.

**Method Steps:**
1.  **Point Cloud Generation:** Form a point cloud from the viewer's eye position using standard rendering pipeline color and depth buffers. Cull occluded points to ensure only visible surfaces contribute to the hologram [S1].
2.  **Phase Correction Calculation:** For each object point $\mathbf{o}$, calculate the corrected object phase $\phi_o$. This correction ensures that the phase contribution is consistent across the sub-hologram associated with that point, mitigating the wide variation in phase that occurs with true 3D depth [S1].
3.  **Hologram Synthesis:** Calculate the total complex amplitude at each hologram pixel by summing the contributions of all object points, using the corrected phase $\phi_o$ and the actual 3D position of each point in the wave propagation model [S1].

**Formulas:**
The corrected object phase $\phi_o$ is calculated as [S1]:

$$ \phi_o = -\frac{2\pi}{\lambda} \left( k|\mathbf{p}_c^o - \mathbf{o}| - k|\mathbf{p}_c^o - \mathbf{k}| \right) $$

Where:
*   $\lambda$ is the wavelength of the reconstruction light.
*   $k$ is the wavenumber ($2\pi/\lambda$).
*   $\mathbf{o}$ is the 3D position of the object point.
*   $\mathbf{p}_c^o$ is the center point of the sub-hologram corresponding to object point $\mathbf{o}$.
*   $\mathbf{k}$ is a reference vector or position used to define the phase reference (often related to the optical axis or a reference beam origin, though the source text implies a geometric correction term involving the distance from the sub-hologram center to the object and a reference point) [S1].

*Note: The source text presents the formula as $\phi_o = -\frac{2\pi}{\lambda} (k|\mathbf{p}_c^o - \mathbf{o}| - k|\mathbf{p}_c^o - \mathbf{k}|)$. The term $|\mathbf{p}_c^o - \mathbf{k}|$ appears to be a constant reference distance for the sub-hologram, ensuring phase continuity.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_true_3d_hologram(point_cloud, wavelength, hologram_plane, sub_hologram_centers):
    """
    Calculate a true 3D hologram with phase correction.
    
    Args:
        point_cloud: List of object points o with 3D coordinates.
        wavelength: Float, lambda of reconstruction light.
        hologram_plane: Grid of pixels (x, y) at depth z_h.
        sub_hologram_centers: Dictionary mapping object point index to center point p_c^o.
        
    Returns:
        hologram: Complex amplitude array for the hologram plane.
    """
    k = 2 * pi / wavelength
    hologram = zeros_like(hologram_plane, dtype=complex)
    
    for o in point_cloud:
        # Get the center of the sub-hologram for this object point
        pc_o = sub_hologram_centers[o.index]
        
        # Calculate corrected object phase phi_o
        # Distance from sub-hologram center to object point
        dist_obj = norm(pc_o - o)
        # Reference distance term (constant for the sub-hologram)
        # Assuming k in the formula refers to a reference point or axis origin
        # The source formula is: phi_o = - (2*pi/lambda) * (k * |pc_o - o| - k * |pc_o - k|)
        # Note: The variable 'k' in the source formula inside the parenthesis 
        # likely represents a reference point vector, distinct from wavenumber k.
        # Let's denote reference point as ref_k.
        dist_ref = norm(pc_o - ref_k) 
        
        phi_o = - (2 * pi / wavelength) * (k * dist_obj - k * dist_ref)
        
        # Calculate contribution to hologram plane
        for pixel in hologram_plane:
            # Distance from object point to hologram pixel
            dist = norm(pixel - o)
            
            # Phase at pixel due to propagation from object
            phi_prop = k * dist
            
            # Total phase at pixel
            total_phase = phi_o + phi_prop
            
            # Amplitude A_o (from point cloud data, e.g., intensity)
            A_o = o.intensity
            
            # Add contribution
            hologram[pixel] += A_o * exp(1j * total_phase)
            
    return hologram
```

**Edge Cases and Pitfalls:**
*   **Constant Phase Error:** Using a constant object phase $\phi_o$ for true 3D holograms results in widely varying object phase at the hologram plane, producing poor quality results with high noise [S1].
*   **Intra-pupil Effects:** The method assumes view-dependent effects are handled by head-tracked rendering. It does not account for intra-pupil view-dependent effects (e.g., differences in occlusion between the left and right edges of the eye pupil) [S1].

## Subtopics

### [Holographic Defocus Blur Matching](15055-holographic-defocus-blur-matching.md)
Holographic Defocus Blur Matching calculates the maximum propagation distance $\Delta z$ and the diameter of the blur circle of a human eye $CoC_{eye}$ in display pixels. It ensures that the holographic defocus blur matches the optical defocus blur of the human eye. It sets itself apart by linking SLM and lens array optical constraints to the physiological limits of the human pupil, allowing for the determination of rendering camera aperture sizes that produce perceptually correct depth cues.

### Digital 3D display system based on image synthesizing technology
A processing method that captures 2D elemental images via a micro lens array, calculates holograms using a corrected Iterative Fourier Transform Algorithm (IFTA), uploads them to an SLM, and reconstructs a 3D image using output-side micro lens array imaging. Implemented by Choi et al. (2005) to construct a full-parallax wide-viewing-angle 3D holographic display system. Steps: 1. Capture 2D elemental images of a 3D object using a CCD and input micro lens array. 2. Calculate the hologram of the elemental images using a corrected IFTA. 3. Upload the calculated hologram to an SLM. 4. Reconstruct the 3D image using image formation from the output micro lens array [S2].

## Operating characteristics
*   **Complexity:** The calculation scales with the number of object points and hologram pixels. For true 3D, every point contributes to every pixel, leading to $O(N \cdot M)$ complexity where $N$ is points and $M$ is hologram pixels [G].
*   **Performance:** Achieving good results with phase-only modulation is difficult without phase correction. The phase correction method allows for high-quality holograms with widely varying depth, but requires careful computation of the reference distances [S1].
*   **Strengths:** Provides accurate 3D depth cues and parallax. Corrects for phase inconsistencies inherent in naive true 3D Fresnel holography [S1].
*   **Limitations:** Computationally expensive. Does not account for intra-pupil view-dependent effects [S1].

## Background
The method is described in the context of holographic near-eye displays for virtual and augmented reality. It addresses the challenge of generating high-quality true 3D holograms using phase-only modulation, where naive approaches fail due to phase variation across the hologram plane [S1]. The phase correction formula was introduced to improve consistency on the hologram plane [S1].

## Sources
[S1] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
[S2] Principle and Outlook of Holography, https://doi.org/10.2991/icmmita-16.2016.54, fetched 2026-10-26
