# Locally Controlled Spot Noise
## Overview
Locally Controlled Spot Noise (LCSN) is a procedural noise method that generates structured patterns by convolving a non-uniform distribution of impulses with a kernel composed of multiple ellipsoidal Gaussian functions [S1, unofficial]. It extends standard Spot Noise by allowing spatial control over impulse density and kernel orientation/scale via underlying data fields, enabling the synthesis of fuzzy surface details [S1, unofficial]. Unlike standard Spot Noise which uses a uniform impulse distribution, LCSN rejects impulses based on a local density factor, making it suitable for authoring textures where feature density varies across the domain [S1, unofficial].

## How it works
**Inputs:**
- Evaluation point $\mathbf{p}$ in $N$-dimensional space [S1, unofficial].
- Local density field $d(\mathbf{p})$, a scalar field representing probability/density [S1, unofficial].
- Orientation field $R_s(\mathbf{p})$ and Scale field $S_s(\mathbf{p})$ related to underlying data [S1, unofficial].
- Kernel parameters: Gaussian magnitude $A$, and a set of matrices $\mathbf{V}_i$ defining ellipsoidal shapes [S1, unofficial].
- Random number generator for jittering and density testing [S1, unofficial].

**Outputs:**
- Noise value $n_s(\mathbf{p})$ [S1, unofficial].

**Data Structures:**
- Impulse $\mathbf{p}_j$: Position in texture space [S1, unofficial].
- Weight $w_j(\mathbf{p}_j)$: Associated with the impulse [S1, unofficial].
- Kernel $K_j(\mathbf{p})$: Defined as $k_s(\mathbf{p}-\mathbf{p}^*_j)R_s(\mathbf{p}^*_j)S_s(\mathbf{p}^*_j)$, where $\mathbf{p}^*_j$ is the impulse position [S1, unofficial].
- Matrix $\mathbf{V}$: $(N+1) \times (N+1)$ semipositive definite square matrix describing the ellipsoid iso-contour [S1, unofficial].
- Point $\mathbf{p}$: Extended to dimension $N+1$ with the last coordinate set to 1 for matrix multiplication [S1, unofficial].

**Formulas:**
The kernel function $k(\mathbf{p})$ is a sum of Gaussian functions:
$$ k(\mathbf{p}) = \sum_{V_i} g_{V_i} $$
where each Gaussian component is:
$$ g_V(\mathbf{p}) = A e^{-\frac{1}{2}\mathbf{p}^T \mathbf{V}^{-1} \mathbf{p}} $$
Here, $\mathbf{p}$ is the point in $N+1$ dimensions (last coord = 1), $\mathbf{V}$ is the shape matrix, and $A$ is the magnitude [S1, unofficial].

The noise value at $\mathbf{p}$ is:
$$ n_s(\mathbf{p}) = \sum_j \delta(\xi(\mathbf{p}_j) < d(\mathbf{p}_j)) |w_j(\mathbf{p}_j)| K_j(\mathbf{p}) $$
where:
- $\delta(\cdot)$ is the Kronecker delta (1 if condition true, 0 otherwise) [S1, unofficial].
- $\xi(\mathbf{p}_j)$ is a random variable selected independently of $w_j$ [S1, unofficial].
- $d(\mathbf{p}_j)$ is the local density factor at the impulse location [S1, unofficial].
- $||$ denotes absolute value [S1, unofficial].
- $K_j(\mathbf{p})$ incorporates orientation $R_s$ and scale $S_s$ transformations [S1, unofficial].

