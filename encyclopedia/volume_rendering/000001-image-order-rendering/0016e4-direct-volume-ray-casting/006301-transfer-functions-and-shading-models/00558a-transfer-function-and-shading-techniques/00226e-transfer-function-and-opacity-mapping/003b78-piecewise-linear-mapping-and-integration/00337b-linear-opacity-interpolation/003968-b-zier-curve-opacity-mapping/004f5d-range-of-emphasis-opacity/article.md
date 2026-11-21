# Range of Emphasis Opacity

Also called: Density Range Emphasizing, Opacity Scaling with Range of Emphasis, Simplified Transfer Function

## Overview
Range of Emphasis Opacity is a transfer function technique that emphasizes specific density ranges by scaling opacity differently inside and outside a user-defined interval $[\rho_{min}, \rho_{max}]$ [S1, unofficial]. It is distinct from its sibling [Bézier Curve Opacity Mapping](8485-b-zier-curve-opacity-mapping.md) in that it uses a piecewise linear function with explicit multipliers rather than a continuous Bézier curve, offering a simpler, more direct control over the visibility of target densities [S1, unofficial]. A variant of this concept, used in encrypted volume rendering, encodes density values as vectors to compute similarity via dot products, avoiding the storage overhead of full oblivious lookup tables [S2].

## How it works
This method operates in two distinct contexts: standard interactive volume rendering [S1, unofficial] and homomorphically encrypted volume rendering [S2]. Both share the goal of emphasizing a specific density range, but differ in implementation.

### Standard Interactive Rendering
In standard rendering, the method calculates opacity $\alpha$ for each sample along a ray based on the scalar density value $\rho$ and user-defined parameters.

**Inputs:**
- $\rho$: Normalized density value of the current sample ($0 \le \rho \le 1$).
- $\rho_{min}$: Lower bound of the emphasis range.
- $\rho_{max}$: Upper bound of the emphasis range.
- $\alpha_1$: Opacity multiplier for densities inside the range of emphasis.
- $\alpha_2$: Opacity multiplier for densities outside the range of emphasis.

**Outputs:**
- $\alpha$: Calculated opacity for the sample.

**Data Structures:**
- No complex data structures are required beyond the scalar parameters and the volume texture.

**Steps:**
1. For each sample along the ray, retrieve the density value $\rho$.
2. Calculate the opacity $\alpha$ using the piecewise function defined below.
3. Perform front-to-back alpha blending using the calculated $\alpha$.
4. Terminate the ray when the accumulated alpha value reaches 0.96 or when the ray exits the volume [S1, unofficial].

**Formula:**
The opacity $\alpha$ is calculated as [S1, unofficial]:

$$
\alpha = \begin{cases} 
\rho \alpha_1 & \rho_{min} < \rho < \rho_{max} \\
\rho \left( (1 - (\rho/\rho_{min})) \alpha_2 + (\rho/\rho_{min}) \alpha_1 \right) & \rho \le \rho_{min} \\
\rho \left( (1 - (\frac{\rho - \rho_{max}}{1 - \rho_{max}})) \alpha_1 + (\frac{\rho - \rho_{max}}{1 - \rho_{max}}) \alpha_2 \right) & \rho \ge \rho_{max}
\end{cases}
$$

Where:
- $\rho$ is the density value.
- $\alpha_1$ is the opacity multiplier for the range of emphasis.
- $\alpha_2$ is the out-of-range opacity multiplier.
- $\rho_{min}$ is the lower bound of the emphasis range.
- $\rho_{max}$ is the upper bound of the emphasis range.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_opacity(density, rho_min, rho_max, alpha1, alpha2):
    """
    Calculate opacity for a given density using Range of Emphasis.
    
    Args:
        density (float): Normalized density value [0.0, 1.0].
        rho_min (float): Lower bound of emphasis range.
        rho_max (float): Upper bound of emphasis range.
        alpha1 (float): Opacity multiplier inside range.
        alpha2 (float): Opacity multiplier outside range.
        
    Returns:
        float: Calculated opacity alpha.
    """
    if rho_min < density < rho_max:
        return density * alpha1
    elif density <= rho_min:
        # Linear interpolation between alpha2 (at 0) and alpha1 (at rho_min)
        # Formula derived from: rho * ((1 - (rho/rho_min)) * alpha2 + (rho/rho_min) * alpha1)
        term1 = (1.0 - (density / rho_min)) * alpha2
        term2 = (density / rho_min) * alpha1
        return density * (term1 + term2)
    else: # density >= rho_max
        # Linear interpolation between alpha1 (at rho_max) and alpha2 (at 1.0)
        # Formula derived from: rho * ((1 - ((rho - rho_max)/(1 - rho_max))) * alpha1 + ((rho - rho_max)/(1 - rho_max)) * alpha2)
        ratio = (density - rho_max) / (1.0 - rho_max)
        term1 = (1.0 - ratio) * alpha1
        term2 = ratio * alpha2
        return density * (term1 + term2)
