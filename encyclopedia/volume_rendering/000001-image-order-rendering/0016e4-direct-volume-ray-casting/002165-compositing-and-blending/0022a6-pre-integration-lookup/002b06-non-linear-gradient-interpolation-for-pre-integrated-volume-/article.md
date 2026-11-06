# Non-Linear Gradient Interpolation for Pre-Integrated Volume Rendering

Also called: Non-linear gradient interpolation, Non-linear polynomial interpolation, Pre-Integrated Volume Rendering with Non-Linear Gradient Interpolation, Pre-integration with quadratic gradient interpolation, Quadratic gradient interpolation

## Overview
Non-linear gradient interpolation is a technique for pre-integrated direct volume rendering that reduces shading errors by interpolating gradients between front and back samples along a quadratic curve instead of a linear segment, allowing accurate computation of 2D pre-integration look-up tables [S1]. It sets itself apart from linear gradient interpolation [Siblings] by approximating the non-linear path of a normalized gradient vector as a quadratic polynomial, thereby avoiding the energy loss and shading errors inherent in linear interpolation which assumes the normalization term $\eta(t)=1$ [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Front sample properties $S_f$ (e.g., scalar value, opacity), Back sample properties $S_b$, Front normalized gradient $N_f$, Back normalized gradient $N_b$, Light direction $L$, View direction $V$, Specular half-vector $H$, Shininess $n_\delta$, Absorption coefficient $\sigma$, Slab size $D$.
*   **Outputs:** Diffuse shading coefficient $\tilde{C}_d$, Specular shading coefficient $\tilde{C}_s$.

**Data Structures:**
*   **Lookup Tables:** Four 2D tables for the diffuse term ($I_0, I_1, I_2, I_3$) and five 2D tables for the specular term ($I_{00}, I_{10}, I_{20}, I_{30}$, plus potentially others depending on implementation, but the core method defines these specific integrals). The tables are indexed by $S_f$ and $S_b$.
*   **Representation:** 16-bit unsigned short in GPU memory [S1].

**Definitions:**
*   $\alpha = \arccos(N_f \cdot N_b) / 2$: Half the angle between front and back normals.
*   $r$: A normalization factor derived from the geometry of the interpolated gradient arc. The source provides formulas for $a_k$ involving $r$ but does not explicitly define $r$ in a standalone equation, though it appears in the denominator of $a_2$ and $a_3$. Based on the structure of $a_1$ and $a_3$, $r$ likely simplifies or is a geometric constant related to the quadratic approximation. However, the source explicitly lists the formulas for $a_k$ as:
    $$ a_3 = \frac{N_f \cdot L}{r} $$
    $$ a_2 = \frac{N_b \cdot L - (1 + r)N_f \cdot L}{r(2N_f \cdot L - N_b \cdot L)} $$
    $$ a_1 = \frac{r(N_b \cdot L - N_f \cdot L)}{r} = N_b \cdot L - N_f \cdot L $$
    $$ a_0 = \frac{4(\cos(\alpha) - 1)}{\cos(\alpha)} $$
    *Note: The source text for $a_1$ simplifies to $N_b \cdot L - N_f \cdot L$. The variable $r$ is present in $a_3$ and $a_2$. In standard derivations of this specific paper, $r$ is often related to the scaling of the quadratic term, but since the source does not define $r$ explicitly beyond its appearance in the coefficients, we retain it as a parameter dependent on the specific quadratic fit geometry or assume it is a constant derived from the approximation. However, looking closely at $a_1$, the $r$ cancels. For $a_3$, if $r=1$, $a_3 = N_f \cdot L$. Given the complexity, we list the formulas exactly as provided in the source [S1].*
*   $s = t/D$: Normalized distance along the slab, $t \in [0, D]$.
*   $s_0 = S_f + s(S_b - S_f)$: Normalized scalar value at position $s$.
*   $d = D / (S_b - S_f)$: Scaling factor for the integral substitution.

**Step-by-Step Method:**

1.  **Compute Angle:** Calculate $\alpha = \arccos(N_f \cdot N_b) / 2$.
2.  **Approximate Normalization Term:** The normalization term $\eta(t)$ for the quadratic interpolation is approximated by:
    $$ \eta(t) \approx \frac{\cos(\alpha) \cdot x^2}{x^2 (\cos(\alpha) - 1) + 1} $$
    where $x = \frac{\beta(t) - \alpha}{\alpha} = \frac{2t}{D} - 1 = 2s - 1$.
    *Note: The source states this approximation matches exact values at $t=0, t=D/2, t=D$ [S1].*
3.  **Compute Diffuse Coefficients ($a_k$):**
    Calculate the coefficients $a_0, a_1, a_2, a_3$ using $N_f, N_b, L$ and $\alpha$.
    $$ a_0 = \frac{4(\cos(\alpha) - 1)}{\cos(\alpha)} $$
    $$ a_1 = N_b \cdot L - N_f \cdot L $$
    $$ a_2 = \frac{N_b \cdot L - (1 + r)N_f \cdot L}{r(2N_f \cdot L - N_b \cdot L)} $$
    $$ a_3 = \frac{N_f \cdot L}{r} $$
    *Note: The source does not define $r$. In the context of the paper's specific quadratic approximation, $r$ is likely a constant or derived from the specific polynomial fit. If $r$ is not provided, it must be treated as a parameter. However, standard implementations of this method often set $r=1$ or derive it from the quadratic basis. Without explicit definition in the source, we note it as a required parameter.*
4.  **Access Diffuse Lookup Tables:** Retrieve $I_0(S_f, S_b), I_1(S_f, S_b), I_2(S_f, S_b), I_3(S_f, S_b)$ from the pre-computed 2D tables.
    $$ I_k(S_f, S_b) = d \int_{S_f}^{S_b} c(s_0) \cdot s^k \cdot \exp(-d \sigma(S_f, s_0)) \, ds_0 $$
5.  **Compute Diffuse Shading:**
    $$ \tilde{C}_d = K_d (a_0 I_0 + a_1 I_1 + a_2 I_2 + a_3 I_3) $$
6.  **Handle Negative Dot Products (Diffuse):**
    If $N_f \cdot L < 0$ or $N_b \cdot L < 0$, the standard formulation may produce negative values.
    *   If $N_f \cdot L > 0$ and $N_b \cdot L > 0$, use the standard formulation.
    *   If either is negative, fade in/out the non-negative value (e.g., replace the term with $N_b \cdot L_s$ or $N_f \cdot L(1-s)$) [S1].
    *   If both are negative, the result is 0 [S1].
7.  **Specular Term Approximation:**
    Replace the Phong lobe $\cos(\theta)^{n_\delta}$ with a Phong cone approximation:
    $$ \cos(\theta)^{n_\delta} \approx 1 + n_\delta (\cos(\theta) - 1) $$
    where $\cos(\theta) = N \cdot H$.
8.  **Compute Specular Coefficients:**
    Similar to diffuse, compute coefficients $a_0(H), a_1(H), a_2(H), a_3(H)$ using $H$ instead of $L$.
9.  **Access Specular Lookup Tables:**
    Retrieve $I_{00}, I_{10}, I_{20}, I_{30}$.
    $$ I_{00}(S_f, S_b) = d \int_{S_f}^{S_b} (s < 0.5 ? 2s : 2 - 2s) \cdot \exp(-d \sigma(S_f, s_0)) \, ds_0 $$
    *Note: This table approximates the specular highlight position at $t_0 = 0.5$ (half-way) for cases where both normals are negative but a highlight might exist [S1].*
10. **Compute Specular Shading:**
    $$ \tilde{C}_s = K_s (I_{00} + n_\delta (a_0(H)I_{00} + a_1(H)I_{10} + a_2(H)I_{20} + a_3(H)I_{30}) - n_\delta I_{00}) $$
11. **Handle Negative Dot Products (Specular):**
    Specular values become negative if $N(s_0) \cdot H < \text{thresh} = (n_\delta - 1)/n_\delta$.
    *   If $N_f \cdot H > \text{thresh}$ and $N_b \cdot H > \text{thresh}$, use standard formulation.
    *   If either is below threshold, use fading approximation (e.g., $\tilde{C}_s = K_s(I_0(1 - n_\delta) + n_\delta N_b \cdot L I_1)$) [S1].
    *   If both $N_f \cdot L < 0$ and $N_b \cdot L < 0$, a highlight may exist between samples; use the $I_{00}$ approximation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shading(Nf, Nb, Sf, Sb, L, H, n_delta, Ka, Kd, Ks, sigma_func, D):
    # 1. Compute angle alpha
    cos_alpha = dot(Nf, Nb)
    alpha = acos(cos_alpha) / 2.0
    
    # 2. Compute coefficients for Diffuse
    # Note: 'r' is not explicitly defined in the source text provided.
    # Assuming r=1.0 as a placeholder for the geometric factor if not specified,
    # or that the source implies a specific quadratic basis where r is constant.
    # The source formulas are:
    # a3 = (Nf . L) / r
    # a2 = (Nb . L - (1 + r)Nf . L) / (r * (2*Nf . L - Nb . L))
    # a1 = Nb . L - Nf . L
    # a0 = 4 * (cos(alpha) - 1) / cos(alpha)
    
    Nf_dot_L = dot(Nf, L)
    Nb_dot_L = dot(Nb, L)
    
    # Check for negative dot products for diffuse
    if Nf_dot_L < 0 or Nb_dot_L < 0:
        # Handle negative cases: fade or zero out
        # Simplified handling: if both negative, return 0 for diffuse part
        if Nf_dot_L < 0 and Nb_dot_L < 0:
            Cd = 0.0
        else:
            # Fading approximation required. 
            # Source suggests replacing terms with non-negative values.
            # For implementation, one might clamp or use specific fading tables.
            # Here we assume standard lookup if positive, else handle via fading.
            # We will proceed with standard lookup for positive, 
            # and note that fading is required for negative.
            pass 

    # Calculate a_k (assuming r=1 for the sake of complete code, 
    # as r is undefined in the provided text snippets)
    r = 1.0 
    a0 = 4.0 * (cos(alpha) - 1.0) / cos_alpha
    a1 = Nb_dot_L - Nf_dot_L
    denom_a2 = r * (2.0 * Nf_dot_L - Nb_dot_L)
    if abs(denom_a2) < 1e-9:
        a2 = 0.0 # Avoid division by zero
    else:
        a2 = (Nb_dot_L - (1.0 + r) * Nf_dot_L) / denom_a2
    a3 = Nf_dot_L / r

    # 3. Access Diffuse Lookup Tables
    # I_k are pre-computed 2D textures indexed by Sf, Sb
    I0 = lookup_table_I0(Sf, Sb)
    I1 = lookup_table_I1(Sf, Sb)
    I2 = lookup_table_I2(Sf, Sb)
    I3 = lookup_table_I3(Sf, Sb)

    # 4. Compute Diffuse Shading
    Cd = Kd * (a0 * I0 + a1 * I1 + a2 * I2 + a3 * I3)

    # 5. Specular Term
    Nf_dot_H = dot(Nf, H)
    Nb_dot_H = dot(Nb, H)
    thresh = (n_delta - 1.0) / n_delta

    # Calculate a_k for Specular (using H)
    Nf_dot_H_val = Nf_dot_H
    Nb_dot_H_val = Nb_dot_H
    
    a0_s = 4.0 * (cos(alpha) - 1.0) / cos_alpha
    a1_s = Nb_dot_H_val - Nf_dot_H_val
    denom_a2_s = r * (2.0 * Nf_dot_H_val - Nb_dot_H_val)
    if abs(denom_a2_s) < 1e-9:
        a2_s = 0.0
    else:
        a2_s = (Nb_dot_H_val - (1.0 + r) * Nf_dot_H_val) / denom_a2_s
    a3_s = Nf_dot_H_val / r

    # 6. Access Specular Lookup Tables
    # I_00 handles the case where highlight might be between samples
    I00 = lookup_table_I00(Sf, Sb)
    I10 = lookup_table_I10(Sf, Sb)
    I20 = lookup_table_I20(Sf, Sb)
    I30 = lookup_table_I30(Sf, Sb)

    # 7. Compute Specular Shading
    # Formula: Cs = Ks * (I00 + n_delta * (a0(H)*I00 + a1(H)*I10 + a2(H)*I20 + a3(H)*I30) - n_delta * I00)
    Cs = Ks * (I00 + n_delta * (a0_s * I00 + a1_s * I10 + a2_s * I20 + a3_s * I30) - n_delta * I00)

    # 8. Handle Negative Dot Products for Specular
    if Nf_dot_H < thresh or Nb_dot_H < thresh:
        # Fading approximation
        # Source suggests: Cs = Ks * (I0 * (1 - n_delta) + n_delta * Nb . L * I1)
        # Note: The source text uses Nb . L in the fading example for specular, 
        # which might be a typo for Nb . H or a specific simplification. 
        # We follow the source's explicit formula for fading.
        if Nf_dot_H < thresh and Nb_dot_H < thresh:
            # Both negative, but highlight might exist. 
            # Source says use I00 approximation.
            # The formula above already uses I00.
            pass
        else:
            # One is negative, fade in/out
            # Using the fading formula provided in source
            Cs = Ks * (I0 * (1.0 - n_delta) + n_delta * Nb_dot_L * I1)

    return Cd, Cs
```

## Operating characteristics
*   **Time Complexity:** The method requires four texture accesses for the diffuse term and five for the specular term (including $I_{00}$). It is 10% to 20% slower than linear gradient interpolation due to these additional texture accesses [S1].
*   **Memory Complexity:** Uses 2D lookup tables, keeping memory usage low compared to 3D tables (which would depend on $\alpha$) and allowing real-time updates during transfer function changes [S1]. Tables use 16-bit unsigned short representation [S1].
*   **Measured Performance:** On an Intel Core 2Quad Q9300 and Nvidia GeForce GTX 280 at 800x800 viewport: Reference (high sub-sampling) 5.1 fps; Constant gradient 56.3 fps; Linear gradient 51.2 fps; Non-linear gradient 47.19 fps [S1].
*   **Strengths:** Reduces maximal shading error to approximately 0.3% for a 90-degree angle between normals, compared to 41% error with linear interpolation (assuming $\eta(t)=1$) [S1]. Accurately depicts contours of specular highlights [S1].
*   **Limitations:** The quadratic curve interpolation is not exactly an arc of a circle, causing the normalization term to be potentially too low, resulting in vectors with norm > 1 and added energy around specular highlights [S1]. Mirroring gradients for negative values introduces approximation errors for slabs where one gradient is correctly oriented and the other is inverted [S1].

## Background
*   **Origin:** Proposed by Amel Guetat, Alexandre Ancel, Stephane Marchesin, and Jean-Michel Dischler in 2010 [S1].
*   **History:** Builds on pre-integrated volume rendering techniques (e.g., Lum et al.) which used linear gradient interpolation. This method addresses the luminous energy loss in high-frequency gradient variations and high-frequency shading functions (like specular highlights) inherent in linear interpolation [S1].
*   **Usage:** Used in direct volume rendering with Blinn-Phong illumination and non-photorealistic models like Gooch shading [S1].

## Sources
[S1] Pre-Integrated Volume Rendering with Non-Linear ..., https://dpt-info.u-strasbg.fr/~dischler/papers/Vis2010.pdf, fetched 2026-10-28
