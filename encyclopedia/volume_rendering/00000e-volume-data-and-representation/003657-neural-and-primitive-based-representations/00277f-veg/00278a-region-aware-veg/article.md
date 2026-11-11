# Region-Aware VEG

Also called: Region-Aware VEG Baseline, VEG with multi-region condition

## Overview
Region-Aware VEG extends the Vector-Encoded Gaussian (VEG) method to support multi-region conditional transfer functions by assigning each Gaussian a fixed categorical region and a learnable scalar coordinate [S1, unofficial]. It differs from standard VEG by enabling distinct color-opacity mappings for different semantic regions within the same volume, allowing for complex, region-specific visualizations without increasing the primitive count [S1, unofficial].

## How it works
Region-Aware VEG builds on [VEG](4089-veg.md) [S1, unofficial]. It modifies the VEG representation to support a multi-region condition $R_T(m, h)$, where $m$ is a region label and $h$ is a normalized intensity coordinate [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: The method takes an input point cloud for initialization, $R$ authored transfer function presets, $BV$ (the number of intensity knots), learning rates, and training iterations [S1, unofficial].
*   **Outputs**: The method outputs a set of Gaussians with attributes including position, scale, rotation, opacity, color (computed via shading), region label, and learnable scalar [S1, unofficial].

### Data Structures
Each Gaussian primitive $i$ stores position $\mathbf{p}_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, opacity weight $w_i$, normal $\mathbf{n}_i$, region label $m_i$, learnable scalar $v_i$, and shading parameters (ambient, diffuse, specular factors, and shininess $s_i$) [S1, unofficial]. The native-quantized representation uses 256-center codebooks for opacity, scale, rotation, normal, scalar, diffuse, and ambient attributes, while codebook centers and remaining continuous attributes use half precision [S1, unofficial].

### Algorithm Steps
1.  **Initialization**: Initialize positions, scalars ($v_i$), and normals ($\mathbf{n}_i$) using VEG's native random draws from the input point cloud's bounding box and point count [S1, unofficial]. Assign each primitive to the nearest source seed's density-weighted majority region to determine the fixed categorical region $m_i$ [S1, unofficial].
2.  **Transfer Function Preparation**: Resample the $R$ authored transfer function curves to VEG's native $BV = 100$ intensity knots to form the lookup table $R_T^{(BV)}$ [S1, unofficial].
3.  **Intensity Coordinate Computation**: Compute the within-region normalized intensity coordinate $h_i$ using the sigmoid of the learnable scalar $v_i$:
    $$ h_i = (BV - 1) \cdot \text{sigmoid}(v_i) $$
    where $\text{sigmoid}(x) = \frac{1}{1 + e^{-x}}$ [S1, unofficial].
4.  **Color and Opacity Lookup**: Perform linear interpolation only between adjacent intensity bins of the same region $m_i$ to determine the base color $\mathbf{c}_i(T)$ and opacity $a_i(T)$:
    $$ (\mathbf{c}_i(T), a_i(T)) = \text{lerp}\left( R_T^{(BV)}(m_i, \lfloor h_i \rfloor), R_T^{(BV)}(m_i, \lfloor h_i \rfloor + 1), h_i - \lfloor h_i \rfloor \right) $$
    *Note: The source formula (14) contains a typo listing the same term twice; standard linear interpolation between bin $\lfloor h_i \rfloor$ and $\lfloor h_i \rfloor + 1$ is used here [G].* Include the authored region-visibility flag in the transfer function alpha [S1, unofficial].
5.  **Shading and Final Opacity**: Query the color $\mathbf{c}_i(T)$ and enter VEG's Blinn-Phong shading model using learned normals, ambient, diffuse, specular factors, and shininess $s_i$ [S1, unofficial]. Compute the final opacity $\alpha_i$ as:
    $$ \alpha_i = w_i \cdot a_i(T) $$
    where $w_i$ is VEG's independent sigmoid-activated weight [S1, unofficial].
6.  **Topology Management**: Region labels $m_i$ remain fixed under motion [S1, unofficial]. During native cloning and splitting, region labels follow the actual parent indices [S1, unofficial].
7.  **Numerical Stability Fix**: The native specular term $|\mathbf{n}_i^\top \mathbf{h}|^{s_i}$ produces non-finite gradients when $0 < s_i < 1$ and the input is exactly zero [S1, unofficial]. Fix: mask exact-zero inputs before exponentiation and assign zero gradients at those inputs, preserving forward values and gradients elsewhere [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))

