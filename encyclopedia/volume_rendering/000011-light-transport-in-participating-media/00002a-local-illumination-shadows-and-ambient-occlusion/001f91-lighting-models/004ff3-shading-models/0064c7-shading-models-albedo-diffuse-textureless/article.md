# Shading Models (Albedo, Diffuse, Textureless)

Also called: Albedo, Diffuse, Textureless Shading, Three Shading Types

## Overview
This method defines three distinct shading primitives for volume rendering: albedo (direct color), diffuse (lighting with normals), and textureless (white diffuse). It is primarily used in optimization-based 3D reconstruction pipelines to provide varied visual feedback during training. It differs from sibling models like [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) by using a fixed point-light approximation and a stochastic switching schedule rather than a continuous analytical BRDF.

## How it works
The method operates on a volumetric representation defined by density $\sigma$ and color $c$. The core logic involves computing surface normals from the density field and applying one of three shading equations based on a stochastic schedule.

**Inputs:**
- Density field $\sigma(u)$ and color field $c(u)$ at position $u$.
- Light position $l$ and light color $l_\rho$.
- Ambient light color $l_a$.
- Current training step count.

**Outputs:**
- Rendered color $I(u)$ at position $u$.

**Data Structures:**
- Position $u$: 3D vector $(u_x, u_y, u_z)$.
- Normal $n$: 3D vector, normalized.
- Light parameters: $l$ (3D vector), $l_\rho$ (RGB vector), $l_a$ (RGB vector).

**Step 1: Compute Surface Normal**
The surface normal $n$ is computed using central finite differences on the density field $\sigma$ with step size $\epsilon$.
$$
n = \frac{1}{2\epsilon} \begin{pmatrix} \sigma(u + \epsilon_x) - \sigma(u - \epsilon_x) \\ \sigma(u + \epsilon_y) - \sigma(u - \epsilon_y) \\ \sigma(u + \epsilon_z) - \sigma(u - \epsilon_z) \end{pmatrix}
$$
where $\epsilon_x = (\epsilon, 0, 0)$, $\epsilon_y = (0, \epsilon, 0)$, and $\epsilon_z = (0, 0, \epsilon)$. The resulting vector is normalized to unit length. [G] In practice, $\epsilon$ is a small constant relative to the scene scale (e.g., $10^{-3}$ to $10^{-2}$). If the density gradient is zero (degenerate), $n$ defaults to $(0,0,1)$ or is skipped.

**Step 2: Determine Shading Type**
The shading type is selected based on the rendering context:
1. **Reconstruction View:** Always uses **Albedo** shading.
2. **Random View (Training):**
   - If step $< 1000$: Use **Albedo** shading with $l_a = 1.0$ and $l_\rho = 0.0$.
   - If step $\ge 1000$: Stochastically select between Albedo, Diffuse, and Textureless with probabilities $0.2$, $0.4$, and $0.4$ respectively. Use $l_a = 0.1$ and $l_\rho = 0.9$.

**Step 3: Calculate Color**
Let $I_\rho(u)$ be the base color output by the volume model $R(u; \sigma, c)$.

- **Albedo Shading:**
  $$
  I(u) = I_\rho(u)
  $$

- **Diffuse Shading:**
  $$
  I(u) = I_\rho(u) \circ \left( l_\rho \circ \max\left(0, n \cdot \frac{l-u}{\|l-u\|}\right) + l_a \right)
  $$
  where $\circ$ denotes element-wise multiplication.

