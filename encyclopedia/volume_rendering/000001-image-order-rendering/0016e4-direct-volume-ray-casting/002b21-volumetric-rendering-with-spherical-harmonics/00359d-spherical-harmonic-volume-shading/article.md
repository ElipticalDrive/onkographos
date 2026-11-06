# Spherical Harmonic Volume Shading

Also called: SH Volume Shading, SH Volumes, Spherical Harmonic Approximation for Shading, Spherical Harmonic Volumes, Spherical Harmonics Shading in Fourier Domain

## Overview
Spherical Harmonic Volume Shading is a method for computing diffuse illumination in Fourier Volume Rendering (FVR) that handles the nonlinearity of the shading function (specifically the $\max(\mathbf{N} \cdot \mathbf{L}, 0)$ term) without requiring repeated 3D Fourier transforms for every light direction change [S1, unofficial]. It approximates the illumination and shading functions using spherical harmonics, allowing the shading equation to be expressed as a linear convolution in spherical coordinates [S1, unofficial]. Unlike linear or cubic illumination models which require slicing only two volumes but perform complex calculations in the inner loop, this method slices multiple volumes (typically 10) to assemble one image, offering a trade-off between memory usage and the ability to interactively change lighting [S1, unofficial].

## How it works
**Inputs:**
* Volumetric dataset $f(x, y, z)$ [S1, unofficial].
* Directional light source direction $\mathbf{L}$ (unit vector) [S2].
* Truncation order $M$ for spherical harmonics (typically $l=0, 1, 2$) [S1, unofficial].

**Outputs:**
* Shaded volume $E(x, y, z)$ or its Fourier transform $F(E)$ [S1, unofficial].
* Rendered 2D slice via inverse Fourier transform [S1, unofficial].

**Data Structures:**
* **Volume Data:** 3D grid of scalar values $f(x,y,z)$ [S1, unofficial].
* **Normal Volumes:** 3D grids storing the components of the normalized gradient vector $\mathbf{N} = (n_x, n_y, n_z)$ for each voxel [S1, unofficial].
* **Harmonic Coefficients:** Pre-computed Fourier transforms of the spherical harmonic basis functions evaluated at the normal vectors, denoted $F(Y_{lm}(n_x, n_y, n_z))$ [S1, unofficial].
* **Light Coefficients:** Scalar values derived from the light direction $\mathbf{L}$ and spherical harmonic coefficients of the light [S2].

**Steps:**

1. **Compute Per-Voxel Normals:**
   Calculate the gradient of the volume data $f(x, y, z)$ using a gradient operator (e.g., central differences) [S1, unofficial]. Normalize the resulting vector to obtain the unit surface normal $\mathbf{N}(x, y, z)$ for each voxel [S1, unofficial]:
   $$
   \mathbf{N}(x, y, z) = \frac{\nabla f(x, y, z)}{|\nabla f(x, y, z)|} = n_x \hat{i} + n_y \hat{j} + n_z \hat{k}
   $$
   [G] Guard against zero-length gradients (e.g., in uniform regions) by setting the normal to $(0,0,1)$ or skipping shading for those voxels to avoid division by zero.

2. **Pre-compute Fourier Transforms of Harmonic Normals:**
   The terms $Y_{lm}(n_x, n_y, n_z)$ depend only on the geometry (normals) and not on the light direction [S1, unofficial]. Compute the 3D Fourier transform of these terms for all $l, m$ in the truncated series [S1, unofficial]. Store these transformed terms for reuse [S1, unofficial]. This step is performed once per volume dataset [S1, unofficial].

3. **Compute Light Coefficients:**
   For a given directional light $\mathbf{L}$, compute the spherical harmonic coefficients $L_{lm}$ [S1, unofficial]. For a directional light, the light function is a delta function in direction, simplifying the coefficients [S2]. The method uses real spherical harmonics to save memory and speed up transforms [S1, unofficial].

4. **Combine in Frequency Domain:**
   Compute the Fourier transform of the shaded volume $F(E)$ using the linear combination of the pre-computed harmonic volumes weighted by the light coefficients [S1, unofficial]. The rendering equation in the Fourier domain is [S1, unofficial]:
   $$
   F(E) = \frac{|L|}{\pi} \sum_{l=0}^{M} \sum_{m=-l}^{l} L_{lm} a_{l0} F(Y_{lm}(n_x, n_y, n_z))
   $$
   Where $a_{l0}$ combines normalization and reflection coefficients [S1, unofficial]. Note: The source text uses $|U|$ or $|L|$ interchangeably for light magnitude; for a unit directional light, $|L|=1$. The term $\frac{1}{\pi}$ accounts for the diffuse reflectance $\rho = 1/\pi$ [S1, unofficial].

