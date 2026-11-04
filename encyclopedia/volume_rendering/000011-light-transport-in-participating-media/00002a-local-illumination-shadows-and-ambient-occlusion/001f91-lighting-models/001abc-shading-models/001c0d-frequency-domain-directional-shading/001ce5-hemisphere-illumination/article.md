# Hemisphere Illumination

Also called: Directional Shading (Hemisphere Illumination), Directional Shading via Hemispherical Illumination, Frequency Domain Directional Shading (Hemisphere Illumination)

## Overview
Hemisphere Illumination is a local shading model for volume rendering that simulates diffuse reflection from a virtual hemisphere light source to provide directional illumination cues in volume projections [S1, unofficial]. It enhances the perception of shape and spatial relationships by weighting volume points based on their orientation relative to a light direction, effectively treating the volume as a collection of diffusely reflecting surfaces [S1, unofficial]. Unlike standard volume rendering which often lacks explicit shading cues, this method uses a hemisphere light source to provide directional illumination, though it remains a local approximation without global occlusion [S1, unofficial]. It builds on [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md) by computing directional shading in the frequency domain using an orientation field [S2, unofficial].

## How it works
**Inputs:**
- Volume data $\Omega(x,y,z)$ (albedo/density) [S2, unofficial].
- Orientation field $N(x,y,z) = (N_x, N_y, N_z)$, typically derived from the gradient of the volume data [S2, unofficial].
- Lighting direction $L = (L_x, L_y, L_z)$, a unit vector [S2, unofficial].
- Source intensity $I_s$ (constant) [S2, unofficial].
- Viewing direction $V$ (normalized, pointing away from the eye) [S2, unofficial].

**Outputs:**
- Rendered image intensity $I$ [S2, unofficial].

**Data Structures:**
- The volume $\Omega$ and orientation field $N$ are represented in the frequency domain [S2, unofficial].
- Four 3D spectra are computed during precomputation [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute(volume_omega, orientation_N, source_intensity):
    # Create four copies of Is * Omega
    spec_x = fft3d(source_intensity * volume_omega * orientation_N.x)
    spec_y = fft3d(source_intensity * volume_omega * orientation_N.y)
    spec_z = fft3d(source_intensity * volume_omega * orientation_N.z)
    spec_c = fft3d(source_intensity * volume_omega)
    return spec_x, spec_y, spec_z, spec_c

def render(spec_x, spec_y, spec_z, spec_c, viewing_direction_V, lighting_direction_L):
    # Extract slices along viewing direction V
    slice_x = extract_slice(spec_x, V)
    slice_y = extract_slice(spec_y, V)
    slice_z = extract_slice(spec_z, V)
    slice_c = extract_slice(spec_c, V)
    # Compute constant c2
    c2 = pi/2 - pi/2 * (L.x + L.y + L.z)
    # Combine slices
    intensity = L.x * slice_x + L.y * slice_y + L.z * slice_z + c2 * slice_c
    return intensity
```

**Steps:**
1. **Precomputation:** Create four copies of the product $I_s \Omega(x,y,z)$ [S2, unofficial]. Multiply the first copy by $N_x(x,y,z)$, the second by $N_y(x,y,z)$, the third by $N_z(x,y,z)$, and leave the fourth copy as $I_s \Omega(x,y,z)$ [S2, unofficial]. Compute the 3D Fourier Transform of each of the four copies to yield four 3D spectra [S2, unofficial].
2. **Rendering (for a given view and light direction):** Extract the relevant slices from the four 3D spectra corresponding to the viewing direction $V$ [S2, unofficial]. Compute the image intensity $I$ using the linear combination of the extracted slices [S2, unofficial]:

   $$
   I = L_x \Pi_V \left[ \left(\frac{\pi}{2} + \frac{\pi}{2} N_x\right) I_s \Omega \right] + L_y \Pi_V \left[ \left(\frac{\pi}{2} + \frac{\pi}{2} N_y\right) I_s \Omega \right] + L_z \Pi_V \left[ \left(\frac{\pi}{2} + \frac{\pi}{2} N_z\right) I_s \Omega \right] + c_2 \Pi_V \left[ I_s \Omega \right]
   $$

   Where:
   - $\Pi_V$ denotes the projection-slice operation along viewing direction $V$ [S2, unofficial].
   - $c_2 = \frac{\pi}{2} - \frac{\pi}{2} (L_x + L_y + L_z)$ [S2, unofficial].

**Formula Derivation Context:**
The weighting assigned to each point $P(t)$ along a ray is based on the integral of the source intensity modulated by the diffuse reflection model [S2, unofficial]:

$$
I_P = \int_a^b I_s \Omega(P(t)) \left[ \frac{\pi}{2} + \frac{\pi}{2} (N(P(t)) \cdot L) \right] dt
$$

This integral is decomposed into the linear combination shown above to allow efficient computation via the Fourier Projection-Slice Theorem [S2, unofficial].

**Parameters:**
- **Light Hemisphere Direction:** Typically placed on the right side of the picture, shining leftwards [S1, unofficial]. This corresponds to a lighting direction $L$ with a negative x-component in a standard right-handed coordinate system where the viewer looks down the negative z-axis or similar convention [S1, unofficial].
- **Viewing Direction:** $V$ points away from the eye [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Padding:** To avoid aliasing in the frequency domain, the input volume may need to be zero-padded. Increasing the volume width by a factor of $k^2$ (surrounding data with a margin of zeros of width $N(k^2-1)/2$) increases per-view processing time by a factor of $k^2$, memory by $k^3$, and preprocessing time by a factor of $k^2$ [S2, unofficial].
- **Orientation Field:** The accuracy of the shading depends on the quality of the orientation field $N$ [S2, unofficial].
- **No Occlusion:** The method does not account for occlusion between volume elements [S1, unofficial].

## Subtopics

### [Global Illumination via Hemisphere Sampling](1982-global-illumination-via-hemisphere-sampling.md)
A global illumination solution that models single-bounce light contribution from the sky by creating a virtual skybox of infinite directional lights and sampling it over the hemisphere directed away from the surface normal. Unlike typical raster renderers that model the sky as one or more directional lights (causing inaccurate shadowing and dark scenes), this method provides accurate shadowing and illumination by integrating radiance over the hemisphere.

## Operating characteristics
- **Time Complexity:** Preprocessing time is proportional to the size of the zero-padded volume. Per-view processing time increases with the amount of zero padding [S2, unofficial].
- **Memory Complexity:** Memory requirements increase with the size of the zero-padded volume [S2, unofficial].
- **Strengths:** Provides strong directional shading cues that enhance shape perception and spatial relationships in volume data [S1, unofficial]. Computed efficiently in the frequency domain via 3D Fourier transforms [S2, unofficial].
- **Limitations:** Does not model occlusion or global illumination effects like inter-reflections [S1, unofficial]. Accuracy depends on the quality of the orientation field [S2, unofficial].
- **When to Use:** When directional shading cues are needed to improve shape perception in volume renderings, and computational efficiency is important [S1, unofficial].
- **When Not to Use:** When accurate shadows, occlusion, or global illumination effects are required [S1, unofficial].

## Background
The method was described in the context of volume rendering using the Fourier Projection-Slice Theorem [S1, unofficial]. It was developed to generate realistically shaded renderings of volume data by simulating diffuse reflection from a hemisphere light source [S1, unofficial]. The approach leverages the efficiency of frequency domain computations to apply complex shading models to volume data [S2, unofficial].

## Sources
[S1] Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