- **Textureless Shading:**
  Same as Diffuse, but replace $I_\rho(u)$ with white $(1, 1, 1)$.
  $$
  I(u) = (1, 1, 1) \circ \left( l_\rho \circ \max\left(0, n \cdot \frac{l-u}{\|l-u\|}\right) + l_a \right)
  $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_normal(density_field, u, epsilon=1e-3):
    """
    Compute surface normal using central finite differences.
    density_field: callable or array-like returning density at u
    u: numpy array (3,)
    epsilon: float step size
    """
    eps_x = np.array([epsilon, 0, 0])
    eps_y = np.array([0, epsilon, 0])
    eps_z = np.array([0, 0, epsilon])
    
    grad_x = density_field(u + eps_x) - density_field(u - eps_x)
    grad_y = density_field(u + eps_y) - density_field(u - eps_y)
    grad_z = density_field(u + eps_z) - density_field(u - eps_z)
    
    normal = np.array([grad_x, grad_y, grad_z]) / (2 * epsilon)
    norm_len = np.linalg.norm(normal)
    if norm_len > 1e-8:
        return normal / norm_len
    else:
        return np.array([0.0, 0.0, 1.0]) # Default normal

def apply_shading(u, l, l_rho, l_a, shading_type, density_field, color_model):
    """
    Apply shading model to volume data.
    shading_type: 'albedo', 'diffuse', or 'textureless'
    color_model: callable returning base color I_rho(u)
    """
    # 1. Compute Normal
    n = compute_normal(density_field, u)
    
    # 2. Get Base Color
    if shading_type == 'textureless':
        i_rho = np.array([1.0, 1.0, 1.0])
    else:
        i_rho = color_model(u)
        
    # 3. Calculate Lighting
    if shading_type == 'albedo':
        return i_rho
    else:
        # Diffuse or Textureless
        direction = l - u
        dist = np.linalg.norm(direction)
        if dist > 1e-8:
            light_dir = direction / dist
        else:
            light_dir = np.array([0.0, 0.0, 0.0])
            
        # Diffuse term: max(0, n . light_dir)
        diffuse_factor = max(0.0, np.dot(n, light_dir))
        
        # Lighting equation: l_rho * diffuse + l_a
        lighting = l_rho * diffuse_factor + l_a
        
        # Element-wise multiplication with base color
        return i_rho * lighting

def render_step(u, l, step_count, density_field, color_model):
    """
    Main entry point for shading selection.
    """
    # Parameters for random view
    l_a = np.array([0.1, 0.1, 0.1])
    l_rho = np.array([0.9, 0.9, 0.9])
    
    # Reconstruction view always uses albedo
    # Assuming this function is called for random views if step_count logic applies
    # If this is reconstruction view, force albedo
    # Note: The source specifies "For the reconstruction view, we only use albedo shading."
    # This logic is typically handled by the caller, but here we implement the random view schedule.
    
    if step_count < 1000:
        shading_type = 'albedo'
        l_a = np.array([1.0, 1.0, 1.0])
        l_rho = np.array([0.0, 0.0, 0.0])
    else:
        # Stochastic selection
        r = np.random.random()
        if r < 0.2:
            shading_type = 'albedo'
        elif r < 0.6: # 0.2 + 0.4
            shading_type = 'diffuse'
        else:
            shading_type = 'textureless'
            
    return apply_shading(u, l, l_rho, l_a, shading_type, density_field, color_model)
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for shading calculation, assuming density evaluation is $O(1)$. Normal computation requires 6 density evaluations.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Simple to implement; provides diverse visual cues for optimization (albedo for color, diffuse for shape, textureless for geometry-only).
- **Limitations:** Uses a single point light; no shadow rays; stochastic switching may introduce noise if not averaged over multiple samples.
- **When to use:** Optimization-based 3D reconstruction (e.g., NeRF, Gaussian Splatting variants) where view-dependent shading helps regularize geometry.
- **When not to use:** Real-time rendering requiring consistent lighting; scenes with complex global illumination.

## Background
Originated in the RealFusion paper [S1] (2023) for 360° reconstruction from a single image. It is used within the rendering loop of the optimization process to guide the learning of density and color fields. The method builds on [Shading models](837-shading-models.md) by providing specific implementations for albedo, diffuse, and textureless shading.

## Sources
[S1] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