5. **Reconstruct Slice:**
   To render a specific 2D slice (e.g., for ray casting or direct volume rendering), extract the relevant frequency components for that slice from the 3D Fourier volume $F(E)$ [S1, unofficial]. Reconstruct the individual harmonic Fourier terms for all $l$ and $m$ planes corresponding to the desired 2D slice, then produce the 2D shaded slice via inverse Fourier transform [S1, unofficial].

**Formulas:**

* **Real Spherical Harmonics Basis:**
  $$
  Y_{lm}(\theta, \phi) = \sqrt{\frac{(2l+1)(l-|m|)!}{4\pi(l+|m|)!}} P'_m(\cos \theta) \begin{cases} \cos m\phi & \text{for } m \geq 0 \\ \sin |m|\phi & \text{for } m < 0 \end{cases}
  $$
  Where $P'_m$ are the associated Legendre polynomials [S1, unofficial].

* **Spherical Harmonic Approximation:**
  Any function $f(\theta, \phi)$ on the sphere is approximated by:
  $$
  f(\theta, \phi) = \sum_{l=0}^{\infty} \sum_{m=-l}^{l} f_{lm} Y_{lm}(\theta, \phi)
  $$
  With coefficients:
  $$
  f_{lm} = \int_0^{2\pi} \int_0^{\pi} f(\theta, \phi) Y_{lm}(\theta, \phi) \sin \theta d\theta d\phi
  $$
  [S1, unofficial]

