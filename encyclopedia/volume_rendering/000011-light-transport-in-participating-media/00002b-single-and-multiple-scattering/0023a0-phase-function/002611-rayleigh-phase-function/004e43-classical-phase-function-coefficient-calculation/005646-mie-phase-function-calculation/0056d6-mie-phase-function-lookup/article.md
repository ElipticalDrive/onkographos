# Mie Phase Function Lookup

Also called: Mie Phase Function Image Lookup, Mie PF EXR Lookup

## Overview
This method provides a practical technique for evaluating the Mie phase function in real-time volume rendering, specifically for clouds and participating media containing microscopic water droplets. It addresses the computational cost of calculating Mie scattering analytically by precomputing the phase function values into a 1D texture (OpenEXR image) and sampling it via a dot-product-derived coordinate. It sets itself apart from analytical calculation methods by trading memory bandwidth for speed, and from generic phase function approximations (like Henyey-Greenstein) by preserving the specific diffraction and interference features of Mie theory, such as the strong forward peak and silver lining effect.

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   Eye direction vector $I$ (view direction).
    *   Light direction vector $L$ (direction to the light source).
    *   Precomputed scattering data derived from Antoine Bouthors’ thesis [S1, unofficial].
*   **Outputs:**
    *   Phase function value $p(\theta)$ for the given scattering angle.
    *   RGB color values if the phase function is spectrally resolved (though typically treated as luminance or grayscale in standard implementations, the source notes normalization over RGB channels).

### Data Structures
*   **Phase Function Storage:** A 1800 $\times$ 1 pixel OpenEXR image [S1, unofficial].
    *   **Format:** OpenEXR (high dynamic range).
    *   **Resolution:** 1800 pixels in the U (horizontal) dimension, 1 pixel in the V (vertical) dimension.
    *   **Content:** The scattering values of the combined Mie phase function.
    *   **Rationale:** VEX arrays in Houdini have size limits that cause compilation errors when storing the full 1800 values; the EXR image serves as a portable, efficient alternative [S1, unofficial].

### Steps
1.  **Preprocessing (Offline):**
    *   Retrieve the split scattering data files `Mie0.txt` and `MiePF3.txt` from Antoine Bouthors’ thesis [S1, unofficial]. These represent a general-purpose droplet size distribution found in clouds.
    *   Combine the split data (forward peak and rest) into a single set if importance sampling is not being used for the phase function itself. The split is intended for sampling efficiency, but for simple weighting, they are summed [S1, unofficial].
    *   Normalize the combined data so that it sums to 1.0 over the RGB channels [S1, unofficial].
    *   Generate the 1800 $\times$ 1 OpenEXR image containing these normalized values [S1, unofficial].

2.  **Runtime Evaluation:**
    *   Calculate the angle $\theta$ between the eye direction $I$ and the light direction $L$.
    *   Compute the U coordinate for texture sampling using the formula below.
    *   Sample the OpenEXR image at the computed U coordinate to retrieve the phase function value.

### Formulas
The U coordinate is calculated by normalizing the dot product of the eye and light directions, mapping the angle to the range $[0, 1]$ [S1, unofficial]:

$$
U = \frac{\arccos(\text{normalize}(I) \cdot \text{normalize}(L))}{\pi}
$$

Where:
*   $I$ is the eye direction vector.
*   $L$ is the light direction vector.
*   $\text{normalize}(v)$ returns the unit vector in the direction of $v$.
*   $\arccos$ returns the angle in radians.
*   $\pi$ is the mathematical constant pi.

