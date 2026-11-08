# Distance to Hypertextures

Also called: Fractal noise deformation, Hypertexture, Hypertexture noise

## Overview
Distance to Hypertextures is a technique for adding geometric relief and detail to surfaces without increasing polygon count by evaluating procedural noise within a volumetric region around the object rather than solely on the surface [S1, unofficial]. It deforms implicit surfaces by adding procedural noise to the function's range instead of the domain, enabling the use of sphere tracing with safe distance bounds [S2]. This method bridges the gap between surface textures and volumetric phenomena, allowing for the modeling of complex, organic details [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base implicit surface defined by a Signed Distance Function (SDF) $f(\mathbf{x})$, a noise function $noise(\mathbf{x})$, and spectral exponent $\alpha$ [S2].
*   **Output:** A deformed implicit surface $g(\mathbf{x})$ and its Lipschitz constant $L$ for sphere tracing [S2].

**Data Structures:**
*   **Noise Lattice:** An array $\Gamma$ of random unit vectors used for solid noise interpolation [S2].
*   **Octave Stack:** A sequence of noise evaluations at scaled frequencies for fractal noise [S2].

**Method Steps:**
1.  **Define Base SDF:** Establish the base implicit surface $f(\mathbf{x})$. For soft objects, this often uses a cubic Gaussian approximation or similar algebraic blend function [S2].
2.  **Compute Band-Limited Solid Noise:** Evaluate the noise function $noise(\mathbf{x})$ using the lattice $\Gamma$ and the interpolation kernel $\phi$ [S2]:
    $$noise(\mathbf{x}) = \sum_{i=-1}^{1} \sum_{j=-1}^{1} \sum_{k=-1}^{1} \Gamma(i,j,k) \cdot \phi(\mathbf{x} - (i,j,k))$$
    where $\phi$ is the cubic Gaussian approximation used for soft objects [S2].
3.  **Compute Fractal Noise (Optional):** If fractal detail is required, sum scaled versions of the noise function over $n$ octaves [S2]:
    $$noise_\alpha(\mathbf{x}) = \sum_{i=0}^{n} \frac{noise(2^i \mathbf{x})}{2^{i\alpha}}$$
4.  **Deform Surface:** The deformed implicit surface is defined by:
    $$g(\mathbf{x}) = f(\mathbf{x}) + noise(\mathbf{x})$$
    Negative keypoints can be incorporated by negating the value returned by the noise function [S2].
5.  **Calculate Lipschitz Constant:** Determine the Lipschitz constant $L$ for the composite function to ensure safe sphere tracing step sizes [S2].
    *   The Lipschitz constant of the base function $f$ (cubic Gaussian approx) is $Lip_f = \frac{3}{2}$ [S2].
    *   The Lipschitz constant of the noise lattice $\Gamma$ is $Lip_\Gamma = 2$ (due to opposing vectors being neighbors) [S2].
    *   The composition yields $Lip_{noise} = Lip_f \cdot Lip_\Gamma = 3$ [S2].
    *   For $1/f$ noise ($\alpha=1$), the Lipschitz constant is the sum of individual functions: $Lip_{1/f} = 3$ [S2].
    *   For Brownian motion ($\alpha=2$), $Lip_{B} = 3(2^{1-n} - 1) \approx 6$ [S2].

**Parameters:**
*   $\alpha$ (Spectral Exponent): Controls the frequency/amplitude scaling. $\alpha=0$ is white noise, $\alpha=1$ is $1/f$ noise, $\alpha=2$ is Brownian motion [S2].
*   $n$ (Octaves): Number of noise layers. Must be finite for $1/f$ noise to remain Lipschitz [S2].

**Edge Cases and Pitfalls:**
*   **Unbounded Noise:** Unbounded $1/f$ noise is not Lipschitz. It must be band-limited to a finite number of octaves $n$ to remain Lipschitz for sphere tracing [S2].
*   **Precision:** The Lipschitz constant calculation assumes specific properties of the base function and noise lattice; deviations may require conservative step sizing [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hypertexture_deformation(x, f_base, alpha, n_octaves, Gamma):
    # x: tuple of 3 floats
    # f_base: callable returning float for base SDF
    # alpha: spectral exponent (0, 1, or 2)
    # n_octaves: number of fractal octaves
    # Gamma: 3x3x3 list of floats representing random unit vector magnitudes
    
    def phi(d):
        # Cubic Gaussian approximation for soft objects [S2]
        return (2.0/3.0) * (3.0 - d**2)**3 if d < 1.0 else 0.0

    def compute_base_noise(x):
        noise_val = 0.0
        for i in range(-1, 2):
            for j in range(-1, 2):
                for k in range(-1, 2):
                    dist = ((x[0]-i)**2 + (x[1]-j)**2 + (x[2]-k)**2)**0.5
                    noise_val += Gamma[i+1][j+1][k+1] * phi(dist)
        return noise_val

    if alpha > 0:
        fractal_noise = 0.0
        for octave in range(n_octaves):
            freq = 2 ** octave
            amp = 1.0 / (2 ** (octave * alpha))
            octave_noise = compute_base_noise((x[0]*freq, x[1]*freq, x[2]*freq))
            fractal_noise += amp * octave_noise
        noise_val = fractal_noise
    else:
        noise_val = compute_base_noise(x)

    f_val = f_base(x)
    g_val = f_val + noise_val

    if alpha == 1:
        L = 3.0
    elif alpha == 2:
        L = 3.0 * (2**(1 - n_octaves) - 1)
    else:
        L = 3.0

    return g_val, L
```

## Subtopics

### On-the-Fly Distance Field & Hypertexture Lookup
A variant where the distance field is derived from the mesh on the fly on the GPU, using the surface vicinity for hypertexture lookups [S3, unofficial]. It aims for very high quality, using up to 20 octaves of Perlin noise for shading and complex materials [S3, unofficial]. Performance is limited by complex GPU computations, achieving around 20 FPS for a $1024^3$ volume [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** High computational cost due to the evaluation of multiple noise octaves and the base SDF at every ray step. On-the-fly variants can reach ~20 FPS for $1024^3$ volumes [S3, unofficial].
*   **Memory Complexity:** Depends on the noise lattice and octave storage. On-the-fly GPU implementations may use compacted textures (e.g., RGBA32) to store node indices, with memory usage scaling with the node pool size (e.g., 4 MB for $64^3$ entries) [S3, unofficial].
*   **Strengths:** Adds significant geometric detail without increasing polygon count; compatible with sphere tracing for efficient ray marching [S1, unofficial][S2].
*   **Limitations:** Requires careful Lipschitz constant calculation to avoid ray marching artifacts; high computational cost for high-resolution or high-octave implementations [S2][S3, unofficial].
*   **When to use:** When organic, detailed surface relief is needed without the cost of high-poly geometry or explicit mesh generation [S1, unofficial].

## Background
The concept of hypertextures was proposed by Perlin [PH89] as a method for procedural texture synthesis evaluated within a volumetric region around an object [S1, unofficial]. It was inspired by local ray-casting methods for volume data but applied to surface relief [S1, unofficial]. The specific formulation for sphere tracing with distance bounds was detailed in subsequent work on implicit surfaces and noise deformation [S2].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