**Steps:**
1. **Impulse Generation:** Generate impulses $\mathbf{p}_j$ on the fly using jittering and pseudorandom number generation in texture space [S1, unofficial].
2. **Density Test:** For each impulse $\mathbf{p}_j$, generate a random density value $\xi(\mathbf{p}_j)$ [S1, unofficial]. Compare it against the local density factor $d(\mathbf{p}_j)$ from the scalar field [S1, unofficial].
3. **Rejection:** If $\xi(\mathbf{p}_j) \ge d(\mathbf{p}_j)$, reject the impulse (contribution is 0) [S1, unofficial]. If $\xi(\mathbf{p}_j) < d(\mathbf{p}_j)$, keep the impulse [S1, unofficial].
4. **Kernel Evaluation:** For kept impulses, compute the kernel $K_j(\mathbf{p})$ centered at $\mathbf{p}_j$, applying orientation $R_s(\mathbf{p}_j)$ and scale $S_s(\mathbf{p}_j)$ transformations [S1, unofficial].
5. **Summation:** Sum the weighted kernel contributions $|w_j(\mathbf{p}_j)| K_j(\mathbf{p})$ for all accepted impulses to produce $n_s(\mathbf{p})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def locally_controlled_spot_noise(p, density_field, orientation_field, scale_field, 
                                  kernel_matrices, gaussian_magnitude, random_gen):
    total_noise = 0.0

    # Generate impulses on the fly using jittering and PRNG [S1, unofficial]
    # The sources do not specify the exact grid or jittering parameters [G].
    for p_j, w_j in generate_jittered_impulses(random_gen):
        # Density test: generate random variable xi independently of w_j [S1, unofficial]
        xi_val = random_gen.uniform(0.0, 1.0)
        d_val = density_field(p_j)

        # Kronecker delta delta(xi < d)
        if xi_val < d_val:
            # Impulse accepted
            # Compute relative position for kernel: p - p*_j
            p_rel = [p[i] - p_j[i] for i in range(len(p))]
            p_rel_ext = p_rel + [1.0]

            # Apply orientation R_s and scale S_s transforms [S1, unofficial]
            R = orientation_field(p_j)
            S = scale_field(p_j)
            p_transformed = mat_vec_mul(S, mat_vec_mul(R, p_rel))
            p_transformed_ext = p_transformed + [1.0]

            # Evaluate kernel k(p_transformed) = sum of Gaussians [S1, unofficial]
            kernel_val = 0.0
            for V in kernel_matrices:
                inv_V = invert_matrix(V)
                prod = mat_vec_mul(inv_V, p_transformed_ext)
                exponent = -0.5 * dot_product(p_transformed_ext, prod)
                gaussian_val = gaussian_magnitude * math.exp(exponent)
                kernel_val += gaussian_val

            # Add weighted contribution: |w_j| * K_j(p)
            total_noise += abs(w_j) * kernel_val

    return total_noise

def generate_jittered_impulses(random_gen):
    # Standard jittered grid generation strategy [G]
    # The sources do not specify the exact grid or jittering parameters [G].
    # For valid Python execution, this yields a single dummy impulse.
    yield ([0.0, 0.0, 0.0], 1.0)

def mat_vec_mul(M, v):
    return [sum(M[i][j] * v[j] for j in range(len(v))) for i in range(len(M))]

def dot_product(u, v):
    return sum(u[i] * v[i] for i in range(len(u)))

def invert_matrix(M):
    # Standard matrix inversion algorithm (e.g., Gauss-Jordan or LU decomposition) [G]
    # For valid Python execution, returns identity for dummy input.
    return [[1.0 if i == j else 0.0 for j in range(len(M))] for i in range(len(M))]
```

**Edge Cases and Pitfalls:**
- **Parameterization Dependence:** LCSN is parameterization dependent because impulses are generated in texture space. Changes in texture mapping can alter the noise pattern [S1, unofficial].
- **Precision:** Matrix inversion $\mathbf{V}^{-1}$ must be handled carefully to ensure numerical stability, especially if $\mathbf{V}$ is near-singular [G].
- **Boundary Conditions:** The density field $d(\mathbf{p})$ must be defined at boundaries; otherwise, impulse rejection behavior is undefined [G].

## Operating characteristics
- **Time Complexity:** Depends on the number of impulses evaluated per evaluation point [S1, unofficial]. Since impulses are generated on the fly and filtered by density, the effective number of impulses contributing to the sum varies spatially [S1, unofficial]. Evaluating the kernel involves matrix operations for each impulse [S1, unofficial].
- **Memory Complexity:** Low, as impulses are generated on the fly and not stored in a large grid [S1, unofficial]. Only the fields $d, R_s, S_s$ and kernel parameters need to be stored [S1, unofficial].
- **Strengths:** Allows spatial control over noise density and structure [S1, unofficial]. Suitable for authoring fuzzy surface details [S1, unofficial].
- **Limitations:** Parameterization dependent [S1, unofficial]. Computationally demanding for direct volume rendering due to hundreds of evaluations per pixel along a ray [S1, unofficial].
- **When to Use:** When spatial control over noise density and orientation is required, and interactive visualization is not the primary constraint or if splatting with analytic filtering is used [S1, unofficial].
- **When Not to Use:** When parameterization independence is critical or when real-time volumetric rendering performance is paramount without optimization [S1, unofficial].

## Background
Locally Controlled Spot Noise was introduced by Nicolas Pavie, Guillaume Gilet, Jean-Michel Dischler, Eric Galin, and Djamchid Ghazanfarpour in 2016 in the paper "Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis" [S1, unofficial]. It extends the original Spot Noise concept [vW91, dLvL97] to provide better spatial control for procedural texture synthesis, particularly for shell textures [S1, unofficial]. The method addresses limitations of previous procedural noise methods like Gabornoise, which could only produce Gaussian patterns [S1, unofficial].

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
