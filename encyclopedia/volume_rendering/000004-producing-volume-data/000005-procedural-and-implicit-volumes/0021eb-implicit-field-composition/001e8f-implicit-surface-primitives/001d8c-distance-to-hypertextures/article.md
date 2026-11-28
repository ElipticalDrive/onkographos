# Distance to Hypertextures

Also called: Fractal noise deformation, Hypertexture, Hypertexture noise

## Overview
Distance to Hypertextures is a technique for adding relief or details to surfaces without increasing geometry cost, using procedural texture synthesis evaluated within a volumetric region around the object rather than only on the surface [S1, unofficial]. It deforms implicit surfaces by adding procedural noise to the function's range instead of the domain, enabling sphere tracing with distance bounds [S2]. Unlike standard hypertexturing which may deform the domain, this variant affects the range, allowing the use of signed distance functions (SDFs) for efficient rendering [S2].

## How it works
**Inputs and Outputs:**
- Input: A base signed distance function $f(\vec{x})$, a noise function $noise(\vec{x})$, and a spectral exponent $\alpha$ [S2].
- Output: A deformed implicit surface defined by $f(\vec{x}) + noise(\vec{x}) = 0$ [S2].

**Data Structures:**
- $\Gamma$: An array of random unit vectors used for solid noise interpolation [S2].
- Octaves: A finite integer $n$ controlling the number of noise layers [S2].

**Steps:**
1. **Define Base Noise:** Construct band-limited solid noise using a piecewise cubic Gaussian approximation $\phi$ and random vectors $\Gamma$ [S2]:
   $$ noise(\mathbf{x}) = \sum_{i=-1}^{1} \sum_{j=-1}^{1} \sum_{k=-1}^{1} \Gamma(i,j,k) \cdot \phi(\mathbf{x} - (i,j,k)) $$
   where $\phi$ is a scalar piecewise cubic approximation of a Gaussian distribution [S2].
2. **Compute Lipschitz Constant for Base Noise:** The Lipschitz constant for the base noise is derived from the composition of $\phi$ and $\Gamma$. Given $Lip_{\phi} = 3/2$ and $Lip_{\Gamma} = 2$, the composition results in $Lip_{noise} = 3$ [S2].
3. **Generate Fractal Noise:** Sum scaled versions of the base noise over $n$ octaves to create fractal noise with spectral exponent $\alpha$ [S2]:
   $$ noise_\alpha(\mathbf{x}) = \sum_{i=0}^{n} \frac{noise(2^i \mathbf{x})}{2^{i\alpha}} $$
4. **Deform Surface:** Evaluate the deformed implicit field $F(\vec{x}) = f(\vec{x}) + noise_\alpha(\vec{x})$ [S2].
5. **Sphere Tracing:** Use the Lipschitz constant of the total function to determine safe step sizes for sphere tracing [S2].

**Parameters:**
- $\alpha$: Spectral exponent. $\alpha=0$ is white noise, $\alpha=1$ is $1/f$ noise, $\alpha=2$ is Brownian motion [S2].
- $n$: Number of octaves. Must be finite to ensure the function remains Lipschitz [S2].
- $\Gamma$: Array of random unit vectors [S2].

**Edge Cases and Pitfalls:**
- Unbounded $1/f$ noise is not Lipschitz; it must be band-limited to a finite number of octaves $n$ to remain Lipschitz for sphere tracing [S2].
- For $1/f$ noise, the Lipschitz constant equals the sum of individual noise functions, $Lip_{1/f \ noise} = 3$ [S2].
- For Brownian motion ($\alpha=2$), the Lipschitz constant is $Lip_{B} = 3(2^{1-n} - 1) \approx 6$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hypertexture_deformation(x, f, alpha, n, Gamma):
    """
    Computes the deformed implicit surface value using Distance to Hypertextures.
    
    Args:
        x: 3D point (list of 3 floats)
        f: Base signed distance function (callable)
        alpha: Spectral exponent (float)
        n: Number of octaves (int)
        Gamma: 3x3x3 array of random unit vectors (list of lists of lists of lists)
        
    Returns:
        Deformed field value (list of 3 floats)
    """
    noise_value = [0.0, 0.0, 0.0]
    for i in range(n + 1):
        scaled_x = [x[j] * (2 ** i) for j in range(3)]
        current_noise = [0.0, 0.0, 0.0]
        for di in range(-1, 2):
            for dj in range(-1, 2):
                for dk in range(-1, 2):
                    vec = Gamma[di + 1][dj + 1][dk + 1]
                    offset = [scaled_x[0] - di, scaled_x[1] - dj, scaled_x[2] - dk]
                    phi_val = cubic_gaussian_approx(offset)
                    current_noise = [current_noise[j] + vec[j] * phi_val for j in range(3)]
        noise_value = [noise_value[j] + current_noise[j] / (2 ** (i * alpha)) for j in range(3)]
    return [f(x[j]) + noise_value[j] for j in range(3)]

def cubic_gaussian_approx(p):
    """
    Piecewise cubic approximation of Gaussian distribution.
    Based on Wyvill et al. [S2]
    """
    r2 = sum(v * v for v in p)
    threshold = 1.5
    if r2 < threshold:
        term = threshold - r2
        return (2.0 / 3.0) * (term ** 3)
    else:
        return 0.0
```

## Subtopics

### On-the-Fly Distance Field & Hypertexture Lookup
A method where a distance field is derived from the mesh on the fly on the GPU, and surface vicinity is used for hypertexture lookups [S3, unofficial]. It uses 20 octaves of Perlin noise for shading and complex materials [S3, unofficial]. Due to complex GPU computations, frame rate is around 20 FPS for a $1024^3$ volume [S3, unofficial].

## Operating characteristics
- **Time Complexity:** High computational cost due to multiple noise evaluations and octaves. Frame rates can be low (e.g., 20 FPS for $1024^3$ volume) [S3, unofficial].
- **Memory Complexity:** Depends on the number of octaves and the size of the Gamma array. In streaming implementations, memory usage can be optimized using node pools and bricks [S3, unofficial].
- **Strengths:** Adds high-frequency detail without increasing geometric complexity; enables sphere tracing for efficient rendering [S1, unofficial][S2].
- **Limitations:** Unbounded $1/f$ noise is not Lipschitz and must be band-limited to a finite number of octaves to remain Lipschitz for stable sphere tracing [S2].
- **When to use:** When high-quality procedural detail is needed on implicit surfaces and performance allows for complex GPU computations [S3, unofficial].

## Background
Proposed by Perlin [PH89], hypertextures were inspired by the idea of evaluating procedural texture synthesis within a volumetric region around an object rather than only on the surface [S1, unofficial]. This allows for the modeling of phenomena intermediate between surface and texture [S1, unofficial]. The specific variant affecting the range to enable sphere tracing was detailed in subsequent work [S2].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
