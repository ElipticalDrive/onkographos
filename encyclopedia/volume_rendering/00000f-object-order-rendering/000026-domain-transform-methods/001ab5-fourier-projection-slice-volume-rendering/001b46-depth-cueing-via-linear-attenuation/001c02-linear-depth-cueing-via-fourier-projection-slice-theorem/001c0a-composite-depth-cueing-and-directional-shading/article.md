# Composite Depth Cueing and Directional Shading

Also called: Composite Shading Model, Hemispherical Illumination Shading

## Overview
Composite Depth Cueing and Directional Shading is a frequency-domain rendering technique that simultaneously applies linear depth cueing (attenuation) and hemispherical directional shading to volume data [S1, unofficial]. It solves the problem of efficiently computing both effects within the Fourier Projection-Slice framework, where naive spatial-domain computation would be prohibitively expensive [S1, unofficial]. It sets itself apart from its parent method, [Linear Depth Cueing via Fourier Projection-Slice Theorem](1129-linear-depth-cueing-via-fourier-projection-slice-theorem.md), by combining the attenuation model with a hemispherical illumination model that avoids the nonlinear absolute value operator, allowing the lighting direction to be factored out of the integral [S1, unofficial]. This requires 16 precomputed spectra and is slower than individual models due to a high constant factor, though asymptotically less expensive than conventional rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume of albedo $\Omega(x,y,z)$, orientation field $N(x,y,z)$, depth vector field $D(x,y,z)$, viewing direction $V$, lighting direction $L$, attenuation parameter $\tau$, and light intensity $I_s$ [S1, unofficial].
*   **Output:** Rendered image intensity $I$ [S1, unofficial].

**Data Structures:**
The method relies on 16 precomputed 3D spectra derived from the albedo volume $\Omega$ and the geometric fields $N$ and $D$ [S1, unofficial].
*   **VL Vector:** A tuple of 13 elements representing products of viewing direction components $V$ and lighting direction components $L$ [S1, unofficial].
    *   Layout: $(V_x L_x, V_x L_y, V_x L_z, V_y L_x, V_y L_y, V_y L_z, V_z L_x, V_z L_y, V_z L_z, V_x, V_y, V_z, L_x, L_y, L_z)$ [S1, unofficial].
    *   *Note: The source text refers to it as a 12-element vector, but explicitly lists 13 components in the formula [S1, unofficial].*
*   **DN Vector:** A tuple of 13 elements representing products of depth vector components $D$ and orientation components $N$ [S1, unofficial].
    *   Layout: $(D_x N_x, D_x N_y, D_x N_z, D_y N_x, D_y N_y, D_y N_z, D_z N_x, D_z N_y, D_z N_z, D_x, D_y, D_z, N_x, N_y, N_z)$ [S1, unofficial].

**Formulas:**
The rendered intensity $I$ is calculated as:
$$ I = \frac{\tau \pi}{2} VL \cdot \Pi_V \left[ \left(\frac{1}{2} + \frac{1}{2} DN(x,y,z)\right) I_s \Omega(x,y,z) \right] + \frac{\tau \pi}{2} c_3 \Pi_V [I_s \Omega(x,y,z)] $$
Where:
*   $I_s$: Intensity of the light source [S1, unofficial].
*   $\tau$: Attenuation coefficient parameter [S1, unofficial].
*   $\Pi_V$: Fourier projection operator along viewing direction $V$ [S1, unofficial].
*   $c_3$: Constant term defined as $c_3 = \frac{1}{2} - \frac{1}{2} \sum_i VL_i$ [S2, unofficial].
*   $VL \cdot \Pi_V [\dots]$ denotes the dot product between the $VL$ vector and the corresponding vector of projected spectra [S1, unofficial].

**Steps:**
1.  **Precomputation (Offline):**
    1.  Expand the integrand product $(\frac{1}{2} + \frac{1}{2} N \cdot L)(\frac{\tau}{2} + \frac{\tau}{2} D \cdot V)$ into 16 basis terms: $1$, $N_i$ (3 terms), $D_j$ (3 terms), and $N_i D_j$ (9 terms) [S1, unofficial].
    2.  Create 16 basis volumes by multiplying $I_s \Omega(x,y,z)$ by each basis term [S1, unofficial].
    3.  Compute the 3D Fourier Transform of each basis volume to yield 16 precomputed spectra [S1, unofficial].
