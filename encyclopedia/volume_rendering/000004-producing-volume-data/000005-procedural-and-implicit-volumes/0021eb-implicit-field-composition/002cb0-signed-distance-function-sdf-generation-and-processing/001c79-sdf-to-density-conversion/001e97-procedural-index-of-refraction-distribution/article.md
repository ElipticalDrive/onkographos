# Procedural Index of Refraction Distribution

Also called: Mirage Simulation via Non-Uniform IOR

## Overview
Procedural Index of Refraction Distribution generates a spatially varying index of refraction (IOR) field to simulate optical phenomena such as mirages, where light bends due to temperature-induced density gradients in air [S1, unofficial]. It differs from density-based volume methods by explicitly modeling the refractive properties of the medium rather than just absorption or emission, allowing for ray bending effects [S1, unofficial][S2].

## How it works
**Inputs:**
- Vertical coordinate $z$ (normalized or physical height).
- Layer definitions for normal and inversion profiles.

**Outputs:**
- A scalar field $n(x, y, z)$ representing the index of refraction at each point.

**Data Structures:**
- The IOR field is typically stored as a 3D texture or grid where each voxel contains a single floating-point value representing $n$.

**Steps:**
1. **Define Normal Lapse Layer:**
   Create a base profile where the IOR decreases exponentially with height $z$. The formula is:
   $$ n_{\text{normal}}(z) = 0.95 \times 2^z $$
   Here, $z$ represents the vertical position [S1, unofficial].

2. **Define Inversion Layer:**
   Create a secondary profile representing an inversion layer, where the IOR behavior changes. The formula is:
   $$ n_{\text{inversion}}(z) = 0.8^z $$
   This creates a distinct gradient profile compared to the normal lapse [S1, unofficial].

3. **Combine Layers:**
   The final IOR profile is the minimum of the two layers at each height $z$:
   $$ n_{\text{combined}}(z) = \min(n_{\text{normal}}(z), n_{\text{inversion}}(z)) $$
   This combination ensures the resulting field respects the constraints of both atmospheric models [S1, unofficial].

4. **Ground Boundary Condition:**
   For any position where $z < 0$ (below ground level), the IOR is clamped to 1.0:
   $$ n(x, y, z) = \begin{cases} 1.0 & \text{if } z < 0 \\ n_{\text{combined}}(z) & \text{if } z \ge 0 \end{cases} $$
   This prevents unphysical refraction values in the ground region [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ior_field(z_grid):
    """
    Computes the procedural index of refraction distribution.
    
    Args:
        z_grid (ndarray): 3D array of vertical coordinates.
        
    Returns:
        ndarray: 3D array of index of refraction values.
    """
    # Step 1: Normal lapse layer
    # n = 0.95 * 2^z
    normal_layer = 0.95 * np.power(2, z_grid)
    
    # Step 2: Inversion layer
    # n = 0.8^z
    inversion_layer = np.power(0.8, z_grid)
    
    # Step 3: Combine layers using minimum
    combined = np.minimum(normal_layer, inversion_layer)
    
    # Step 4: Set IOR to 1.0 for z < 0
    # Create a mask for positions below ground
    below_ground = z_grid < 0
    
    # Apply the boundary condition
    ior_field = np.where(below_ground, 1.0, combined)
    
    return ior_field
```

**Parameters:**
- Base coefficients: 0.95 for normal lapse, 0.8 for inversion [S1, unofficial].
- Exponent base: 2 for normal lapse, 0.8 for inversion [S1, unofficial].
- Ground threshold: $z=0$ [S1, unofficial].

**Edge Cases:**
- **Precision:** Exponential functions can lead to very small or large numbers if $z$ is not normalized. Ensure $z$ is within a reasonable range to avoid floating-point underflow/overflow [S1, unofficial].
- **Boundaries:** The transition at $z=0$ is a hard clamp, which may cause discontinuities in the gradient of the IOR field if not handled carefully during ray marching [S1, unofficial].

## Subtopics

### Extinction and Refraction Reconstruction
This variant uses a random process to generate refraction indices restricted to the inside of a glass sphere, implicitly defining volume shape via varying refractive index [S2]. It employs per-pixel weights to concentrate importance on main parts of user images, leading to less constrained boundaries and more variable shapes [S2]. The volume does not scatter or emit light ($L_m = 0$) [S2]. A key pitfall is that intermediate views appear randomly colored during optimization due to refraction [S2]. This method is used to demonstrate complex shape construction and compatibility with bent rays [S2].

## Operating characteristics
- **Time Complexity:** Depends on the resolution of the $z$-grid and the number of rays traced. The IOR field computation is $O(N)$ where $N$ is the number of voxels.
- **Memory Complexity:** $O(N)$ for storing the 3D IOR field.
- **Strengths:** Efficiently simulates mirage-like effects without explicit particle systems [S1, unofficial]. Compatible with continuous refraction models [S2].
- **Limitations:** The procedural nature limits the complexity of the IOR distribution compared to learned or scanned data [S2]. Random color artifacts can occur in intermediate views during optimization [S2].
- **When to use:** When simulating atmospheric mirages or stylized refractive volumes with simple geometric constraints [S1, unofficial][S2].
- **When not to use:** When precise control over complex, non-procedural IOR distributions is required [S2].

## Background
The method originates from techniques for simulating mirages by creating a non-uniform distribution of the index of refraction in air, where IOR decreases with increasing temperature [S1, unofficial]. It has been adapted for volume stylization, where refraction indices are generated via random processes to match target images, demonstrating the expressiveness of refractive volumes in computer graphics [S2].

## Sources
[S1] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S2] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
