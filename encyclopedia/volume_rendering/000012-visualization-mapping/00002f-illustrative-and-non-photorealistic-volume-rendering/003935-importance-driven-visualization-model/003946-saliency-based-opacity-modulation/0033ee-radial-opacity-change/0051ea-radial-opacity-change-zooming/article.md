# Radial Opacity Change (Zooming)

## Overview
Radial Opacity Change (Zooming) is an illustrative volume rendering technique that reveals inner structures by modulating voxel opacity based on the distance from a user-defined focus point during zooming interactions [S1, unofficial][S2, unofficial]. Unlike its parent [Radial Opacity Change](7140-radial-opacity-change.md), which uses angular velocity and distance from the center to create a tunnel effect during rotation, this method shifts the transparency gradient to follow the user's interaction point, allowing detailed inspection of specific regions while maintaining context elsewhere [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: Volume dataset $V$, camera parameters, user interaction point $\vec{P}_{focus}$ (initial point of interaction on the object), and a scaling factor $S(t)$ [S1, unofficial][S2, unofficial].
- **Output**: Rendered image with modulated opacity.

**Data Structures**
- **Volume Data**: Standard 3D grid of scalar values [G].
- **Focus Point**: A 3D vector $\vec{P}_{focus}$ representing the location in world or object space where the user is focusing [S1, unofficial][S2, unofficial].

**Algorithm**
For each voxel $\vec{P}_{voxel}$ along the viewing ray:
1. Calculate the Euclidean distance from the voxel to the rendering area center $\vec{P}_{center}$:
   $$ \rho(\vec{P}_{voxel}, \vec{P}_{center}) = \| \vec{P}_{voxel} - \vec{P}_{center} \| $$ [S1, unofficial][S2, unofficial]
2. Calculate the Euclidean distance from the voxel to the focus point:
   $$ \rho(\vec{P}_{voxel}, \vec{P}_{focus}) = \| \vec{P}_{voxel} - \vec{P}_{focus} \| $$ [S1, unofficial][S2, unofficial]
3. Compute the radial difference:
   $$ \Delta \rho = \max(0, \rho(\vec{P}_{voxel}, \vec{P}_{center}) - \rho(\vec{P}_{voxel}, \vec{P}_{focus})) $$ [S1, unofficial][S2, unofficial]
4. Apply exponential attenuation to determine the effective opacity:
   $$ \alpha_{voxel} = e^{-S(t) \cdot \Delta \rho} $$ [S1, unofficial][S2, unofficial]

**Parameters**
- $\vec{P}_{focus}$: The initial point of interaction on the object. This point defines the center of the "zoom" effect [S1, unofficial][S2, unofficial].
- $S(t)$: A scaling factor that controls the strength of the opacity modulation. It may be constant or vary with time/interaction [S1, unofficial][S2, unofficial]. Typical values are not specified in the sources.

**Edge Cases and Pitfalls**
- If $\vec{P}_{focus}$ is outside the volume, the distance calculations still hold, but the effect may be unintuitive [G].
- Precision issues may arise if $\vec{P}_{voxel}$ and $\vec{P}_{focus}$ are very close, leading to $\Delta \rho \approx 0$ and thus $\alpha_{voxel} \approx 1$ [G].
- The term $\max(0, \dots)$ ensures that opacity is only reduced when the voxel is closer to the focus point than to the center, preventing unintended brightening of outer layers [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_zoom_opacity(voxel_pos, center_pos, focus_pos, scale_factor):
    dist_center = norm(voxel_pos - center_pos)
    dist_focus = norm(voxel_pos - focus_pos)
    delta_rho = max(0.0, dist_center - dist_focus)
    return exp(-scale_factor * delta_rho)
```

## Subtopics

### MIDA Extension (Opacity Modulation)
This variant extends the Maximum Intensity Display (MIDA) technique by modulating only the opacity contribution based on importance measurements, rather than applying the maximum operator to both color and opacity [S3, unofficial]. It accumulates colors and opacities along rays as in DVR but modulates opacity when a local maximum is found [S3, unofficial]. The modulation is driven by importance measurements derived from gradient magnitudes, depth, intensity information, or a combination of these values [S3, unofficial]. This approach aims to enhance depth perception and reveal the arrangement of rendered structures with minimum loss of information [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity**
- **Time**: $O(N)$ per frame, where $N$ is the number of voxels or samples, as it requires computing distances for each voxel [G].
- **Memory**: $O(1)$ additional memory per voxel for storing the focus point and intermediate distance calculations [G].

**Measured Performance**
- Not given in the sources.

**Scaling**
- The method scales linearly with the number of voxels [G]. Performance may be impacted by the computational cost of distance calculations on large volumes [G].

**Strengths and Limitations**
- **Strengths**: Allows for detailed inspection of specific regions without losing context. Effective for revealing inner structures near the focus point [S1, unofficial][S2, unofficial].
- **Limitations**: The loss of information is a key issue, as modifying opacity may conceal some information that is worth preserving [S3, unofficial]. The effectiveness depends on the choice of $\vec{P}_{focus}$ and $S(t)$ [S1, unofficial][S2, unofficial].

**When to Use**
- Use when the user needs to inspect specific inner structures of a volume while maintaining a sense of the overall context [S1, unofficial][S2, unofficial].
- Suitable for interactive exploration where the focus point changes frequently [G].

**When Not to Use**
- Not suitable when preserving all information is critical, as opacity modulation may hide details [S3, unofficial].
- Less effective if the focus point is not well-defined or if the volume has no clear inner/outer structure [G].

## Background
The Radial Opacity Change (Zooming) technique was introduced in the paper "Dynamic Focus + Context for Volume Rendering" [S1, unofficial][S2, unofficial]. It is part of a broader effort to enhance depth perception and reveal inner structures in volume visualization [S1, unofficial][S2, unofficial]. The MIDA extension, described in "Enhanced perception in volume visualization" [S3, unofficial], further refines this approach by integrating importance-based opacity modulation.

## Sources
[S1] Dynamic Focus + Context for Volume Rendering, https://doi.org/10.2312/pe/vmv/vmv10/331-338, fetched 2026-10-26 (unofficial)
[S2] PDF Dynamic Focus+Contextfor Volume Rendering - GitHub Pages, https://sbruckner.github.io/assets/pdf/Sikachev-2010-DFC.pdf, fetched 2026-10-27 (unofficial)
[S3] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
