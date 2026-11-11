# Helicity Visualization

Also called: Helicity Volume Rendering

## Overview
Helicity Visualization maps the scalar helicity field to opacity and color to identify vortices in fluid flows [S1, unofficial]. It sets itself apart from standard velocity visualization by explicitly removing the uniform background flow through opacity mapping, thereby highlighting regions of interaction such as wakes or boundary layers which would otherwise be obscured by the high opacity of the free stream [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A vector velocity field $\mathbf{u}(\mathbf{x})$ defined on a 3D grid.
*   **Output:** A volume-rendered image where voxel color and opacity are determined by the computed helicity.

**Data Structures:**
*   **Velocity Field:** A 3D array of 3-component vectors $\mathbf{u} = (u_x, u_y, u_z)$.
*   **Vorticity Field:** A 3D array of 3-component vectors $\boldsymbol{\omega} = \nabla \times \mathbf{u}$.
*   **Helicity Field:** A 3D array of scalar values $H$.

**Steps:**
1.  **Compute Vorticity:** Calculate the curl of the velocity field at each grid point $\mathbf{x}$:
    $$ \boldsymbol{\omega}(\mathbf{x}) = \nabla \times \mathbf{u}(\mathbf{x}) $$
    [G] In discrete grids, this is typically done using central finite differences.
2.  **Compute Helicity:** Calculate the dot product of the velocity and vorticity vectors at each point:
    $$ H(\mathbf{x}) = \mathbf{u}(\mathbf{x}) \cdot \boldsymbol{\omega}(\mathbf{x}) $$
    [S1, unofficial] Helicity is a scalar value with a large magnitude near vortices. Its sign indicates the direction of rotation [S1, unofficial].
3.  **Map Opacity:** Map the magnitude of helicity $|H|$ to opacity $\alpha$:
    *   Large magnitude $|H| \rightarrow$ High opacity (opaque).
    *   Small magnitude $|H| \rightarrow$ Near transparency.
    [S1, unofficial] The specific functional form (e.g., linear, thresholded) is not detailed, but the principle is magnitude-based visibility [S1, unofficial].
4.  **Map Color:** Map the signed value of helicity $H$ to color $C$:
    *   Extreme negative values ($H \ll 0$) $\rightarrow$ Cyan.
    *   Extreme positive values ($H \gg 0$) $\rightarrow$ Magenta.
    *   Values near zero ($H \approx 0$) $\rightarrow$ Yellow.
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_helicity_visualization(velocity_field):
    """
    velocity_field: 3D numpy array of shape (Nx, Ny, Nz, 3)
    Returns: 4D numpy array of shape (Nx, Ny, Nz, 4) with RGBA values
    """
    # 1. Compute Vorticity (curl of velocity)
    # Using central differences for interior points
    ux, uy, uz = velocity_field[..., 0], velocity_field[..., 1], velocity_field[..., 2]
    
    # d/dx, d/dy, d/dz approximations
    # Note: Boundary handling is not specified in sources [S1, unofficial]
    dw_z_dy = np.gradient(uz, axis=1) - np.gradient(uy, axis=2)
    dw_x_dz = np.gradient(ux, axis=2) - np.gradient(uz, axis=0)
    dv_x_dy = np.gradient(uy, axis=0) - np.gradient(ux, axis=1)
    
    vorticity_x = dw_x_dz
    vorticity_y = dw_z_dy
    vorticity_z = dv_x_dy
    
    # 2. Compute Helicity (dot product)
    helicity = ux * vorticity_x + uy * vorticity_y + uz * vorticity_z
    
    # 3. Map Opacity based on magnitude
    # Not given in the sources: specific transfer function for opacity.
    # Assuming a simple linear mapping from max magnitude for demonstration.
    max_helicity = np.max(np.abs(helicity))
    opacity = np.clip(np.abs(helicity) / max_helicity, 0, 1)
    
    # 4. Map Color based on sign and magnitude
    # Define color mapping:
    # Extreme negative -> Cyan (0, 1, 1)
    # Extreme positive -> Magenta (1, 0, 1)
    # Near zero -> Yellow (1, 1, 0)
    
    # Normalize helicity to range [-1, 1] for color interpolation
    # Not given in the sources: how "extreme" is defined.
    # Using min/max of the field as bounds.
    h_min = np.min(helicity)
    h_max = np.max(helicity)
    
    # Avoid division by zero if h_min == h_max
    if h_max == h_min:
        normalized_h = np.zeros_like(helicity)
    else:
        normalized_h = 2.0 * (helicity - h_min) / (h_max - h_min) - 1.0
        
    # Interpolate color
    # Cyan (0,1,1) at -1, Yellow (1,1,0) at 0, Magenta (1,0,1) at 1
    # This is a piecewise linear interpolation
    color = np.zeros((*helicity.shape, 3))
    
    # Negative part: Cyan to Yellow
    neg_mask = normalized_h < 0
    t_neg = -normalized_h[neg_mask] # t in [0, 1]
    color[neg_mask] = (1 - t_neg) * np.array([0, 1, 1]) + t_neg * np.array([1, 1, 0])
    
    # Positive part: Yellow to Magenta
    pos_mask = normalized_h > 0
    t_pos = normalized_h[pos_mask] # t in [0, 1]
    color[pos_mask] = (1 - t_pos) * np.array([1, 1, 0]) + t_pos * np.array([1, 0, 1])
    
    # Zero values remain yellow (handled by neither mask if strict inequality, 
    # but typically 0 is included in one or both. Let's ensure 0 is yellow)
    zero_mask = normalized_h == 0
    color[zero_mask] = np.array([1, 1, 0])
    
    # Combine into RGBA
    rgba = np.zeros((*helicity.shape, 4))
    rgba[..., 0:3] = color
    rgba[..., 3] = opacity
    
    return rgba
```

## Subtopics

## Operating characteristics
*   **Strengths:** Effectively isolates vortices by suppressing the background flow [S1, unofficial]. The sign of helicity provides additional information about the direction of rotation (cyan vs. magenta) [S1, unofficial].
*   **Limitations:** Requires accurate computation of the curl of the velocity field, which can be sensitive to noise [G]. The method relies on the assumption that high helicity magnitude correlates with vortices [S1, unofficial].
*   **Performance:** Computationally more expensive than simple velocity magnitude visualization due to the need for gradient calculation (vorticity) and dot product [S1, unofficial].

## Background
Helicity Visualization was presented in the context of Computational Fluid Dynamics (CFD) volume rendering [S1, unofficial]. It was used to visualize flow around a fin, specifically to identify vortices and shock structures [S1, unofficial]. The technique was described in a NASA technical report from 1991 [S1, unofficial].

## Sources
[S1] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
