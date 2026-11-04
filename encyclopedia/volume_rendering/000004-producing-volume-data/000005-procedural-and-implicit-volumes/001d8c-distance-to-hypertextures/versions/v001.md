# Distance to Hypertextures

Also called: Fractal noise deformation, Hypertexture, Hypertexture noise

## Overview
Distance to Hypertextures is a technique for adding relief or details to surfaces without increasing geometry cost, using procedural texture synthesis evaluated within a volumetric region around the object rather than only on the surface [S1, unofficial]. It deforms implicit surfaces by adding procedural noise to the function's range instead of the domain, enabling sphere tracing with distance bounds [S2]. This method allows for the modeling of phenomena intermediate between surface and texture [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base signed distance function (SDF) $f(\mathbf{x})$, a spectral exponent $\alpha$, and the number of octaves $n$.
*   **Output:** A deformed implicit surface defined by $f(\mathbf{x}) + \text{noise}(\mathbf{x}) = 0$, and a Lipschitz constant $L$ for safe sphere tracing.

**Data Structures:**
*   **Noise Lattice:** An array of random unit vectors $\Gamma(i,j,k)$ for $i,j,k \in \{-1, 0, 1\}$.
*   **Base Function:** A cubic Gaussian approximation $\phi(r)$ used for soft objects, defined as:
    $$ \phi(r) = \begin{cases} \frac{2}{3} - r^2 + \frac{1}{2}r^3 & \text{if } 0 \le r \le 1 \\ \frac{1}{6}(2-r)^3 & \text{if } 1 < r \le 2 \\ 0 & \text{otherwise} \end{cases} $$
    (Note: The source text fragment for Eq. 36 is garbled, but standard Wyvill cubic Gaussian is implied by the context of "piecewise cubic in distance" and the coefficients $\frac{2}{3}$ and $\frac{1}{6}$).

**Steps:**
1.  **Define Base Noise:** Compute the band-limited solid noise at position $\mathbf{x}$ using the lattice of random unit vectors $\Gamma$ and the cubic Gaussian $\phi$:
    $$ \text{noise}(\mathbf{x}) = \sum_{i=-1}^{1} \sum_{j=-1}^{1} \sum_{k=-1}^{1} \Gamma(i,j,k) \cdot \phi(\|\mathbf{x} - (i,j,k)\|) $$
    [S2]
2.  **Compute Lipschitz Constant for Base Noise:**
    *   The Lipschitz constant of the cubic Gaussian $\phi$ is $Lip_\phi = \frac{3}{2}$ [S2].
    *   The Lipschitz constant of the vector field $\Gamma$ is $Lip_\Gamma = 2$ (due to opposing vectors being neighbors) [S2].
    *   The composition results in $Lip_{\text{noise}} = Lip_\phi \cdot Lip_\Gamma = \frac{3}{2} \cdot 2 = 3$ [S2].
3.  **Construct Fractal Noise:** Sum scaled versions of the base noise over $n+1$ octaves:
    $$ \text{noise}_\alpha(\mathbf{x}) = \sum_{i=0}^{n} \frac{\text{noise}(2^i \mathbf{x})}{2^{i\alpha}} $$
    [S2]
4.  **Determine Global Lipschitz Constant:**
    *   If $\alpha = 1$ ($1/f$ noise), the noise is not Lipschitz if unbounded. It must be band-limited to a finite number of octaves $n$ to remain Lipschitz for sphere tracing [S2].
    *   For band-limited $1/f$ noise, the Lipschitz constant is the sum of individual functions: $Lip_{1/f} = 3$ [S2].
    *   For Brownian motion ($\alpha = 2$), the Lipschitz constant is:
        $$ Lip_{B} = 3(2^{1-n} - 1) \approx 6 \quad (\text{for small } n) $$
        [S2]
    *   The total Lipschitz constant for the deformed surface $f(\mathbf{x}) + \text{noise}_\alpha(\mathbf{x})$ is $Lip_f + Lip_{\text{noise}_\alpha}$.
5.  **Sphere Tracing:** Use the computed Lipschitz constant to determine the maximum step size for sphere tracing the implicit surface $f(\mathbf{x}) + \text{noise}_\alpha(\mathbf{x}) = 0$.

**Parameters:**
*   $\alpha$: Spectral exponent. $\alpha=0$ is white noise, $\alpha=1$ is $1/f$ noise, $\alpha=2$ is Brownian motion [S2].
*   $n$: Number of octaves. Must be finite for $1/f$ noise to be Lipschitz [S2].
*   Typical values: $n$ varies by application; [S3, unofficial] uses 20 octaves for high-quality shading and 5 octaves for cloud simulation.

**Edge Cases and Pitfalls:**
*   Unbounded $1/f$ noise is not Lipschitz and cannot be used with sphere tracing unless band-limited [S2].
*   Precision issues may arise with high octave counts due to the summation of small amplitudes [S3, unofficial].

## Subtopics

### On-the-Fly Distance Field & Hypertexture Lookup
A variant where a distance field is derived from the mesh on the fly on the GPU, and surface vicinity is used for hypertexture lookups [S3, unofficial]. It uses 20 octaves of Perlin noise for shading and complex materials [S3, unofficial]. Due to complex GPU computations, the frame rate is relatively low (around 20 FPS for a $1024^3$ volume) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** High. The evaluation of fractal noise with many octaves is computationally expensive. [S3, unofficial] reports ~20 FPS for a $1024^3$ volume.
*   **Memory Complexity:** Depends on the implementation. [S3, unofficial] uses a node pool (e.g., 4 MB for $64^3$ entries) and a brick pool (e.g., 430 MB) to store indices and data.
*   **Strengths:** Adds high-frequency detail without increasing geometric complexity; compatible with sphere tracing if Lipschitz bounds are respected.
*   **Limitations:** Computationally intensive; requires careful handling of Lipschitz constants for stability.

## Background
Proposed by Perlin [PH89], hypertextures were inspired by the idea of evaluating procedural texture synthesis within a volumetric region around an object rather than solely on the surface [S1, unofficial]. This allows for the modeling of phenomena intermediate between surface and texture [S1, unofficial]. The method was later adapted for sphere tracing by Wyvill et al. and others, who established the Lipschitz bounds necessary for efficient rendering [S2].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
