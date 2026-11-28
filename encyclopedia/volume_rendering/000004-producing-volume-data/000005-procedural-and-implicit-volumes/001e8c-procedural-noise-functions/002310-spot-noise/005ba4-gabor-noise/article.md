# Gabor Noise

Also called: GPU Gabor Noise (Procedural), Sparse convolution noise using Gabor kernel

## Overview
Gabor noise is a procedural noise function that generates band-limited, anisotropic or isotropic patterns by convolving a sparse set of random impulses with a Gabor kernel [S1][S2, unofficial][S3]. It solves the problem of visible kernel structures common in general sparse convolution noise at low densities, while providing explicit spectral control over orientation, frequency, and bandwidth [S1][S2, unofficial]. Unlike its sibling [Locally Controlled Spot Noise](4599-locally-controlled-spot-noise.md), which allows spatial variation of kernel parameters, standard Gabor noise typically uses fixed or globally varied parameters to define a specific power spectrum [S1, S3].

## How it works
Gabor noise builds on the sparse convolution framework defined in [Spot Noise](2959-spot-noise.md) [S3]. It replaces the generic kernel with a Gabor kernel, which is the product of a circular Gaussian and a 2D cosine wave [S1, S3].

### Inputs and Outputs
*   **Inputs:**
    *   Query point $(x, y)$ in 2D space [S1, S3].
    *   Random point positions $\{(x_i, y_i)\}$ distributed according to a Poisson impulse process [S2, unofficial].
    *   Random weights $\{w_i\}$ [S1, S3].
    *   Kernel parameters for each point: magnitude $K_i$, inverse width $a_i$, frequency $F_{0,i}$, and orientation $\omega_{0,i}$ [S1, S3].
*   **Output:**
    *   Noise value $N(x, y)$ [S1, S3].

### Data Structures
The sources describe a procedural implementation where properties are generated on the fly using a pseudo-random number generator to ensure random accessibility [S2, unofficial]. A virtual grid is used where the cell size equals the Gabor kernel radius, requiring only the containing cell and direct neighboring cells for evaluation [S2, unofficial].

### Formulas
The Gabor kernel in the spatial domain $g(x, y)$ is defined as [S1, S3]:

$$
g(x, y) = K e^{-\pi a^2 (x^2 + y^2)} \cos [2\pi F_0 (x \cos \omega_0 + y \sin \omega_0)]
$$

Where:
*   $K$ is the magnitude [S1, S3].
*   $a$ is the inverse width of the Gaussian [S1, S3].
*   $F_0$ is the frequency of the cosine [S1, S3].
*   $\omega_0$ is the orientation of the cosine [S1, S3].

The noise value $N(x, y)$ is the sum of weighted kernels centered at random positions [S1, S3]:

$$
N(x, y) = \sum_i w_i g(K_i, a_i, F_{0,i}, \omega_{0,i}; x - x_i, y - y_i)
$$

Substituting the kernel definition:

$$
N(x, y) = \sum_i w_i K_i e^{-\pi a_i^2 ((x - x_i)^2 + (y - y_i)^2)} \cos [2\pi F_{0,i} ((x - x_i) \cos \omega_{0,i} + (y - y_i) \sin \omega_{0,i})]
$$

The power spectrum of the Gabor kernel in the frequency domain $G(f_x, f_y)$ is a pair of circular Gaussians [S1, S3]:

$$
G(f_x, f_y) = \frac{K}{2a^2} \exp \left[ -\frac{\pi}{2a^2} ((f_x \pm F_0 \cos \omega_0)^2 + (f_y \pm F_0 \sin \omega_0)^2) \right]
$$

This indicates that the energy is concentrated around polar coordinates $(F_0, \omega_0)$ in the frequency domain, with a width determined by $a$ [S1, S3].

### Parameters and Typical Values
*   **Fixed Parameters:** When $K_i, a_i, F_{0,i}, \omega_{0,i}$ are constant for all $i$, the noise is anisotropic and band-limited [S1, S3].
    *   $\omega_0$ controls orientation.
    *   $F_0$ controls frequency.
    *   $a$ controls bandwidth [S1, S3].
*   **Varying Parameters:** When parameters vary, the power spectrum is the Gabor kernel integrated over the parameter distribution [S1, S3].
    *   If $\omega_{0,i}$ is uniformly distributed over $[0, 2\pi)$, the noise becomes isotropic band-limited noise [S1, S3].