```

**Parameters and Typical Values:**
- $\alpha_1$: Typical values are 0.16 (Karman Vortex Street), 0.1 (Jet Stream), 0.1 (Dam Break) [S1, unofficial].
- $\alpha_2$: Typical values are 0.01 (Karman Vortex Street, Jet Stream), 0.03 (Dam Break) [S1, unofficial].
- $\rho_{min}$: Typical values are 0.31 (Karman Vortex Street), 0.55 (Jet Stream), 0.24 (Dam Break) [S1, unofficial].
- $\rho_{max}$: Typical values are 0.37 (Karman Vortex Street), 0.59 (Jet Stream), 0.27 (Dam Break) [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $\rho_{min} = 0$, the division by $\rho_{min}$ in the $\rho \le \rho_{min}$ case causes a singularity. The sources do not specify a guard for this, but standard practice would require $\rho_{min} > 0$ or a special case for $\rho=0$ [S1, unofficial].
- The formula assumes normalized density values in $[0, 1]$. If densities are not normalized, the terms $(\rho/\rho_{min})$ and $(\frac{\rho - \rho_{max}}{1 - \rho_{max}})$ may exceed valid ranges, leading to incorrect opacity scaling [S1, unofficial].

### Encrypted Rendering Variant
In homomorphically encrypted volume rendering, the method emphasizes density ranges by encoding density values as vectors and computing dot products with a plaintext emphasizing vector [S2].

**Inputs:**
- `density`: Normalized density value of the voxel.
- `dim`: Dimensionality of the encoding vector.

**Outputs:**
- `v`: A normalized vector of length `dim` representing the density.

**Data Structures:**
- `v`: A vector of length `dim`, initialized to zeros.

**Steps:**
1. Initialize a vector `v` of length `dim` with all indices set to 0.
2. Calculate `s = density * 2 * (dim - 1)`.
3. Calculate `f = (floor(s) + 1) / 2`.
4. Calculate `d = floor(f)`.
5. Set `v[d] = 1`.
6. If `d > 0` and `d != f`, set `v[d-1] = 1 - (s - floor(s))`.
7. Else if `d + 1 < dim` and `d < f`, set `v[d+1] = s - floor(s)`.
8. Return the normalized vector `v`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def encode_density(density, dim):
    """
    Encode a normalized density value into a vector for encrypted rendering.
    
    Args:
        density (float): Normalized density value [0.0, 1.0].
        dim (int): Dimensionality of the output vector.
        
    Returns:
        list: Normalized vector of length dim.
    """
    # Initialize vector with zeros
    v = [0.0] * dim
    
    # Calculate intermediate values
    s = density * 2 * (dim - 1)
    f = (math.floor(s) + 1) / 2
    d = math.floor(f)
    
    # Set the primary component
    # Note: The source algorithm sets v[d] = 1. 
    # We must ensure d is within bounds [0, dim-1].
    # Given density in [0,1] and dim >= 1, d should be valid.
    if 0 <= d < dim:
        v[d] = 1.0
    
    # Set adjacent components based on conditions
    if d > 0 and d != f:
        # d-1 is guaranteed >= 0 because d > 0
        v[d - 1] = 1.0 - (s - math.floor(s))
    elif d + 1 < dim and d < f:
        # d+1 is guaranteed < dim because d+1 < dim
        v[d + 1] = s - math.floor(s)
        
    # Normalize the vector
    magnitude = math.sqrt(sum(x*x for x in v))
    if magnitude > 0:
        v = [x / magnitude for x in v]
        
    return v
```

**Parameters:**
- `dim`: The number of dimensions of the encoded vector. Must be the same during encryption and for the emphasizing density encoding. Typical values mentioned are 4 or 6 [S2].

**Pitfalls:**
- The vector components must be normalized so that the dot product result is always between 0 and 1, ensuring the highest result (1) occurs at the user-defined emphasizing density [S2].
- The encoding curve should be steep and narrow to clearly distinguish the selected density [S2].
- Increasing `dim` makes the dot product response curve steeper but increases storage and computation [S2].

## Operating characteristics
- **Time Complexity:** Standard rendering is $O(1)$ per sample for opacity calculation. Encrypted rendering adds the cost of vector encoding and dot product computation, which is $O(dim)$ per sample, where `dim` is the vector dimensionality [S2].
- **Memory Complexity:** Standard rendering requires no additional memory beyond the volume data. Encrypted rendering requires storing vectors of length `dim` for each voxel, increasing storage by a factor of `dim` compared to scalar storage [S2].
- **Measured Performance:** In encrypted rendering, using a 2048-bit key with 256-dimensional vectors requires 16 TB for a volume dataset, which is considered too large for practical use, motivating the use of lower-dimensional vectors (e.g., 4D or 6D) [S2].
- **Strengths:** Simple interactive control over density ranges [S1, unofficial]. In encrypted contexts, it avoids the storage overhead of full oblivious lookup tables by using lower-dimensional vectors [S2].
- **Limitations:** The piecewise linear function may produce less smooth transitions than Bézier curves [S1, unofficial]. In encrypted contexts, higher dimensions improve selectivity but drastically increase storage and computation [S2].

## Background
The Range of Emphasis Opacity technique was introduced in the context of illustrative rendering of particle systems, where it was used to highlight specific density structures in datasets like the von Karman vortex street and jet data [S1, unofficial]. A simplified variant of this approach, adapted for homomorphically encrypted volume rendering, was developed to allow users to emphasize selectable density ranges in encrypted data without the prohibitive storage costs of full oblivious lookup tables [S2].

## Sources
[S1] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
[S2] Homomorphic-Encrypted Volume Rendering, https://arxiv.org/abs/2009.02122v2, fetched 2026-10-26
