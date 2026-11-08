# Helicity Visualization
Also called: Helicity Volume Rendering

## Overview
Helicity Visualization maps the scalar helicity field to opacity and color to identify vortices in fluid flow data [S1, unofficial]. It sets itself apart from sibling methods like [Stream Lines](4796-stream-lines.md) by rendering the volumetric intensity of rotational structures rather than tracing pathlines, and from [Vector Topology Analysis](7694-vector-topology-analysis.md) by providing a continuous visual representation of vortex strength and direction rather than discrete critical points.

## How it works
**Inputs and Outputs:**
- Input: A 3D velocity vector field $\mathbf{u}(\mathbf{x}) = (u_x, u_y, u_z)$ defined on a regular grid.
- Output: A rendered image where voxel color and opacity represent helicity magnitude and sign.

**Data Structures:**
- The velocity field is stored as three scalar components per grid point.
- The helicity field is a derived scalar field $H(\mathbf{x})$.

**Steps:**
1.  **Calculate Vorticity:** Compute the vorticity vector $\boldsymbol{\omega} = \nabla \times \mathbf{u}$ at each grid point. This typically involves central finite differences for the spatial derivatives of the velocity components.
2.  **Calculate Helicity:** Compute the scalar helicity $H$ as the dot product of the velocity and vorticity vectors:
    $$ H(\mathbf{x}) = \mathbf{u}(\mathbf{x}) \cdot \boldsymbol{\omega}(\mathbf{x}) $$
    The sign of $H$ indicates the direction of rotation of the vortex relative to the flow direction [S1, unofficial].
3.  **Map to Opacity:** Map the magnitude $|H|$ to opacity $\alpha$. Large magnitudes of helicity are mapped to high opacity, while small magnitudes are mapped to near transparency. This makes the bulk of irrelevant free stream flow invisible, highlighting only regions with strong rotational characteristics [S1, unofficial].
4.  **Map to Color:** Map the signed value of $H$ to color. The mapping uses a diverging color scheme:
    - Extreme negative values ($H \ll 0$) are mapped to cyan.
    - Extreme positive values ($H \gg 0$) are mapped to magenta.
    - Values near zero ($H \approx 0$) are mapped to yellow [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_helicity_visualization(velocity_field):
    """
    velocity_field: 3D array of shape (Nx, Ny, Nz, 3) containing u, v, w components.
    Returns: 3D array of shape (Nx, Ny, Nz, 4) with RGBA values.
    """
    # 1. Calculate Vorticity (curl of velocity)
    # omega = curl(u)
    # Using central differences for interior points
    omega_x = (velocity_field[:, 1:, :, 2] - velocity_field[:, :-1, :, 2]) / dy - \
              (velocity_field[:, :, 1:, 1] - velocity_field[:, :, :-1, 1]) / dz
    omega_y = (velocity_field[:, :, 1:, 0] - velocity_field[:, :, :-1, 0]) / dz - \
              (velocity_field[1:, :, :, 2] - velocity_field[:-1, :, :, 2]) / dx
    omega_z = (velocity_field[1:, :, :, 1] - velocity_field[:-1, :, :, 1]) / dy - \
              (velocity_field[:, 1:, :, 0] - velocity_field[:, :-1, :, 0]) / dx
    
    # Pad omega to match velocity field dimensions (simple zero-padding for boundaries)
    omega = np.zeros_like(velocity_field)
    omega[:-1, :-1, :-1, 0] = omega_x
    omega[:-1, :-1, :-1, 1] = omega_y
    omega[:-1, :-1, :-1, 2] = omega_z

    # 2. Calculate Helicity
    # H = u . omega
    helicity = np.sum(velocity_field * omega, axis=3)

    # 3. Map to Opacity based on magnitude
    # Large magnitude -> high opacity, small magnitude -> near transparency
    # Assuming a linear or threshold-based mapping for demonstration
    max_helicity = np.max(np.abs(helicity))
    opacity = np.clip(np.abs(helicity) / max_helicity, 0.0, 1.0)
    
    # 4. Map to Color based on sign
    # Negative -> Cyan, Positive -> Magenta, Near Zero -> Yellow
    color = np.zeros((helicity.shape[0], helicity.shape[1], helicity.shape[2], 3))
    
    # Define colors (RGB)
    cyan = np.array([0.0, 1.0, 1.0])
    magenta = np.array([1.0, 0.0, 1.0])
    yellow = np.array([1.0, 1.0, 0.0])
    
    # Mask for negative values
    neg_mask = helicity < -1e-6
    # Mask for positive values
    pos_mask = helicity > 1e-6
    # Mask for near zero
    zero_mask = ~neg_mask & ~pos_mask
    
    color[neg_mask] = cyan
    color[pos_mask] = magenta
    color[zero_mask] = yellow
    
    # Combine into RGBA
    rgba = np.zeros((helicity.shape[0], helicity.shape[1], helicity.shape[2], 4))
    rgba[:, :, :, :3] = color
    rgba[:, :, :, 3] = opacity
    
    return rgba
```

**Parameters:**
- Opacity mapping function: Not specified in the sources, but described as "large magnitude is opaque... small magnitude is nearly transparent" [S1, unofficial].
- Color thresholds: "Extreme" negative/positive and "near zero" are qualitative. Implementation requires defining thresholds for these categories.

**Edge Cases:**
- Boundary conditions for vorticity calculation are not specified in the sources [S1, unofficial].
- Flow reversal areas (negative x-component velocity) are noted as interesting but not explicitly handled differently in the helicity mapping [S1, unofficial].

## Subtopics

## Operating characteristics
- **Strengths:** Effectively isolates vortices from the background flow by suppressing regions with low helicity magnitude. The color coding distinguishes the direction of rotation (positive vs. negative helicity) [S1, unofficial].
- **Limitations:** Requires calculation of derivatives (vorticity), which can amplify noise in the velocity field. The qualitative nature of "extreme" and "near zero" color mapping may require tuning for specific datasets.
- **Performance:** Depends on the resolution of the velocity field and the efficiency of the curl calculation. Volume rendering itself is computationally intensive but standard for this domain.

## Background
Helicity Visualization was introduced in the context of Volume Rendering for Computational Fluid Dynamics (CFD) to visualize vortices in flow fields [S1, unofficial]. It was used to analyze flow around a fin, identifying vortices starting near the base of the leading edge. The method leverages the physical property that helicity is large near vortices [S1, unofficial].

## Sources
[S1] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
