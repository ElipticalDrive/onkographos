# Gradient Selection
Also called: Central differences, Finite Difference Stencils, Gradient Mode Selection, Gradient estimation via finite differences

## Overview
Gradient Selection is a runtime strategy for volume rendering that chooses between different finite difference gradient estimation schemes (typically Central Difference or Sobel) based on a configuration uniform or boundary proximity [S1, S2, unofficial]. It solves the problem of selecting the appropriate normal approximation for shading, balancing computational cost against accuracy and noise robustness [S2, unofficial]. It is distinct from fixed-stencil methods by introducing conditional logic to switch between estimation modes, often prioritizing Central Differences for interior regions and Sobel or one-sided differences for boundary handling or specific visual effects [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar volume texture $V$, a sampling coordinate $\mathbf{c}$ (or voxel index), and a `gradientMode` selector (e.g., `GRADIENT_CENTRAL`, `GRADIENT_SOBEL`) [S1, unofficial].
- Output: A normalized gradient vector $\mathbf{g} = \frac{\nabla V}{\|\nabla V\|}$ used as a surface normal substitute in shading models [S2, unofficial].

**Data Structures:**
- The volume is stored as a 3D texture or array.
- The `gradientMode` is a uniform variable or integer flag [S1, unofficial].

**Steps:**
1. **Mode Check:** Evaluate the `gradientMode` parameter [S1, unofficial].
2. **Branch Execution:**
   - If `gradientMode == GRADIENT_SOBEL`: Compute the gradient using the Sobel operator [S1, unofficial].
   - Else (defaulting to Central Difference): Compute the gradient using central finite differences [S1, S2, unofficial].
3. **Normalization:** Normalize the resulting gradient vector $\mathbf{G} = (G_x, G_y, G_z)$ to unit length [S2, unofficial].

**Formulas:**
- **Central Difference:** For a voxel at $(x, y, z)$ with spacing $h$:
  $$ G_x = \frac{V(x+1, y, z) - V(x-1, y, z)}{2h} $$
  $$ G_y = \frac{V(x, y+1, z) - V(x, y-1, z)}{2h} $$
  $$ G_z = \frac{V(x, y, z+1) - V(x, y, z-1)}{2h} $$
  [S2, unofficial]
- **Sobel Difference:** Uses weighted sums of neighbors in a 3x3x3 kernel (specific weights depend on implementation, but generally emphasizes center and immediate neighbors with specific axial weights) [S1, unofficial].
- **Normalization:**
  $$ \mathbf{g} = \frac{(G_x, G_y, G_z)}{\sqrt{G_x^2 + G_y^2 + G_z^2}} $$
  [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_selection(volume, coord, gradient_mode, voxel_size):
    """
    Selects and computes gradient based on mode.
    volume: 3D array or texture.
    coord: (x, y, z) indices.
    gradient_mode: 'SOBEL' or 'CENTRAL'.
    voxel_size: float h.
    Returns: Normalized gradient vector (gx, gy, gz).
    """
    if gradient_mode == 'SOBEL':
        gx, gy, gz = gradient_sobel(volume, coord, voxel_size)
    else:
        gx, gy, gz = gradient_central(volume, coord, voxel_size)
    
    magnitude = sqrt(gx**2 + gy**2 + gz**2)
    if magnitude == 0:
        return (0.0, 0.0, 0.0)
    
    return (gx / magnitude, gy / magnitude, gz / magnitude)

def gradient_central(volume, coord, voxel_size):
    x, y, z = coord
    # Boundary handling: clamp to valid range [G]
    def get_val(dx, dy, dz):
        vx = clamp(x + dx, 0, volume.shape[0] - 1)
        vy = clamp(y + dy, 0, volume.shape[1] - 1)
        vz = clamp(z + dz, 0, volume.shape[2] - 1)
        return volume[vx, vy, vz]
    
    gx = (get_val(1, 0, 0) - get_val(-1, 0, 0)) / (2 * voxel_size)
    gy = (get_val(0, 1, 0) - get_val(0, -1, 0)) / (2 * voxel_size)
    gz = (get_val(0, 0, 1) - get_val(0, 0, -1)) / (2 * voxel_size)
    return gx, gy, gz

def gradient_sobel(volume, coord, voxel_size):
    # Sobel implementation details are not explicitly defined in sources [S1, unofficial]
    # Standard 3D Sobel involves weighted neighbors. 
    # Placeholder for specific Sobel kernel application [G]
    pass
```

**Parameters:**
- `gradientMode`: Controls the selection. Typical values: `GRADIENT_CENTRAL`, `GRADIENT_SOBEL` [S1, unofficial].

**Edge Cases:**
- **Noisy Data:** Simple central differences may produce inaccurate normals in noisy volumes; Sobel or more complex schemes are preferred [S2, unofficial].
- **Boundaries:** Central differences require neighbors on both sides. Clamping or one-sided differences (as noted in parent/children) are needed at boundaries to avoid out-of-bounds access [S1, S2, unofficial].

## Subtopics

### [Cubic Shading](9681-cubic-shading.md)
Uses all 26 neighbors in a 3x3x3 window for smoother normals, distinct from standard 6-neighbor gradients.

### [Gradient Selection — Central/Forward/Backward](12132-gradient-selection-central-forward-backward.md)
Explicitly handles boundaries by switching to one-sided differences based on proximity to edges, avoiding out-of-bounds errors.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a fixed number of texture lookups and arithmetic operations [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory [S1, S2, unofficial].
- **Strengths:** Simple to implement; Central Difference is accurate for smooth data; Sobel offers noise robustness [S2, unofficial].
- **Limitations:** Central Difference is sensitive to noise; Sobel is more expensive than Central Difference [S2, unofficial].
- **When to Use:** Use Central Difference for smooth, high-quality data; use Sobel or one-sided variants for noisy data or boundary regions [S1, S2, unofficial].

## Background
Gradient estimation via finite differences is a fundamental technique in volume rendering for computing surface normals for shading [S2, unofficial]. The specific selection mechanism between Central and Sobel modes is documented in community implementations [S1, unofficial]. The theoretical basis for finite difference gradients is established in standard graphics literature [S2, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
