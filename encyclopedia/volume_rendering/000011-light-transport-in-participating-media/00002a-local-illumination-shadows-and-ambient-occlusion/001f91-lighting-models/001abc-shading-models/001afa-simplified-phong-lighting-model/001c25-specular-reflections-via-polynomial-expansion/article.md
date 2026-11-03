# Specular Reflections via Polynomial Expansion

Also called: Fourier Projection-Slice Phong, Polynomial Phong Shading, Specular Reflections via Polynomial Approximation

## Overview
This method extends the Fourier Projection-Slice volume rendering technique to support Phong specular reflections by expanding the specular term $|N \cdot H|^n$ into a polynomial using hemispherical illumination [S1]. It solves the problem of computing view-dependent specular highlights in participating media without per-voxel shading, allowing the specular contribution to be evaluated via linear combinations of precomputed Fourier spectra [S1]. It differs from [Phong Shading](1853-phong-shading.md) by deferring the dot product evaluation to the projection domain, and from [Blinn-Phong Shading Model](2239-blinn-phong-shading-model.md) by using the exact Phong half-vector formulation expanded via the multinomial theorem [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume dataset $I_\Omega$, viewing direction $V$, light direction $L$, surface normal field $N$, specular exponent $n$ [S1, S2, unofficial].
- Output: Rendered image $I$ [S2, unofficial].

**Data Structures:**
- Precomputed Spectra: A set of 3D Fourier transforms of the volume data, indexed by the polynomial terms of the expansion [S1]. The number of spectra $K$ is determined by the multinomial theorem [S1].
- Polynomial Coefficients: Vectors $H^n$ and $N^n$ containing the combinatorial factors from the expansion of $(N \cdot H)^n$ [S2, unofficial].

**Steps:**
1. **Polynomial Expansion:** Expand the term $(N \cdot H)^n$ into a sum of monomials using the multinomial theorem [S1, S2, unofficial]. Let $H^n$ be the vector of coefficients for the view/light dependent terms and $N^n$ be the vector of coefficients for the normal dependent terms [S2, unofficial].
2. **Precomputation:** Compute the Fourier projection-slice spectra for the volume [S1, S2, unofficial]. For each monomial term in the expansion, a specific linear combination of the volume's Fourier transform is precomputed [S1]. The number of required spectra equals the number of terms in the expansion [S1].
3. **Per-View Rendering:**
   a. Extract the necessary slices from the precomputed spectra corresponding to the current view $V$ and light $L$ [S2, unofficial].
   b. Apply the Inverse Fourier Transform to these slices to obtain spatial domain images [S2, unofficial].
   c. Weight these images by the corresponding components of $H^n$ and $N^n$ [S2, unofficial].
   d. Sum the weighted images to produce the final specular contribution [S2, unofficial].

**Formulas:**
The integral for a viewing ray $P(t)$ is defined as [S1, S2, unofficial]:
$$ I_P = \int_a^b I_\Omega(P(t)) \left[ \frac{\pi}{2} + \frac{\pi}{2} (N(P(t)) \cdot H)^n \right] dt $$
where $H$ is the halfway vector between $L$ and $V$, and $n$ is the specular exponent [S1, S2, unofficial].

After factoring and applying the Fourier projection operator $\Pi_V$, the output image $I$ is [S1, S2, unofficial]:
$$ I = H^n \cdot \Pi_V \left[ \left(\frac{\pi}{2} + \frac{\pi}{2} N^n(x,y,z)\right) I_\Omega(x,y,z) \right] + c_4 \Pi_V [I_\Omega(x,y,z)] $$
where $c_4$ is a constant derived from the expansion, and $H^n, N^n$ contain the list of factors in the polynomial $(N \cdot H)^n$ [S1, S2, unofficial].

**Parameters:**
- $n$: Specular exponent. Typical value is 10 [S1].

**Edge Cases and Pitfalls:**
- Boundaries: The method assumes hemispherical illumination to avoid the absolute value operator in $|N \cdot H|$ [S1].
- Scale: The number of required spectra grows combinatorially with $n$, quickly exceeding practical limits for large exponents [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_multinomial_factors(n):
    """Compute combinatorial factors for the multinomial expansion of (N.H)^n."""
    H_n = []
    N_n = []
    for k1 in range(n + 1):
        for k2 in range(n + 1 - k1):
            k3 = n - k1 - k2
            coeff = math.factorial(n) // (math.factorial(k1) * math.factorial(k2) * math.factorial(k3))
            H_n.append(coeff)
            N_n.append(coeff)
    return H_n, N_n

def render_specular_polynomial(precomputed_spectra, V, L, n):
    """Render specular reflections using polynomial expansion of the Phong term."""
    # 1. Determine polynomial terms
    K = (n + 1) * (n + 2) * (n + 3) // 6
    H_n, N_n = compute_multinomial_factors(n)

    # 2. Per-view rendering
    result = np.zeros_like(precomputed_spectra[0].data)
    for i in range(K):
        # Extract slice from the i-th precomputed spectrum along view V
        slice_data = precomputed_spectra[i].extract_slice(V)
        # Inverse Fourier transform to spatial domain
        spatial_img = np.fft.ifftn(slice_data).real
        # Weight by corresponding components of H_n and N_n
        weight = H_n[i] * N_n[i]
        result += weight * spatial_img

    # Add constant term c_4 * projection of volume
    c_4 = 0.5 - 0.5 * sum(N_n)
    constant_proj = np.fft.ifftn(precomputed_spectra[0].extract_slice(V)).real
    result += c_4 * constant_proj

    return result
```

## Subtopics

### Polynomial Depth Cueing
Polynomial Depth Cueing is an adaptation of the specular reflection method to provide non-linear depth cueing, such as inverse square law falloff, by replacing the normal $N$ and halfway vector $H$ with the depth vector $D$ and view vector $V$ in the polynomial expansion [S2, unofficial]. For an inverse square law falloff using $(D \cdot V)^2$, 10 slices from 10 precomputed spectra are required [S2, unofficial].

### [Simple Reflections](1234-simple-reflections.md)
Simple Reflections is a local illumination technique that calculates a reflected ray direction off a surface normal and raymarches along it, blending the result with the surface color.

## Operating characteristics
- **Time Complexity:** The rendering time is proportional to the number of precomputed spectra, which grows combinatorially with $n$ according to the multinomial theorem [S1].
- **Memory Complexity:** Requires storing $K$ precomputed spectra, where $K$ is the number of terms in the polynomial expansion [S1].
- **Measured Performance:** For $n=10$, 286 precomputed spectra are needed, making this extension slower than conventional volume rendering even for very large volumes [S1].
- **Strengths:** Allows view-dependent specular highlights in volume rendering without per-voxel shading [S1].
- **Limitations:** High memory and computational cost for large $n$; limited to polynomial approximations of the specular term [S1].
- **When to use:** When view-dependent specular highlights are critical and computational resources allow for precomputation of many spectra [G].
- **When not to use:** For real-time applications or when $n$ is large, due to the high number of required spectra [G].

## Background
This method extends the basic Fourier projection-slice volume rendering technique to include Phong-like shading models [S1]. The approach leverages the linearity of the Fourier projection operator to decompose the non-linear specular term into a sum of linear terms [S2, unofficial].

## Sources
[S1] Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