* **Diffuse Shading Equation (Spatial):**
  The diffuse radiance $E$ is a convolution in spherical coordinates:
  $$
  E = \int_{\Omega} L(\omega) \rho(\omega, \omega') \max(\cos \theta, 0) d\omega
  $$
  With $\rho = 1/\pi$ [S1, unofficial].

* **Final Rendering Equation (Simplified):**
  After applying rotation properties and orthogonality, the shaded radiance for a voxel with normal $\mathbf{N}$ (spherical coords $\alpha, \beta$) and light direction $\mathbf{L}$ (spherical coords $\theta, \Phi$) is:
  $$
  E = \frac{|L|}{\pi} \sum_{l=0}^{M} \sum_{m=-l}^{l} \sqrt{\frac{4\pi}{2l+1}} L_{lm} c_{l0} Y_{lm}(\alpha, \beta)
  $$
  Where the rotation coefficient identity $D_{m0}^l(\alpha) e^{im\beta} = \sqrt{\frac{4\pi}{2l+1}} Y_{lm}(\alpha, \beta)$ is used [S1, unofficial].

**Parameters:**
* **Truncation Order:** $l=0, 1, 2$ (9 planes). This captures over 98% of the image energy [S1, unofficial].
* **Ambient Term:** 0.4 [S1, unofficial].
* **Harmonic Type:** Real spherical harmonics (to save memory and speed up FFTs) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.special import lpmv

def compute_gradients(volume):
    """Compute normalized gradient vectors for each voxel."""
    fx = np.gradient(volume, axis=0)
    fy = np.gradient(volume, axis=1)
    fz = np.gradient(volume, axis=2)
    mag = np.sqrt(fx**2 + fy**2 + fz**2)
    mag[mag == 0] = 1e-8
    return fx / mag, fy / mag, fz / mag

def spherical_harmonic_value(l, m, theta, phi):
    """Evaluate real spherical harmonic Y_lm at given spherical angles."""
    cos_theta = np.cos(theta)
    P_lm = lpmv(m, l, cos_theta)
    norm_factor = np.sqrt(((2*l + 1) * np.math.factorial(l - abs(m))) / 
                          (4 * np.pi * np.math.factorial(l + abs(m))))
    if m >= 0:
        angular = np.cos(m * phi)
    else:
        angular = np.sin(abs(m) * phi)
    return norm_factor * P_lm * angular

def precompute_harmonic_volumes(volume, max_l=2):
    """Pre-compute the 3D FFT of spherical harmonic basis functions evaluated at voxel normals."""
    nx, ny, nz = compute_gradients(volume)
    theta = np.arccos(np.clip(nz, -1.0, 1.0))
    phi = np.arctan2(ny, nx)
    
    harmonic_ffts = {}
    for l in range(max_l + 1):
        for m in range(-l, l + 1):
            Y_lm_voxel = spherical_harmonic_value(l, m, theta, phi)
            harmonic_ffts[(l, m)] = np.fft.fftn(Y_lm_voxel)
    return harmonic_ffts

def compute_light_coefficients(light_direction, max_l=2):
    """Compute spherical harmonic coefficients for a directional light source."""
    lx, ly, lz = light_direction
    theta_l = np.arccos(np.clip(lz, -1.0, 1.0))
    phi_l = np.arctan2(ly, lx)
    
    coeffs = {}
    for l in range(max_l + 1):
        for m in range(-l, l + 1):
            coeffs[(l, m)] = spherical_harmonic_value(l, m, theta_l, phi_l)
    return coeffs

def render_shaded_volume(volume, light_direction, max_l=2):
    """Render the shaded volume using Spherical Harmonic Volume Shading."""
    harmonic_ffts = precompute_harmonic_volumes(volume, max_l)
    light_coeffs = compute_light_coefficients(light_direction, max_l)
    
    shaded_fft = np.zeros_like(volume, dtype=complex)
    
    for l in range(max_l + 1):
        for m in range(-l, l + 1):
            L_lm = light_coeffs[(l, m)]
            # c_l0 coefficients for diffuse hemisphere (max(cos,0))
            if l % 2 != 0:
                c_l0 = 0.0
            elif l == 0:
                c_l0 = np.sqrt(np.pi)
            elif l == 2:
                c_l0 = np.sqrt(5 * np.pi) / 2
            else:
                c_l0 = 0.0
                
            # c'_l0 = (4pi / (2l+1)) * c_l0
            c_prime_l0 = (4 * np.pi / (2 * l + 1)) * c_l0
            
            shaded_fft += L_lm * c_prime_l0 * harmonic_ffts[(l, m)]
            
    shaded_fft = shaded_fft / np.pi
    shaded_volume = np.fft.ifftn(shaded_fft).real
    return shaded_volume
```

## Subtopics

### Spherical Harmonics
An orthonormal basis on the unit sphere, analogous to Fourier basis functions in spherical coordinates. They allow functions to be approximated by a series expansion. [Link](5603-spherical-harmonics.md)

### Ray Tracing Backward Pass
A different shading technique that computes gradients for optimization, not directly related to the frequency-domain convolution approach of SH Volumes. [Link](5966-ray-tracing-backward-pass.md)

## Operating characteristics
* **Time Complexity:** The pre-computation of harmonic volumes is $O(N^3 \log N)$ due to the 3D FFTs, where $N$ is the volume dimension. Rendering a frame involves a linear combination of these pre-computed volumes ($O(N^3)$) and a final 3D IFFT ($O(N^3 \log N)$) if the whole volume is needed, or 2D IFFT for slices.
* **Memory Complexity:** Requires storing $M^2$ volumes (where $M$ is the number of harmonic planes). For $l=0,1,2$, this is 9-10 volumes (including the original data and normals). This is significantly higher than linear/cubic illumination which only requires 2 volumes [S1, unofficial].
* **Measured Performance:** On a synthetic sphere dataset, the method achieved 8 fps without depth cueing and 6 fps with depth cueing [S1, unofficial]. This is slower than the cubic illumination model (35 fps) because it slices 10 volumes instead of 2 [S1, unofficial]. Traditional FVR without shading achieved 36 fps [S1, unofficial].
* **Strengths:** Allows real-time interactive changes to light direction. Handles the nonlinearity of the $\max(\mathbf{N} \cdot \mathbf{L}, 0)$ function accurately.
* **Limitations:** High memory usage due to multiple harmonic volumes. Slower rendering speed compared to simpler illumination models due to the number of volumes to slice.
* **When to use:** When interactive lighting changes are required and memory is not a severe constraint.
* **When not to use:** When maximum rendering speed is critical and lighting is static (use cubic illumination instead).

## Background
The method was introduced by researchers at the Scientific Computing and Imaging Institute, building on Fourier Volume Rendering (FVR) by Totsuka and Levoy. It specifically addresses the nonlinearity of shading in the Fourier domain, a problem previously tackled with hemispherical reflections or cubic illumination. The use of spherical harmonics for shading was pioneered by Ramamoorthi and Hanrahan (SIGGRAPH 2001) [S1, unofficial]. The method was presented in a paper titled "Fourier Volume Shading" (Vis02) [S1, unofficial].

## Sources
[S1] Fourier Volume Shading, https://www.sci.utah.edu/~jmk/papers/others/vis02_forierVolumeShading.pdf, fetched 2026-10-27 (unofficial)
[S2] Shading For Fourier Volume Rendering, https://eprints.cs.univie.ac.at/5034/1/2002_-_shading_for_fourier.pdf, fetched 2026-10-27
