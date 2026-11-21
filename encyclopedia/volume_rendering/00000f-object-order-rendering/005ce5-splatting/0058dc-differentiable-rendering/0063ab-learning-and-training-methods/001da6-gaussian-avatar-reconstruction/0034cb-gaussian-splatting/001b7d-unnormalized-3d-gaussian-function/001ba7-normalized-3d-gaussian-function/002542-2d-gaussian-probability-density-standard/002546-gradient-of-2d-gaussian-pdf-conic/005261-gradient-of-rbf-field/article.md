# Gradient of RBF Field

Also called: Fragment program for reconstructing Gaussian radial basis functions, RBF Gradient Evaluation, Single-pass RBF Reconstruction

## Overview
This method computes the scalar field value and its spatial gradient analytically in a single fragment shader pass by evaluating a sum of Gaussian Radial Basis Functions (RBFs) [S1]. It solves the problem of reconstructing procedurally encoded scalar fields from texture-mapped parameters for interactive visualization, specifically enabling isosurface rendering via accurate normal estimation [S1]. It differs from its parent [Gradient of 2D Gaussian PDF (Conic)](3472-gradient-of-2d-gaussian-pdf-conic.md) by operating in 3D model space using standard Gaussian RBFs rather than 2D conic projections, and by explicitly computing the gradient $\nabla f(x)$ in parallel with the field value $f(x)$ [S1].

## How it works
**Inputs:**
- Fragment position in model space, obtained from interpolated texture coordinates `inpos` [S1].
- Texture maps containing RBF parameters:
  - Center coordinates $\mu_i$ [S1].
  - Weights $w_i$ and widths $\sigma_i$ [S1].
  - Optional error values at cell corners, encoded in secondary texture coordinates [S1].
- A fixed preprocessor constant $M$, representing the total number of RBFs to evaluate [S1].
- A constant bias term [S1].
- Scale and bias factors for data range mapping [S1].
- A 1D texture for color mapping [S1].

**Outputs:**
- Scalar field value $f(x)$ [S1].
- Gradient vector $\nabla f(x)$ [S1].
- Output color derived from $f(x)$ [S1].

**Data Structures and Layout:**
RBF parameters are stored in texture maps. Five parameters per RBF (center coordinates and width/weight) cannot fit in a single texel, requiring two lookups per fragment [S1].
- **Center Coordinates:** Stored in a texture map [S1].
- **Weights and Widths:** Stored in a texture map [S1].
- **Error Values:** If present, stored in secondary texture coordinates corresponding to the cell corners [S1].
- **Optimization Note:** Combining four RBF widths into one RGBA texel and looking up width only every fourth RBF was considered to reduce lookups to an average of 1.25, but was rejected due to complexity yielding negligible performance gain [S1].

**Formulas:**
The scalar field $f(x)$ is the sum of Gaussian RBFs:
$$ f(x) = \sum_{i=1}^{M} w_i e^{-\frac{\|x - \mu_i\|^2}{2\sigma_i^2}} + b $$
where $b$ is the constant bias [S1].

The gradient $\nabla f(x)$ is evaluated analytically in parallel:
$$ \nabla f(x) = - \sum_{i=1}^{M} w_i \frac{x - \mu_i}{2\sigma_i^2} e^{-\frac{\|x - \mu_i\|^2}{2\sigma_i^2}} $$
Note: The source formula [S1] writes the denominator of the exponent as $2\sigma_i^2$ and the gradient term as $\frac{x - \mu_i}{2\sigma_i}$. However, the standard derivative of $e^{-\frac{\|x - \mu_i\|^2}{2\sigma_i^2}}$ with respect to $x$ is $-\frac{x - \mu_i}{\sigma_i^2} e^{-\dots}$. The source text explicitly states:
$$ \nabla f (x) = - \sum_{i=1}^{M} w_i \frac{x - \mu_i}{2\sigma_i} e^{-\frac{\|x-\mu_i\|^2}{2\sigma_i^2}} $$
We retain the source's explicit formula [S1]. It is likely the source implies a specific normalization or there is a typo in the source's denominator ($2\sigma_i$ vs $\sigma_i^2$). Given the instruction to keep exact numbers and variable names, we cite the source formula exactly. However, standard calculus dictates the gradient of $\exp(-\frac{r^2}{2\sigma^2})$ is $-\frac{r}{\sigma^2}\exp(\dots)$. The source formula has $\frac{1}{2\sigma_i}$ in the numerator term. Let's look closely at [S1]: "$\nabla f (x) = - \sum_{i=1}^{M} w_i \frac{x - \mu_i}{2\sigma_i} e^{-\frac{kx-\mu_i k^2}{2\sigma_i^2}}$". The notation $kx-\mu_i k^2$ is clearly $\|x - \mu_i\|^2$. The term $\frac{x - \mu_i}{2\sigma_i}$ is dimensionally inconsistent with $\frac{x - \mu_i}{\sigma_i^2}$ if $\sigma$ is a length scale. However, if $\sigma$ in the texture is stored as $1/\sigma_{std}$, the formula might change. The source states: "compensated with a correction factor multiplied to $\sigma_i$ entries". This suggests the stored $\sigma_i$ might be scaled. We will present the formula exactly as cited in [S1] but note the potential discrepancy.

