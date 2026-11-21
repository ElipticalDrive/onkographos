# Noise Kernel Generation

Also called: GetNoiseKernel

## Overview
Noise Kernel Generation constructs a rotation matrix aligned with the light direction to transform a fixed set of 6 offset vectors into a local coordinate system for cone tracing [S1, unofficial]. It serves as the directional basis for Random Cone Rotation within the Voxel Cone Tracing pipeline [S1, unofficial]. Unlike TraceDiffuse, which samples indirect lighting around the surface normal, this method aligns the cone distribution with the light vector [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `lightDirection`: A normalized 3D vector representing the direction of the light source.

**Outputs:**
- `sunRotMatrix`: A $3 \times 3$ rotation matrix.
- `noise_kernel`: An array of 6 transformed 3D vectors.

**Data Structures:**
The kernel consists of 6 fixed base offsets defined in a local space [S1, unofficial]. These are transformed by the rotation matrix to align with the light direction [S1, unofficial].

Base offsets (`base_offsets[6]`):
1. $(0.1, 0.25, -0.15)$
2. $(0.2, 0.50, 0.20)$
3. $(-0.2, 0.10, -0.10)$
4. $(-0.05, 0.75, 0.05)$
5. $(-0.1, 1.00, 0.00)$
6. $(0.0, 3.00, 0.00)$

**Steps:**
1. **Determine Reference Vector:** Identify the component of `lightDirection` with the largest absolute value. Create a unit vector `maxCompUnitVector` aligned with that axis (the source notes the axis selection example `( |x|, 0, 0 )` for the x-axis case) [S1, unofficial].
2. **Compute Z-axis:** Calculate the cross product of `lightDirection` and `maxCompUnitVector` to get the local Z-axis vector `zcom` [S1, unofficial].
   $$ zcom = \text{cross}(\text{lightDirection}, \text{maxCompUnitVector}) $$
3. **Compute X-axis:** Calculate the cross product of `zcom` and `lightDirection` to get the local X-axis vector `xcom` [S1, unofficial].
   $$ xcom = \text{cross}(zcom, \text{lightDirection}) $$
4. **Construct Rotation Matrix:** Form the $3 \times 3$ matrix `sunRotMatrix` using `xcom`, `lightDirection`, and `zcom` as columns [S1, unofficial].
   $$ \text{sunRotMatrix} = \begin{bmatrix} | & | & | \\ xcom & \text{lightDirection} & zcom \\ | & | & | \end{bmatrix} $$
5. **Transform Kernel Offsets:** For each base offset $b_i$ in `base_offsets`, compute the final kernel vector $k_i$ by multiplying `sunRotMatrix` with $b_i$ [S1, unofficial]. Store $k_i$ in `noise_kernel[i]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_noise_kernel(light_direction):
    # Normalize input
    L = normalize(light_direction)
    
    # Step 1: Determine reference vector aligned with max component
    abs_x, abs_y, abs_z = abs(L.x), abs(L.y), abs(L.z)
    if abs_x >= abs_y and abs_x >= abs_z:
        ref_vec = vec3(1.0, 0.0, 0.0)
    elif abs_y >= abs_x and abs_y >= abs_z:
        ref_vec = vec3(0.0, 1.0, 0.0)
    else:
        ref_vec = vec3(0.0, 0.0, 1.0)
        
    # Step 2: Compute Z-axis
    zcom = cross(L, ref_vec)
    
    # Guard against degenerate case where L is parallel to ref_vec
    # The sources do not specify a fallback; standard practice applies [G]
    if length(zcom) < 1e-6:
        if abs_x < 0.9:
            ref_vec = vec3(1.0, 0.0, 0.0)
        else:
            ref_vec = vec3(0.0, 1.0, 0.0)
        zcom = cross(L, ref_vec)
    zcom = normalize(zcom)
    
    # Step 3: Compute X-axis
    xcom = cross(zcom, L)
    xcom = normalize(xcom)
    
    # Step 4: Construct Rotation Matrix (columns: xcom, L, zcom)
    rot_matrix = mat3(xcom, L, zcom)
    
    # Step 5: Transform Kernel Offsets
    base_offsets = [
        vec3(0.1, 0.25, -0.15),
        vec3(0.2, 0.50, 0.20),
        vec3(-0.2, 0.10, -0.10),
        vec3(-0.05, 0.75, 0.05),
        vec3(-0.1, 1.00, 0.00),
        vec3(0.0, 3.00, 0.00)
    ]
    
    noise_kernel = []
    for offset in base_offsets:
        noise_kernel.append(rot_matrix * offset)
        
    return rot_matrix, noise_kernel
```

**Parameters:**
- No adjustable parameters for the kernel generation itself; the base offsets are hardcoded [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If `lightDirection` is parallel to the chosen reference axis, the cross product for `zcom` becomes zero. The pseudocode includes a guard to select an alternative reference vector [G].
- **Precision:** Not given in the sources.

## Subtopics

### TraceDiffuse
Approximates indirect diffuse lighting by casting multiple cones in a tangent space around the surface normal rather than the light direction [S2, unofficial]. It defines 9 fixed cone directions in a $3 \times 3$ grid on the upper hemisphere ($z=1$) [S2, unofficial]. It calculates an aperture scale `apertureTan2 = 2 * tan(radians(apertureAngle) / 2)` [S2, unofficial]. It generates random tangent vectors `xtan` and `ytan` based on position and time; if `xtan` is nearly parallel to the normal, it uses an orthogonal vector [S2, unofficial]. It constructs a tangent space matrix `tanSpace = mat3(xtan, ytan, normal)` [S2, unofficial]. It loops `CONES` times (default 5) to transform cone directions via `dir = tanSpace * cones[i]`, normalizes `dir`, and calls `ConeTracing(pos, dir, apertureTan2)` to accumulate color [S2, unofficial]. Choose TraceDiffuse when surface-normal-based indirect lighting is required, as opposed to light-direction-based sampling [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$, as it involves a fixed number of vector operations and matrix multiplications [G].
- **Memory Complexity:** $O(1)$, storing a fixed $3 \times 3$ matrix and 6 vectors [G].
- Measured performance, scaling, strengths, limitations, and specific use cases are not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S2] righier/gidemo, https://github.com/righier/gidemo, fetched 2026-10-26 (unofficial)
