# Pre-Integrated Diffuse Shading with Non-Linear Gradient Interpolation

Also called: Pre-Integrated Diffuse Shading

## Overview
This method computes the diffuse component of Blinn-Phong shading in pre-integrated volume rendering by decomposing the integral into four 2D look-up tables weighted by coefficients dependent on light direction and gradient angles [S1]. It avoids the prohibitive memory cost of full 4D normal-based pre-integration while reducing shading errors caused by large angles between consecutive sample gradients compared to linear interpolation [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Front scalar value $S_f$ and back scalar value $S_b$ for the current sampling step of length $D$ [S1].
    *   Front normal $\mathbf{N}_f$ and back normal $\mathbf{N}_b$ (gradients of the scalar field) [S1].
    *   Light direction $\mathbf{L}$ [S1].
    *   Diffuse coefficient $K_d$ [S1].
    *   Pre-computed 2D LUTs for $I_0, I_1, I_2, I_3$ [S1].
*   **Outputs:**
    *   Pre-integrated diffuse color $\tilde{C}_d$ [S1].

### Data Structures
*   **Lookup Tables:** Four 2D arrays indexed by quantized front and back scalar values [S1]. The exact array dimensions and binning strategy are not given in the sources [S1].

### Formulas
The pre-integrated diffuse color is calculated as:
$$ \tilde{C}_d = K_d \cdot (a_0(\mathbf{L})I_0 + a_1(\mathbf{L})I_1 + a_2(\mathbf{L})I_2 + a_3(\mathbf{L})I_3) $$
where $K_d$ is the diffuse coefficient [S1].

The integral terms $I_k(S_f, S_b)$ are defined as:
$$ I_k(S_f, S_b) = d \int_{S_f}^{S_b} c(s_0) \cdot s^k \cdot \exp(-d \cdot \sigma(S_f, s_0)) \, ds_0 $$
where:
*   $k \in \{0, 1, 2, 3\}$ is the polynomial exponent [S1].
*   $c(s_0)$ is the classification color at scalar value $s_0$ [S1].
*   $\sigma(S_f, s_0)$ is the accumulated absorption from $S_f$ to $s_0$ [S1].
*   $d = \frac{D}{S_b - S_f}$ is the normalized step length [S1].
*   $s = \frac{s_0 - S_f}{S_b - S_f}$ is the normalized position within the step [S1].

The coefficients $a_k(\mathbf{L})$ depend on the light direction and normals:
$$
\begin{aligned}
a_0(\mathbf{L}) &= \mathbf{N}_f \cdot \mathbf{L} \\
a_1(\mathbf{L}) &= \mathbf{N}_b \cdot \mathbf{L} - (1 + r)\mathbf{N}_f \cdot \mathbf{L} \\
a_2(\mathbf{L}) &= r(2\mathbf{N}_f \cdot \mathbf{L} - \mathbf{N}_b \cdot \mathbf{L}) \\
a_3(\mathbf{L}) &= r(\mathbf{N}_b \cdot \mathbf{L} - \mathbf{N}_f \cdot \mathbf{L})
\end{aligned}
$$
where $r$ is a factor derived from the angle $\alpha$ between the front and back normals:
$$ r = \frac{4(\cos(\alpha) - 1)}{\cos(\alpha)} $$
$$ \alpha = \frac{1}{2} \arccos(\mathbf{N}_f \cdot \mathbf{N}_b) $$
All coefficients and the factor $r$ are computed per sampling step [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def precompute_luts(num_bins, scalar_min, scalar_max, classification_func, absorption_func):
    """
    Pre-compute the four 2D lookup tables I0, I1, I2, I3.
    classification_func(s) returns color c(s) as a list/tuple of 3 floats.
    absorption_func(sf, s) returns sigma(Sf, s) as a float.
    """
    # Initialize 2D tables with zeros [G]
    LUT_I0 = [[None] * num_bins for _ in range(num_bins)]
    LUT_I1 = [[None] * num_bins for _ in range(num_bins)]
    LUT_I2 = [[None] * num_bins for _ in range(num_bins)]
    LUT_I3 = [[None] * num_bins for _ in range(num_bins)]
    
    # Discretize scalar range [G]
    scalars = [scalar_min + i * (scalar_max - scalar_min) / (num_bins - 1) for i in range(num_bins)]
    
    for i, Sf in enumerate(scalars):
        for j, Sb in enumerate(scalars):
            if Sb <= Sf:
                continue  # Degenerate case [G]
            
            # Normalized step length for table indexing [S1]
            D_norm = 1.0
            d = D_norm / (Sb - Sf)
            
            # Accumulators for RGB channels [G]
            acc_I0 = [0.0, 0.0, 0.0]
            acc_I1 = [0.0, 0.0, 0.0]
            acc_I2 = [0.0, 0.0, 0.0]
            acc_I3 = [0.0, 0.0, 0.0]
            
            num_steps = 100  # Resolution for pre-computation integration [G]
            ds0 = (Sb - Sf) / num_steps
            
            for step_idx in range(num_steps):
                s0 = Sf + step_idx * ds0
                s = (s0 - Sf) / (Sb - Sf)
                
                c_s0 = classification_func(s0)
                sigma = absorption_func(Sf, s0)
                weight = math.exp(-d * sigma)
                
                # I_k = d * integral(c(s0) * s^k * exp(-d*sigma) ds0)
                for ch in range(3):
                    term = c_s0[ch] * weight * ds0 * d
                    acc_I0[ch] += term * (s ** 0)
                    acc_I1[ch] += term * (s ** 1)
                    acc_I2[ch] += term * (s ** 2)
                    acc_I3[ch] += term * (s ** 3)
                
            LUT_I0[i][j] = acc_I0
            LUT_I1[i][j] = acc_I1
            LUT_I2[i][j] = acc_I2
            LUT_I3[i][j] = acc_I3
            
    return LUT_I0, LUT_I1, LUT_I2, LUT_I3

def compute_diffuse_shading(LUT_I0, LUT_I1, LUT_I2, LUT_I3, Sf, Sb, Nf, Nb, L, Kd, num_bins, scalar_min, scalar_max):
    """
    Compute pre-integrated diffuse shading for a single step.
    """
    if Sb <= Sf:
        return [0.0, 0.0, 0.0]
        
    # 1. Compute angle alpha and factor r
    cos_alpha_raw = Nf[0]*Nb[0] + Nf[1]*Nb[1] + Nf[2]*Nb[2]
    cos_alpha_raw = max(-1.0, min(1.0, cos_alpha_raw))  # Clamp for numerical stability [G]
    alpha = math.acos(cos_alpha_raw) / 2.0
    cos_alpha = math.cos(alpha)
    
    # Handle division by zero when alpha approaches 90 degrees [G]
    if abs(cos_alpha) < 1e-6:
        r = -4.0  # Limit approximation [G]
    else:
        r = 4.0 * (cos_alpha - 1.0) / cos_alpha
        
    # 2. Compute coefficients a0 to a3
    Nf_dot_L = Nf[0]*L[0] + Nf[1]*L[1] + Nf[2]*L[2]
    Nb_dot_L = Nb[0]*L[0] + Nb[1]*L[1] + Nb[2]*L[2]
    
    a0 = Nf_dot_L
    a1 = Nb_dot_L - (1.0 + r) * Nf_dot_L
    a2 = r * (2.0 * Nf_dot_L - Nb_dot_L)
    a3 = r * (Nb_dot_L - Nf_dot_L)
    
    # 3. Quantize indices [G]
    idx_Sf = int((Sf - scalar_min) / (scalar_max - scalar_min) * (num_bins - 1))
    idx_Sb = int((Sb - scalar_min) / (scalar_max - scalar_min) * (num_bins - 1))
    idx_Sf = max(0, min(num_bins - 1, idx_Sf))
    idx_Sb = max(0, min(num_bins - 1, idx_Sb))
    
    I0 = LUT_I0[idx_Sf][idx_Sb]
    I1 = LUT_I1[idx_Sf][idx_Sb]
    I2 = LUT_I2[idx_Sf][idx_Sb]
    I3 = LUT_I3[idx_Sf][idx_Sb]
    
    # 4. Compute final color
    sum_I = [0.0, 0.0, 0.0]
    for ch in range(3):
        sum_I[ch] = a0 * I0[ch] + a1 * I1[ch] + a2 * I2[ch] + a3 * I3[ch]
        
    C_d = [Kd * sum_I[ch] for ch in range(3)]
    return C_d
```

### Edge Cases and Pitfalls
*   **Negative Dot Products:** The dot product between the light direction and the normalized gradient may become negative, which is physically invalid for diffuse shading [S1]. Two solutions are common: applying `max(0, dot)` to the dot products before computing coefficients, or mirroring the gradient if the dot product is negative [S1]. Applying `max(0, dot)` makes exact pre-integration difficult, so the source distinguishes three cases but does not detail them [S1].
*   **Degenerate Normals:** If $\mathbf{N}_f$ and $\mathbf{N}_b$ are identical, $\alpha = 0$, $r = 0$, and the method reduces to standard linear shading [S1]. If they are orthogonal, $\alpha = \pi/4$ [S1].
*   **Precision:** The factor $r$ can become large negative values for large angles, potentially amplifying numerical errors in the coefficient calculation [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Not given in the sources [S1].
*   **Limitations:** Not given in the sources [S1].
*   **When to use:** Not given in the sources [S1].

## Background
This method was proposed by Lum et al. [S1] in 2010 as an improvement over linear gradient interpolation in pre-integrated volume rendering [S1]. It addresses visual discontinuities and errors that arise when gradients change significantly between sample points, which linear interpolation fails to capture accurately [S1]. The approach builds on the concept of pre-integration lookup tables introduced in earlier work, extending it to handle diffuse shading with a non-linear gradient model [S1].

## Sources
[S1] Pre-Integrated Volume Rendering with Non-Linear ..., https://dpt-info.u-strasbg.fr/~dischler/papers/Vis2010.pdf, fetched 2026-10-28
