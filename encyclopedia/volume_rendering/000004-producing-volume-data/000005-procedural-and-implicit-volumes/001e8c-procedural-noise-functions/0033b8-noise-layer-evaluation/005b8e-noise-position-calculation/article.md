# Noise Position Calculation

Also called: Noise Space Position Calculation, Procedural Noise Positioning

## Overview
Calculates the 3D or 4D position in noise space for each fragment to apply procedural noise and turbulence functions [S1, unofficial]. It solves the problem of mapping spatial coordinates to a noise field that supports animation (time) and instance uniqueness, distinct from standard 2D texture coordinate mapping [S1, unofficial]. It sets up the input for turbulence and color lookup in procedural volume rendering [S1, unofficial].

## How it works
**Inputs:**
- `tex`: Unit volume texture coordinates (3D vector, typically $[0,1]^3$).
- `noiseScale`: Scalar scaling factor for spatial frequency.
- `time`: Current animation time (scalar).
- `instanceOffset`: 3D constant vector unique to each fire instance.
- `stabilityFactor`: Scalar for turbulence kernel stability.

**Outputs:**
- `noisePos`: 4D position vector in noise space $(x, y, z, t)$.
- `finalTex`: Adjusted 3D texture coordinates for color lookup.

**Data Structures:**
- `vec3`: 3D floating-point vector.
- `vec4`: 4D floating-point vector.

**Steps:**
1. **Scale Spatial Coordinates:** Multiply the unit texture coordinates by `noiseScale` to set the base spatial frequency.
   $$ \mathbf{p}_{scaled} = \mathbf{tex} \times \text{noiseScale} $$
2. **Construct 4D Position:** Initialize the noise position with the scaled coordinates. Set the fourth dimension ($w$) to `time` to enable 4D noise evaluation.
   $$ \mathbf{p}_{4d} = (\mathbf{p}_{scaled}.x, \mathbf{p}_{scaled}.y, \mathbf{p}_{scaled}.z, \text{time}) $$
3. **Apply Vertical Flow Offset:** Offset the vertical axis ($y$) by $-\text{time}$ to create an upward flow effect.
   $$ \mathbf{p}_{flow} = (\mathbf{p}_{4d}.x, \mathbf{p}_{4d}.y - \text{time}, \mathbf{p}_{4d}.z, \mathbf{p}_{4d}.w) $$
4. **Add Instance Offset:** Add the `instanceOffset` to the spatial components ($x, y, z$) to ensure each fire instance has a unique noise pattern.
   $$ \mathbf{p}_{final} = (\mathbf{p}_{flow}.x + \text{instanceOffset}.x, \mathbf{p}_{flow}.y + \text{instanceOffset}.y, \mathbf{p}_{flow}.z + \text{instanceOffset}.z, \mathbf{p}_{flow}.w) $$
5. **Calculate Turbulence Kernel:** Compute a kernel value $f(\text{tex}_z)$ using the stability factor and the square root of the height ($z$).
   $$ f(\text{tex}_z) = \text{stabilityFactor} \times \sqrt{\text{tex}.z} $$
   [G] Note: The source specifies `texz` as the input to the kernel, interpreted as the z-component of the original texture coordinate.
6. **Apply Turbulence to Position:** Multiply the turbulence value by the kernel and add it to the vertical component of the texture lookup position. This step modifies the sampling position based on local noise density.
   $$ \text{finalTex}.z = \mathbf{p}_{final}.z + f(\text{tex}_z) $$
   $$ \text{finalTex}.x = \mathbf{p}_{final}.x $$
   $$ \text{finalTex}.y = \mathbf{p}_{final}.y $$
   [G] The source implies the turbulence is added to the vertical component of the *texture look-up*, not necessarily the noise position itself, to drive the color sampling.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_noise_position(tex: vec3, noiseScale: float, time: float, instanceOffset: vec3, stabilityFactor: float) -> tuple[vec4, vec3]:
    # Step 1: Scale spatial coordinates
    p_scaled = tex * noiseScale
    
    # Step 2: Construct 4D position with time
    p_4d = vec4(p_scaled.x, p_scaled.y, p_scaled.z, time)
    
    # Step 3: Offset vertical axis for upward flow
    p_flow = vec4(p_4d.x, p_4d.y - time, p_4d.z, p_4d.w)
    
    # Step 4: Add instance-specific offset
    p_final = vec4(p_flow.x + instanceOffset.x, 
                   p_flow.y + instanceOffset.y, 
                   p_flow.z + instanceOffset.z, 
                   p_flow.w)
    
    # Step 5: Calculate turbulence kernel
    # f(texz) = stabilityFactor * sqrt(tex.z)
    kernel = stabilityFactor * sqrt(tex.z)
    
    # Step 6: Apply turbulence to vertical texture coordinate
    final_tex = vec3(p_final.x, p_final.y, p_final.z + kernel)
    
    return p_final, final_tex
```

**Parameters:**
- `noiseScale`: Controls spatial frequency. Not given in the sources.
- `stabilityFactor`: Controls turbulence magnitude. Not given in the sources.
- `instanceOffset`: Unique per instance. Not given in the sources.

**Edge Cases:**
- `tex.z < 0`: Square root is undefined. [G] Clamp `tex.z` to $\ge 0$ or handle as invalid.
- `time` large: Potential precision loss in floating-point subtraction. [G] Use modulo or relative time if needed.

## Subtopics

### [Solid Texturing](16946-solid-texturing.md)
Applies procedural noise to surfaces/volumes by evaluating 3D noise at spatial coordinates, avoiding 2D mapping seams. Chosen when continuous material appearance is needed across complex topologies.

## Operating characteristics
- **Time Complexity:** $O(1)$ per fragment.
- **Memory Complexity:** $O(1)$ auxiliary storage.
- **Strengths:** Enables animated, instance-unique procedural volumes with minimal overhead.
- **Limitations:** Requires 4D noise function support. Kernel choice affects visual stability.
- **When to use:** Procedural fire, smoke, or terrain where spatial continuity and animation are required.
- **When not to use:** Static 2D texturing or when instance uniqueness is not needed.

## Background
Originated in real-time procedural volumetric fire rendering [S1, unofficial]. Developed by Fuller, Krishnan, Mahrous, Hamann, and Joy at UC Davis, presented at I3D 2007 [S1, unofficial]. Used in GPU-based fire and smoke simulations.

## Sources
[S1] Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
