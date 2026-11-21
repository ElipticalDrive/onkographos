# Helicity Visualization

Also called: Helicity Calculation

## Overview
Helicity Visualization maps the scalar helicity field to color and opacity to identify vortices and their rotation direction in fluid flows [S1, unofficial][S2]. It sets itself apart from siblings like [Metric Scalar Field Rendering from Vector Fields](14065-metric-scalar-field-rendering-from-vector-fields.md) by explicitly utilizing the directional relationship between velocity and vorticity (via the dot product) rather than just magnitude or eigenvalues, allowing for the distinction of vortex chirality through signed color mapping [S1, unofficial].

## How it works
**Inputs:**
- Velocity field $\mathbf{F}(\mathbf{x}) = (u, v, w)$ defined on a grid.
- Vorticity field $\boldsymbol{\omega}(\mathbf{x}) = \nabla \times \mathbf{F}$.

**Outputs:**
- Color $C(\mathbf{x})$ and Opacity $\alpha(\mathbf{x})$ for each sample point.

**Data Structures:**
- The helicity scalar field $h(\mathbf{x})$ is computed per voxel.
- Transfer function $T(h)$ maps scalar value to $(R, G, B, \alpha)$.

**Steps:**
1. **Compute Vorticity:** Calculate the curl of the velocity field at each grid point $\mathbf{x}$:
   $$ \boldsymbol{\omega}(\mathbf{x}) = \nabla \times \mathbf{F}(\mathbf{x}) $$
   [S2]

2. **Compute Helicity:** Calculate the dot product of the velocity and vorticity vectors:
   $$ h(\mathbf{x}) = \boldsymbol{\omega}(\mathbf{x}) \cdot \mathbf{F}(\mathbf{x}) $$
   [S1, unofficial][S2]
   This scalar represents the curl in the direction of the velocity [S2].

3. **Map Opacity:** Map the magnitude of helicity $|h(\mathbf{x})|$ to opacity $\alpha$:
   - Large magnitude $\rightarrow$ High opacity (opaque).
   - Small magnitude $\rightarrow$ Near transparency.
   [S1, unofficial]

4. **Map Color:** Map the signed value of helicity $h(\mathbf{x})$ to color:
   - Extreme negative values $\rightarrow$ Cyan.
   - Extreme positive values $\rightarrow$ Magenta.
   - Values near zero $\rightarrow$ Yellow.
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_helicity_visualization(velocity_field):
    """
    velocity_field: 3D array of shape (Nx, Ny, Nz, 3) containing (u, v, w)
    Returns: 4D array of shape (Nx, Ny, Nz, 4) containing (R, G, B, alpha)
    """
    # 1. Compute Vorticity (Curl of Velocity)
    # Using central differences for gradient approximation
    # omega = (dw/dy - dv/dz, du/dz - dw/dx, dv/dx - du/dy)
    vorticity = compute_curl(velocity_field)
    
    # 2. Compute Helicity (Dot Product)
    # h = omega . F
    # Element-wise dot product of velocity and vorticity arrays
    helicity = sum(v * w for v, w in zip(velocity_field, vorticity))
    
    # 3. Initialize Output Buffer
    output = np.zeros_like(helicity, dtype=float)
    output = np.append(output, np.zeros((helicity.shape[0], helicity.shape[1], helicity.shape[2], 1)), axis=3)
    
    # 4. Apply Transfer Function
    for x in range(helicity.shape[0]):
        for y in range(helicity.shape[1]):
            for z in range(helicity.shape[2]):
                h_val = helicity[x, y, z]
                mag_h = abs(h_val)
                
                # Map Opacity based on magnitude
                # Large magnitude -> opaque, small -> transparent
                # Assuming a linear or sigmoidal mapping not explicitly defined in sources
                # [G] Standard practice: alpha = sigmoid(mag_h * scale) or normalized linear
                alpha = normalize_magnitude_to_opacity(mag_h)
                
                # Map Color based on sign
                if h_val > threshold_positive:
                    color = [1.0, 0.0, 1.0] # Magenta
                elif h_val < threshold_negative:
                    color = [0.0, 1.0, 1.0] # Cyan
                else:
                    color = [1.0, 1.0, 0.0] # Yellow
                
                output[x, y, z, 0] = color[0]
                output[x, y, z, 1] = color[1]
                output[x, y, z, 2] = color[2]
                output[x, y, z, 3] = alpha
                
    return output

def compute_curl(field):
    # Placeholder for standard finite difference curl implementation
    # [G] Implementation detail: Central differences for interior points, 
    # forward/backward for boundaries.
    pass

def normalize_magnitude_to_opacity(mag):
    # [G] Specific mapping function not provided in sources.
    # Typically involves normalization against max helicity in the domain.
    pass
```

**Parameters:**
- Thresholds for "extreme" positive/negative and "near zero" are not explicitly defined in the sources [S1, unofficial][S2].
- Scaling for opacity mapping is not specified [S1, unofficial].

**Edge Cases:**
- **Zero Velocity:** If $\mathbf{F} = 0$, helicity is 0, resulting in yellow color and low opacity [S1, unofficial].
- **Flow Reversal:** Negative x-component values in velocity (flow reversal) are of particular interest as they indicate separation, which may result in negative helicity values mapped to cyan [S1, unofficial].

## Subtopics

## Operating characteristics
- **Strengths:** Effectively visualizes vortices and distinguishes their rotation direction (chirality) using color [S1, unofficial]. It makes features like "lambda" shocks and vortex cores visible by mapping magnitude to opacity [S1, unofficial].
- **Limitations:** Requires accurate computation of both velocity and its curl (vorticity), which can be noisy in discrete data [S2]. The specific thresholds for color mapping are not standardized in the source material [S1, unofficial].
- **Performance:** Depends on the cost of computing the curl and the dot product per voxel. No specific hardware performance metrics are provided in the sources.

## Background
- **Origin:** The method is described in early CFD volume rendering work [S1, unofficial].
- **History:** Helicity is defined as the scalar function representing the curl in the direction of the velocity [S2]. It was used to visualize vortices generated by fins in flow fields, identifying features like the "lambda" shock [S1, unofficial].
- **Usage:** Used in Computational Fluid Dynamics (CFD) to isolate flow disturbances and identify vortices [S1, unofficial][S2].

## Sources
[S1] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S2] Multi-dimensional Transfer Functions for Interactive 3D Flow Visualization, https://web.cs.ucdavis.edu/~hamann/ParkBudgeLinsenHamannJoy2004.pdf, fetched 2026-10-27
