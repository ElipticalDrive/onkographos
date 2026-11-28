# Noise Position Calculation

Also called: Noise Space Position Calculation, Procedural Noise Positioning

## Overview
Calculates the 3D or 4D position in noise space for each triangle fragment to apply procedural noise and turbulence functions [S1, unofficial]. It maps spatial coordinates to a noise field supporting animation and instance uniqueness, distinct from standard 2D texture mapping [S1, unofficial]. It prepares the input for turbulence evaluation and subsequent color lookup [S1, unofficial].

## How it works
**Inputs:**
- `tex`: Unit volume texture coordinates (3D vector).
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
1. **Scale Spatial Coordinates:** Multiply the unit texture coordinates by `noiseScale` to set the base spatial frequency [S1, unofficial].
   $$ \mathbf{p}_{\text{scaled}} = \mathbf{tex} \times \text{noiseScale} $$
2. **Construct 4D Position:** Initialize the noise position with the scaled coordinates and set the fourth dimension to `time` to enable 4D noise evaluation [S1, unofficial].
   $$ \mathbf{p}_{4d} = (\mathbf{p}_{\text{scaled}}.x, \mathbf{p}_{\text{scaled}}.y, \mathbf{p}_{\text{scaled}}.z, \text{time}) $$
3. **Apply Vertical Flow Offset:** Offset the vertical axis ($y$) by $-\text{time}$ to create an upward flow effect [S1, unofficial].
   $$ \mathbf{p}_{\text{flow}} = (\mathbf{p}_{4d}.x, \mathbf{p}_{4d}.y - \text{time}, \mathbf{p}_{4d}.z, \mathbf{p}_{4d}.w) $$
4. **Add Instance Offset:** Add the `instanceOffset` to the spatial components ($x, y, z$) to ensure each fire instance has a unique noise pattern [S1, unofficial].
   $$ \mathbf{p}_{\text{final}} = (\mathbf{p}_{\text{flow}}.x + \text{instanceOffset}.x, \mathbf{p}_{\text{flow}}.y + \text{instanceOffset}.y, \mathbf{p}_{\text{flow}}.z + \text{instanceOffset}.z, \mathbf{p}_{\text{flow}}.w) $$
5. **Calculate Turbulence Kernel:** Compute a kernel value $f(\text{tex}_z)$ using the stability factor and the square root of the height ($z$) [S1, unofficial].
   $$ f(\text{tex}_z) = \text{stabilityFactor} \times \sqrt{\text{tex}.z} $$
6. **Apply Turbulence to Position:** Multiply the turbulence value by the kernel and add it to the vertical component of the texture lookup position [S1, unofficial].
   $$ \text{finalTex}.z = \mathbf{p}_{\text{final}}.z + f(\text{tex}_z) $$
   $$ \text{finalTex}.x = \mathbf{p}_{\text{final}}.x $$
   $$ \text{finalTex}.y = \mathbf{p}_{\text{final}}.y $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_noise_position(tex: vec3, noiseScale: float, time: float, instanceOffset: vec3, stabilityFactor: float) -> tuple[vec4, vec3]:
    p_scaled = tex * noiseScale
    p_4d = vec4(p_scaled.x, p_scaled.y, p_scaled.z, time)
    p_flow = vec4(p_4d.x, p_4d.y - time, p_4d.z, p_4d.w)
    p_final = vec4(p_flow.x + instanceOffset.x, p_flow.y + instanceOffset.y, p_flow.z + instanceOffset.z, p_flow.w)
    kernel = stabilityFactor * sqrt(tex.z)
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
Time and memory complexity per fragment are not given in the sources. Strengths, limitations, and usage guidelines are not given in the sources.

## Background
Originated in real-time procedural volumetric fire rendering [S1, unofficial].

## Sources
[S1] Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