### Parameters
*   **Data Source:** `Mie0.txt` and `MiePF3.txt` from Antoine Bouthors’ PhD thesis, *Realistic rendering of clouds in realtime* [S1, unofficial].
*   **Texture Resolution:** 1800 samples [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_mie_phase_function(exr_image, eye_dir, light_dir):
    """
    Samples the Mie phase function from a 1D EXR image.
    
    Args:
        exr_image: A 1D texture (1800x1) containing normalized Mie scattering data.
        eye_dir: Vector3, the view direction (I).
        light_dir: Vector3, the direction to the light source (L).
        
    Returns:
        float: The phase function value p(theta).
    """
    # 1. Normalize directions
    I = eye_dir.normalized()
    L = light_dir.normalized()
    
    # 2. Calculate cosine of the angle between I and L
    cos_theta = I.dot(L)
    
    # 3. Clamp to [-1, 1] to handle floating point errors
    cos_theta = max(-1.0, min(1.0, cos_theta))
    
    # 4. Calculate U coordinate: angle / pi
    # arccos returns radians in [0, pi]
    angle = math.acos(cos_theta)
    u = angle / math.pi
    
    # 5. Sample the EXR image
    # U is in [0, 1]. Assuming standard texture coordinates where 0 is left, 1 is right.
    # The EXR image is 1800 pixels wide.
    # Note: Exact sampling method (linear vs nearest) depends on the renderer.
    # For high precision, linear interpolation between texels is preferred.
    phase_value = exr_image.sample_1d(u)
    
    return phase_value
```

### Edge Cases and Pitfalls
*   **Array Size Limits:** In Houdini VEX, arrays of a handful of floats work, but full arrays of 1800 values cause compilation errors. Using an EXR image avoids this limitation [S1, unofficial].
*   **Forward Scattering Intensity:** The Mie phase function is heavily forward-scattering, with over 90% of energy scattered in the forward direction. This causes the volume to appear dark from angles other than directly in front of the light [S1, unofficial].
*   **Dynamic Range:** The intensity difference between the strong forward peak (silver lining) and backward/sideways scattering is extreme. High dynamic range (HDR) lighting and tone mapping (or gamma correction) are required to visualize the weaker scattering components without clipping the peak [S1, unofficial].
*   **Multiple Scattering:** Single scattering alone is insufficient for realistic cloud rendering. Multiple scattering must be added to fill out the overall light in the volume. A common approximation is to add a weak isotropic phase function, as higher-order scattering tends toward isotropy [S1, unofficial].

## Subtopics

### Mie Phase Function Calculation
The analytical computation of the phase function using Mie theory for homogeneous spheres. This is the source of the data used in the lookup method. [Mie Phase Function Calculation](15655-mie-phase-function-calculation.md)

### Classical Phase Function Coefficient Calculation
The broader context of calculating phase function coefficients, of which Mie theory is a specific, physically accurate case for spherical particles. [Classical Phase Function Coefficient Calculation](15654-classical-phase-function-coefficient-calculation.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The lookup involves a dot product, an arccos, and a texture sample. This is significantly faster than analytical Mie calculation, which involves infinite series summations.
*   **Memory Complexity:** $O(N)$ where $N=1800$. The storage is minimal (a single 1D texture).
*   **Strengths:**
    *   Physically accurate representation of cloud scattering (forward peak, diffraction effects).
    *   Fast evaluation suitable for real-time rendering.
    *   Portable across renderers (EXR is a standard format).
*   **Limitations:**
    *   Requires precomputation and storage of the texture.
    *   Does not inherently handle multiple scattering; requires additional techniques (e.g., isotropic fake) to appear realistic.
    *   The strong forward peak requires careful handling of HDR and tone mapping to avoid visual artifacts (darkness in non-forward directions).
*   **When to Use:** Real-time cloud rendering where physical accuracy of the single-scattering lobe is desired, and analytical Mie calculation is too slow.
*   **When Not to Use:** When multiple scattering is the dominant effect and a simple isotropic approximation is sufficient, or when memory bandwidth is extremely constrained and even a 1D texture lookup is prohibitive (though unlikely).

## Background
The method originates from practical real-time rendering of clouds, as described by the author of the blog post "Rendering clouds with the Mie phase function" [S1, unofficial]. It builds upon the work of Antoine Bouthors, whose PhD thesis *Realistic rendering of clouds in realtime* provided the Mie scattering data (`Mie0.txt`, `MiePF3.txt`) generated using Mieplot. The technique was developed to overcome the limitations of simpler phase functions (like Henyey-Greenstein) and the computational cost of analytical Mie calculation, specifically addressing the "silver lining" effect and the anisotropic nature of cloud scattering. The use of an EXR image was a pragmatic solution to Houdini VEX array size limitations [S1, unofficial].

## Sources
[S1] Rendering clouds with the Mie phase function, https://mattebb.com/blog/weblog/rendering-clouds-with-the-mie-phase-function/, fetched 2026-10-28 (unofficial)