2.  **Per-View Rendering (Online):**
    1.  For the current viewing direction $V$ and lighting direction $L$, extract 2D slices from each of the 16 precomputed 3D spectra along $V$ [S1, unofficial].
    2.  Compute the 2D Inverse Fourier Transform of each extracted slice to obtain spatial domain images [S1, unofficial].
    3.  Weight each image by coefficients derived from $L$ and $V$, and sum them according to the expanded formula [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def fft3d(vol):
    # Standard 3D Fast Fourier Transform
    return np.fft.fftn(vol)

def ifft2d(slice_2d):
    # Standard 2D Inverse Fast Fourier Transform
    return np.fft.ifft2(slice_2d)

def slice_spectrum(spec_3d, dir_v):
    # Extract 2D slice from 3D spectrum along direction dir_v
    # Standard operation in Fourier projection-slice rendering
    return extract_slice_along_vector(spec_3d, dir_v)

def composite_depth_cueing_shading(volume, orientation, depth_field, viewing_dir, lighting_dir, tau, light_intensity):
    """
    Renders a volume using composite depth cueing and directional shading.
    
    Args:
        volume: 3D array of albedo Omega(x,y,z)
        orientation: 3D array of orientation field N(x,y,z)
        depth_field: 3D array of depth vector field D(x,y,z)
        viewing_dir: 3D vector V
        lighting_dir: 3D vector L
        tau: float, attenuation parameter
        light_intensity: float, Is
        
    Returns:
        2D array, rendered image
    """
    # Precomputation (Assumed to be done once or cached)
    # Basis functions for the 16 spectra based on expansion of (1/2 + 1/2 N.L) * (tau/2 + tau/2 D.V)
    # Terms: 1, N_i (3), D_j (3), N_i*D_j (9)
    basis_volumes = []
    
    # Base albedo (1)
    basis_volumes.append(light_intensity * volume)
    
    # N components (3)
    basis_volumes.append(light_intensity * volume * orientation[:,:,0]) # Nx
    basis_volumes.append(light_intensity * volume * orientation[:,:,1]) # Ny
    basis_volumes.append(light_intensity * volume * orientation[:,:,2]) # Nz
    
    # D components (3)
    basis_volumes.append(light_intensity * volume * depth_field[:,:,0]) # Dx
    basis_volumes.append(light_intensity * volume * depth_field[:,:,1]) # Dy
    basis_volumes.append(light_intensity * volume * depth_field[:,:,2]) # Dz
    
    # Cross terms N_i * D_j (9)
    for i in range(3):
        for j in range(3):
            basis_volumes.append(light_intensity * volume * orientation[:,:,i] * depth_field[:,:,j])
            
    # Compute 3D FFT for each basis volume
    spectra = [fft3d(bv) for bv in basis_volumes]
    
    # Per-view rendering
    V = viewing_dir
    L = lighting_dir
    
    # Extract slices from spectra along V
    slices = [slice_spectrum(spec, V) for spec in spectra]
    
    # Inverse FFT to get spatial images
    images = [ifft2d(s) for s in slices]
    
    # Coefficients from expanded integrand: I_s * Omega * (pi*tau/4) * [1 + sum(L_i N_i) + sum(V_j D_j) + sum(L_i V_j N_i D_j)]
    coeff_base = (tau * np.pi) / 4
    coeff_n = [(tau * np.pi) / 4 * L[i] for i in range(3)]
    coeff_d = [(tau * np.pi) / 4 * V[i] for i in range(3)]
    coeff_nd = [[(tau * np.pi) / 4 * L[i] * V[j] for j in range(3)] for i in range(3)]
    
    # Map images to coefficients and sum
    # Image 0: Base
    # Image 1-3: N_i
    # Image 4-6: D_j
    # Image 7-15: N_i * D_j
    image_sum = images[0] * coeff_base
    for i in range(3):
        image_sum += images[1+i] * coeff_n[i]
    for i in range(3):
        image_sum += images[4+i] * coeff_d[i]
    for i in range(3):
        for j in range(3):
            image_sum += images[7 + i*3 + j] * coeff_nd[i][j]
            
    return image_sum
```

**Edge Cases and Pitfalls:**
*   **Emission:** The composite model ignores emission $\rho C$ from the depth cueing equation. In this context, emission can be interpreted as ambient illumination [S1, unofficial].
*   **Memory:** The requirement for 16 spectra can lead to significant memory usage, potentially causing paging on older systems [S1, unofficial].

## Subtopics

### [Combined Depth Cueing and Shading](1170-combined-depth-cueing-and-shading.md)
This method combines linear depth cueing and directional shading within the frequency domain to replace $O(n^3)$ spatial domain computations with $O(n^2)$ frequency domain operations. It solves the problem of efficiently applying both attenuation and illumination in Fourier Projection-Slice Volume Rendering [Parent]. It sets itself apart from naive frequency domain combinations by reversing the standard order of operations (applying depth cueing and interpolation first, then shading) to utilize a single composite filter, thereby avoiding the computational cost of multiple convolutions.

## Operating characteristics
*   **Time Complexity:** Asymptotically less expensive than conventional volume rendering ($O(n^2)$ vs $O(n^3)$), but with a high constant factor due to the 16 spectra and 16 inverse FFTs per view [S1, unofficial].
*   **Memory Complexity:** Requires storing 16 3D spectra. For a $128^3$ voxel cube, this amounts to 256 megabytes [S1, unofficial].
*   **Measured Performance:** On a 36 MHz Silicon Graphics 4D/320 VGX, rendering time was over a minute per view for a $128^2$ image, mostly due to paging [S1, unofficial].
*   **Strengths:** Efficiently combines two shading effects; avoids nonlinear operators; plausible shading for diffuse objects resembling a cloudy day appearance [S1, unofficial].
*   **Limitations:** High constant factor makes it slower than conventional rendering for small volumes; sensitive to memory constraints; ignores emission/ambient light [S1, unofficial].
*   **When to Use:** Large volumes where asymptotic efficiency outweighs constant overhead; scenes requiring both depth cueing and directional shading [S1, unofficial].
*   **When Not to Use:** Small volumes; hardware with limited memory; scenes requiring accurate ambient lighting or emission [S1, unofficial].

## Background
The method was developed by Marc Levoy as part of his work on Fourier Projection-Slice Volume Rendering [S1, unofficial]. It appears in the paper "Volume Rendering using the Fourier Projection-Slice Theorem" [S1, unofficial]. The technique was demonstrated on a CT scan of a human skull [S1, unofficial]. The approach was implemented in C on a Silicon Graphics 4D/320 VGX [S1, unofficial]. The paper suggests that replacing the FFT with an FHT (Fast Hartley Transform) could halve the rendering time and memory requirements [S1, unofficial].

## Sources
[S1] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