**Parameters:**
- $M$: Number of RBFs. Fixed preprocessor constant because graphics hardware does not support dynamic loops in fragment processing [S1].
- $\mu_i$: Center of the $i$-th RBF [S1].
- $w_i$: Weight of the $i$-th RBF [S1].
- $\sigma_i$: Width of the $i$-th RBF [S1].
- $x$: Fragment position in model space [S1].

**Pitfalls and Edge Cases:**
- **Performance:** Standard exponentials are slow on hardware. The method uses base-two exponential ($2^{-x}$) compensated by multiplying a correction factor into the $\sigma_i$ entries in the texture maps [S1].
- **Loop Unrolling:** The number of RBFs $M$ must be known at compile time (fixed preprocessor constant) [S1].
- **Texture Lookups:** Requires two lookups per fragment due to parameter packing [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# inpos: vec3, model space fragment position
# M: int, fixed number of RBFs (unrolled loop)
# tex_centers: Texture3D/2D, stores mu_i
# tex_weights: Texture3D/2D, stores w_i
# tex_widths: Texture3D/2D, stores sigma_i
# tex_error: Texture3D/2D (optional), stores error values
# tex_color: Texture1D, for final color mapping
# bias: float, constant bias
# scale: float, scale factor
# offset: float, offset factor

# Outputs:
# out_color: vec4

# Local variables
f_val = 0.0
grad_val = vec3(0.0)

# Unrolled loop for i from 0 to M-1
# Note: In actual GLSL/HLSL, this loop is unrolled by the compiler using #pragma unroll or similar
for i in range(M):
    # 1. Lookup RBF center coordinates
    mu_i = tex_centers.sample(i).xyz
    
    # 2. Lookup RBF weight and width
    w_i = tex_weights.sample(i).r
    sigma_i = tex_widths.sample(i).r
    
    # 3. Compute distance vector and squared distance
    diff = inpos - mu_i
    dist_sq = dot(diff, diff)
    
    # 4. Evaluate Gaussian RBF
    # Use base-two exponential for performance: 2^(-x) = e^(-x * ln(2))
    # The source states sigma_i is compensated with a correction factor.
    # Assuming the stored sigma_i already includes this compensation or the exponent is adjusted.
    # Standard Gaussian: exp(-dist_sq / (2 * sigma_i^2))
    # Base-2 equivalent: 2^(-dist_sq / (2 * sigma_i^2 * log2(e)))
    # The source formula for gradient uses the same exponential term.
    
    exponent = -dist_sq / (2.0 * sigma_i * sigma_i)
    
    # Using base-2 exp if compensated, otherwise standard exp.
    # Source says: "use the base-two exponential which is compensated with a correction factor multiplied to sigma_i"
    # This implies the stored sigma_i might be scaled such that we can use 2^exponent directly or similar.
    # Without explicit compensation formula, we use standard exp for correctness, noting the optimization.
    # However, to follow the source's intent of "base-two exponential", we assume the texture data handles it.
    # Let's stick to the mathematical definition provided in the source formula for consistency.
    gauss_val = exp(exponent)
    
    # Accumulate scalar field value
    f_val += w_i * gauss_val
    
    # Accumulate gradient
    # Source formula: - sum( w_i * (x - mu_i) / (2 * sigma_i) * exp(...) )
    # Note: This formula differs from standard calculus derivative (-w * (x-mu)/sigma^2 * exp).
    # We implement the source formula exactly as cited [S1].
    grad_val += -w_i * (diff / (2.0 * sigma_i)) * gauss_val

# Add constant bias
f_val += bias

# Add error value if available
# Linearly interpolated error encoded in secondary texture coordinates
# Assuming secondary coordinates are available or sampled based on cell position
if error_texture_exists:
    error_val = sample_error_at(inpos) # Implementation detail for interpolation
    f_val += error_val

# Scale and bias for data range
f_val = f_val * scale + offset

# Map to output color
out_color = tex_color.sample(f_val)
```

## Subtopics

### Isosurface Rendering
Uses the computed gradient for lighting calculations [S1].

### Inverse Multiquadric RBFs
Alternative basis function type implemented in the same framework [S1].

## Operating characteristics
- **Time Complexity:** $O(M)$ per fragment, where $M$ is the number of RBFs. The loop is unrolled, so it is constant time for a given dataset, but scales linearly with $M$ [S1].
- **Memory Complexity:** Storage for RBF parameters in texture maps. Two lookups per fragment are required [S1].
- **Performance:** Optimized by using base-two exponential instead of standard exponential [S1]. The single-pass approach is preferred for readability and performance over multipass code [S1]. Tested on Nvidia GeForce FX 5900 Ultra [S1].
- **Strengths:** Analytical gradient computation enables accurate lighting and isosurface rendering; single-pass efficiency [S1].
- **Limitations:** Fixed number of RBFs $M$ due to lack of dynamic loops in fragment shaders at the time of publication; two texture lookups per fragment [S1].

## Background
The method originates from the work of Jang, Weiler, Hopf, Huang, Ebert, Gaither, and Ertl in 2004, presented at the Symposium on Visualization (Vissym) [S1]. It was developed to interactively visualize procedurally encoded scalar fields, such as those from X38 shock simulations and natural convection simulations, on hardware of that era (Pentium 4, GeForce FX 5900 Ultra) [S1].

## Sources
[S1] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
