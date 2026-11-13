# Gradient Selection
Also called: Gradient Mode Selection, Gradient estimation via finite differences

## Overview
Gradient Selection is a runtime dispatch mechanism that chooses between specific finite-difference gradient estimation schemes (typically Central Difference or Sobel) based on a configuration uniform or boundary proximity [S1, unofficial]. It solves the problem of selecting the appropriate normal estimation strategy for volume shading, balancing computational cost against noise robustness [S2, unofficial]. It is distinct from fixed-stencil methods by introducing conditional logic to switch between isotropic central differences and edge-aware Sobel operators [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `textureCoord` (normalized coordinates in $[0,1]^3$), `gradientMode` (enum or integer flag).
- Output: Normalized gradient vector $\mathbf{g} = (g_x, g_y, g_z)$.

**Data Structures:**
- `gradientMode`: Integer constant. Common values: `GRADIENT_CENTRAL` (default), `GRADIENT_SOBEL`.

**Steps:**
1. Evaluate the scalar field $V$ at the current position using trilinear interpolation to obtain $V(\mathbf{p})$.
2. Check the `gradientMode` flag.
3. If `gradientMode == GRADIENT_SOBEL`:
   - Compute partial derivatives using the Sobel operator, which applies weighted averages to neighbors to reduce noise sensitivity [S1, unofficial].
   - Return normalized Sobel gradient.
4. Else (default to Central Difference):
   - Compute partial derivatives using central differences [S1, unofficial].
   - Return normalized central gradient.

**Formulas:**
For Central Difference with voxel spacing $h=1$:
$$ g_x = \frac{V(x+1, y, z) - V(x-1, y, z)}{2} $$
$$ g_y = \frac{V(x, y+1, z) - V(x, y-1, z)}{2} $$
$$ g_z = \frac{V(x, y, z+1) - V(x, y, z-1)}{2} $$

For Sobel, the x-direction derivative uses a $3 \times 1 \times 1$ kernel in x and $1 \times 3 \times 1$ in y/z (conceptually a 3x3x1 or 3x1x3 window depending on axis), typically:
$$ g_x = \frac{1}{4} [ (V_{x+1,y+1,z} + 2V_{x+1,y,z} + V_{x+1,y-1,z}) - (V_{x-1,y+1,z} + 2V_{x-1,y,z} + V_{x-1,y-1,z}) ] $$
(Exact Sobel weights vary by implementation; the key distinction is the weighted averaging [S1, unofficial]).

**Pitfalls:**
- Noisy data may produce inaccurate normals with simple central differences, necessitating the Sobel mode [S2, unofficial].
- Boundary handling must be consistent with the chosen mode (Sobel requires more neighbors than central differences).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_selection(texture_coord, gradient_mode, volume_texture):
    """
    Dispatches between Central Difference and Sobel gradient estimation.
    
    Args:
        texture_coord: vec3, normalized coordinates [0,1]
        gradient_mode: int, 0 for Central, 1 for Sobel
        volume_texture: function or object providing trilinear sampling V(x,y,z)
        
    Returns:
        vec3, normalized gradient vector
    """
    # Sample the scalar field at the center
    v_center = volume_texture.sample(texture_coord)
    
    if gradient_mode == 1: # GRADIENT_SOBEL
        # Sobel estimation handles noise better but is more expensive
        # Implementation details for Sobel are in [S1, unofficial]
        return gradient_sobel(texture_coord, volume_texture)
    else: # Default to Central Difference
        # Central difference estimation
        return gradient_central(texture_coord, volume_texture)

def gradient_central(coord, tex):
    """
    Computes gradient using central differences.
    Assumes isotropic spacing h=1 in voxel indices.
    """
    # Convert normalized coord to voxel index for neighbor lookup
    # Note: Actual implementation requires handling boundary conditions
    # [G] Assuming tex.sample handles boundary clamping or wrapping
    dx = (tex.sample(coord + vec3(1, 0, 0)) - tex.sample(coord - vec3(1, 0, 0))) / 2.0
    dy = (tex.sample(coord + vec3(0, 1, 0)) - tex.sample(coord - vec3(0, 1, 0))) / 2.0
    dz = (tex.sample(coord + vec3(0, 0, 1)) - tex.sample(coord - vec3(0, 0, 1))) / 2.0
    
    grad = vec3(dx, dy, dz)
    return normalize(grad)

def gradient_sobel(coord, tex):
    """
    Computes gradient using Sobel operator.
    Specific weights and neighbor access pattern defined in [S1, unofficial].
    """
    # [G] Standard Sobel 3x3x1 kernels applied per axis.
    # This is a placeholder for the specific Sobel implementation referenced in [S1, unofficial].
    # The method selects this mode to mitigate noise [S2, unofficial].
    pass # Implementation details not fully specified in sources; see [S1, unofficial]
```

## Subtopics

### [Cubic Shading](9681-cubic-shading.md)
Uses all 26 neighbors in a 3x3x3 window for smoother normals; chosen when high-quality surface approximation is needed over speed.

### [Gradient Selection — Central/Forward/Backward](12132-gradient-selection-central-forward-backward.md)
Selects schemes based on proximity to boundaries to avoid out-of-bounds access; distinct from Sobel by using one-sided differences at edges.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, but Sobel requires more texture fetches (up to 18 neighbors) than Central Difference (6 neighbors).
- **Memory:** Minimal overhead; requires only the volume texture and a mode flag.
- **Strengths:** Flexible; allows runtime tuning for noise vs. performance. Central difference is fast and isotropic. Sobel reduces noise artifacts.
- **Limitations:** Sobel is computationally heavier. Boundary handling complexity increases with Sobel.
- **When to use:** Use Central Difference for smooth, low-noise data or performance-critical applications. Use Sobel for noisy data where edge preservation is critical [S2, unofficial].

## Background
Gradient selection is a common pattern in GPU-based volume rendering shaders, particularly in WebGL and OpenGL implementations where hardware acceleration is leveraged [S1, unofficial]. It emerged as a practical solution to the trade-off between gradient accuracy and noise sensitivity in real-time rendering [S2, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