def region_aware_veg_lookup(v_i, m_i, BV, R_T_BV):
    """
    Computes color and opacity for a Gaussian based on its region and scalar.
    Args:
        v_i: Learnable scalar for the Gaussian.
        m_i: Fixed categorical region label.
        BV: Number of intensity knots (e.g., 100).
        R_T_BV: Lookup table for transfer functions, shape (R, BV, 4) 
                where R is number of regions, BV is knots, 4 is (r,g,b,a).
    Returns:
        color: RGB color vector.
        base_opacity: Scalar opacity value from TF.
    """
    # Step 3: Compute intensity coordinate
    h_i = (BV - 1) * sigmoid(v_i)
    
    # Step 4: Linear interpolation within region
    h_floor = int(math.floor(h_i))
    h_ceil = min(h_floor + 1, BV - 1)
    
    # Get transfer function values for the region
    val_low = R_T_BV[m_i, h_floor]  # [r, g, b, a]
    val_high = R_T_BV[m_i, h_ceil]  # [r, g, b, a]
    
    # Fractional part for interpolation
    frac = h_i - h_floor
    
    # Interpolate color and opacity
    color = val_low[:3] + frac * (val_high[:3] - val_low[:3])
    base_opacity = val_low[3] + frac * (val_high[3] - val_low[3])
    
    return color, base_opacity

def apply_shading_and_opacity(color, w_i, base_opacity, normal, light_dir, view_dir, ambient, diffuse, specular_factor, shininess):
    """
    Applies Blinn-Phong shading and computes final opacity.
    Note: Standard Blinn-Phong formulation used [G].
    """
    n = normal / np.linalg.norm(normal)
    l = light_dir / np.linalg.norm(light_dir)
    v = view_dir / np.linalg.norm(view_dir)
    h = l + v
    h = h / np.linalg.norm(h)
    
    diff = max(0.0, np.dot(n, l))
    nh_dot = np.dot(n, h)
    
    # Step 7: Numerical stability fix for specular term
    if nh_dot == 0.0:
        spec = 0.0
    else:
        spec = np.power(nh_dot, shininess)
        
    shaded_color = color * ambient + color * diffuse * diff + specular_factor * spec
    final_opacity = w_i * base_opacity
    
    return shaded_color, final_opacity
```

### Parameters
*   **BV**: Number of intensity knots in VEG's native transfer function representation. Value: 100 [S1, unofficial].
*   **Learning Rate (Scalar $v_i$)**: Value: $2.5 \times 10^{-4}$ [S1, unofficial].
*   **Learning Rate (Opacity Weight $w_i$)**: Value: $2.5 \times 10^{-2}$ [S1, unofficial].
*   **Training Iterations**: Value: 30,000 [S1, unofficial].
*   **Authored Presets**: Number of authored presets used for training. Value: 24 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Numerical Instability**: The specular term $|\mathbf{n}_i^\top \mathbf{h}|^{s_i}$ produces non-finite gradients when $0 < s_i < 1$ and the input is exactly zero. This interrupts training. The fix is to mask exact-zero inputs before exponentiation and assign zero gradients at those inputs [S1, unofficial].
*   **Quantization Error**: Using 256-center codebooks for attributes lowers mean PSNR by 0.15 dB on validation, 0.12 dB on interpolation, 0.10 dB on composition, and 0.05 dB on OOD compared to float evaluation [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Scales linearly with the number of primitives, independently of the number of transfer functions, as persistent descriptors contain at most $N_K$ weighted region–intensity samples [S1, unofficial].
*   **Memory Complexity**: Linear with the number of primitives. The native-quantized representation uses half precision for codebook centers and continuous attributes, with 256-center codebooks for specific attributes [S1, unofficial].
*   **Performance**: Quantization lowers mean PSNR by 0.15/0.12/0.10/0.05 dB on validation/interpolation/composition/OOD splits compared to float evaluation [S1, unofficial].
*   **Strengths**: Supports multi-region conditional transfer functions without increasing primitive count. Fixed region labels simplify topology management during motion [S1, unofficial].
*   **Limitations**: Requires pre-defined regions and transfer function presets. Quantization introduces minor quality loss. Numerical stability requires specific handling of the specular term [S1, unofficial].

## Background
Region-Aware VEG is an extension of the Vector-Encoded Gaussian (VEG) method, originally proposed by Dyken et al. (2026) [S1, unofficial]. It was introduced to address the limitation of standard VEG in handling multi-region conditional transfer functions, which are common in medical volume rendering where different anatomical structures require distinct visual encodings [S1, unofficial]. The method retains the upstream optimizer, image losses, regularizers, and weight-based pruning of the original VEG implementation [S1, unofficial].

## Sources
[S1] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