*   **GUI Specification:** The original implementation uses graphical user interface widgets to specify how parameters vary to define the power spectrum [S1, S3].

### Pseudocode
The following Python-like pseudocode implements the evaluation of Gabor noise for a single query point [S2, unofficial]. In practice, points are generated procedurally on the fly using a PRNG to ensure random accessibility [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gabor_noise(x, y, points):
    """
    Evaluate Gabor noise at (x, y).
    
    Args:
        x (float): X coordinate of the query point.
        y (float): Y coordinate of the query point.
        points (list of dict): List of random points. Each dict contains:
            - 'x': x_i (float)
            - 'y': y_i (float)
            - 'w': w_i (float)
            - 'K': K_i (float)
            - 'a': a_i (float)
            - 'F0': F_{0,i} (float)
            - 'omega0': omega_{0,i} (float)
            
    Returns:
        float: The noise value N(x, y).
    """
    result = 0.0
    
    for p in points:
        # Relative coordinates
        dx = x - p['x']
        dy = y - p['y']
        
        # Gaussian envelope: exp(-pi * a^2 * (dx^2 + dy^2))
        gaussian_arg = -math.pi * (p['a'] ** 2) * (dx**2 + dy**2)
        gaussian_val = math.exp(gaussian_arg)
        
        # Cosine component: cos(2 * pi * F0 * (dx * cos(omega0) + dy * sin(omega0)))
        cos_omega = math.cos(p['omega0'])
        sin_omega = math.sin(p['omega0'])
        cos_arg = 2 * math.pi * p['F0'] * (dx * cos_omega + dy * sin_omega)
        cos_val = math.cos(cos_arg)
        
        # Kernel value
        kernel_val = p['K'] * gaussian_val * cos_val
        
        # Weighted sum
        result += p['w'] * kernel_val
        
    return result
```

### Edge Cases and Pitfalls
*   **Boundary Conditions:** As a sparse convolution noise, it does not have inherent periodicity unless the point distribution and parameters are made periodic [S3].

## Subtopics

### NPR Gabor Noise
A variant where noise parameters are defined in 2D screen space while point distribution is in 3D object space, used for coherent stylization in non-photorealistic rendering [S3].

### Setup-Free Surface Noise
Gabor noise is used to generate noise directly on parametric surfaces without requiring a 3D volume or texture mapping setup [S1, S3, S4].

### Analytic Anisotropic Filtering
The method supports analytic anisotropic filtering of the noise [S1, S3, S4].

## Operating characteristics
*   **Time Complexity:** Approximately $O(N)$ per query, where $N$ is the number of random points within the evaluation radius, with about nine contributing kernels needed per point in 2D [S2, unofficial].
*   **Memory Complexity:** $O(1)$ storage requirements relative to the resolution, as it is a procedural function relying on a fixed number of random points [S4].
*   **Performance:** Speed is generally slower than Perlin noise due to the higher computational cost of the kernel evaluation and the need to iterate over random points [S4].
*   **Strengths:**
    *   Provides explicit spectral control (orientation, frequency, bandwidth) [S1, S3].
    *   Avoids visible kernel structures at low densities compared to general sparse convolution [S1][S2, unofficial].
    *   Supports setup-free surface noise and analytic filtering [S1, S3, S4].
    *   Has a Gaussian amplitude distribution [S4].
*   **Limitations:**
    *   Requires more computation per point than lattice noises [S2, unofficial].
*   **When to use:** When spectral control is required, or when generating noise on surfaces without setup overhead [S1, S3, S4].
*   **When not to use:** When performance is critical and precise spectral control is not required, simpler noise functions are preferred [S2, unofficial][S4].

## Background
Gabor noise was introduced by Lagae et al. in 2009 [S1, S3, S4]. It was presented in the paper "A Survey of Procedural Noise Functions" and related works [LLDD09a, LLDD09b, LLD09] [S1, S3, S4]. The method extends the framework of sparse convolution noise by introducing the Gabor kernel, which combines a Gaussian envelope with a cosine wave [S1, S3]. It is used for setup-free surface noise and analytic anisotropic filtering of noise [S1, S3, S4].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S3] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S4] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
