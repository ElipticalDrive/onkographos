# Radial Opacity Change (Zooming)

## Overview
Radial Opacity Change (Zooming) is an illustrative volume rendering technique that reveals inner structures by modulating voxel opacity based on the distance from a user-defined focus point, rather than the distance from the volume center. Unlike its sibling [Radial Opacity Change](7140-radial-opacity-change.md), which uses angular velocity and distance from the center to create a "tunnel" effect during rotation, this method shifts the transparency gradient to follow the user's interaction point, allowing for detailed inspection of specific regions while maintaining context elsewhere.

## How it works
**Inputs and Outputs**
- **Input**: Volume dataset $V$, camera parameters, user interaction point $\vec{P}_{focus}$ (initial point of interaction on the object), and a scaling factor $S(t)$ that may vary with time or interaction intensity.
- **Output**: Rendered image with modulated opacity.

**Data Structures**
- **Volume Data**: Standard 3D grid of scalar values.
- **Focus Point**: A 3D vector $\vec{P}_{focus}$ representing the location in world or object space where the user is focusing.

**Algorithm**
For each voxel $\vec{P}_{voxel}$ in the volume:

1. Calculate the Euclidean distance from the voxel to the volume center $\vec{P}_{center}$:
   $$ \rho(\vec{P}_{voxel}, \vec{P}_{center}) = \| \vec{P}_{voxel} - \vec{P}_{center} \| $$
2. Calculate the Euclidean distance from the voxel to the focus point $\vec{P}_{focus}$:
   $$ \rho(\vec{P}_{voxel}, \vec{P}_{focus}) = \| \vec{P}_{voxel} - \vec{P}_{focus} \| $$
3. Compute the radial opacity change factor using the maximum of zero and the difference between the distance to the center and the distance to the focus:
   $$ \Delta \rho = \max(0, \rho(\vec{P}_{voxel}, \vec{P}_{center}) - \rho(\vec{P}_{voxel}, \vec{P}_{focus})) $$
4. Apply the exponential attenuation to determine the effective opacity $\alpha_{voxel}$:
   $$ \alpha_{voxel} = e^{-S(t) \cdot \Delta \rho} $$

**Parameters**
- $\vec{P}_{focus}$: The initial point of interaction on the object. This point defines the center of the "zoom" effect.
- $S(t)$: A scaling factor that controls the strength of the opacity modulation. It may be constant or vary with time/interaction. Typical values are not specified in the sources.

**Edge Cases and Pitfalls**
- If $\vec{P}_{focus}$ is outside the volume, the distance calculations still hold, but the effect may be unintuitive.
- Precision issues may arise if $\vec{P}_{voxel}$ and $\vec{P}_{focus}$ are very close, leading to $\Delta \rho \approx 0$ and thus $\alpha_{voxel} \approx 1$.
- The term $\max(0, \dots)$ ensures that opacity is only reduced when the voxel is closer to the focus point than to the center, preventing unintended brightening of outer layers.

## Subtopics

### MIDA Extension (Opacity Modulation)
This variant extends the Maximum Intensity Display (MIDA) technique by modulating only the opacity contribution based on importance measurements. Instead of using the maximum operator for both color and opacity, it accumulates colors and opacities along rays as in DVR but modulates opacity when a local maximum is found. The modulation is driven by importance measurements derived from gradient magnitudes, depth, intensity information, or a combination of these values [S2, unofficial]. This approach aims to enhance depth perception and reveal the arrangement of rendered structures with minimum loss of information, addressing the main issue of information loss in MIP visualizations when adding color cues or modifying shading [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity**
- **Time**: $O(N)$ per frame, where $N$ is the number of voxels or samples, as it requires computing distances for each voxel.
- **Memory**: $O(1)$ additional memory per voxel for storing the focus point and intermediate distance calculations.

**Measured Performance**
- Not given in the sources.

**Scaling**
- The method scales linearly with the number of voxels. Performance may be impacted by the complexity of calculating distances for large volumes.

**Strengths and Limitations**
- **Strengths**: Allows for detailed inspection of specific regions without losing context. Effective for revealing inner structures near the focus point.
- **Limitations**: The loss of information is a key issue, as modifying opacity may conceal some information that is worth preserving. The effectiveness depends on the choice of $\vec{P}_{focus}$ and $S(t)$.

**When to Use**
- Use when the user needs to inspect specific inner structures of a volume while maintaining a sense of the overall context.
- Suitable for interactive exploration where the focus point changes frequently.

**When Not to Use**
- Not suitable when preserving all information is critical, as opacity modulation may hide details.
- Less effective if the focus point is not well-defined or if the volume has no clear inner/outer structure.

## Background
The Radial Opacity Change (Zooming) technique was introduced in the paper "Dynamic Focus + Context for Volume Rendering" [S1, unofficial]. It is part of a broader effort to enhance depth perception and reveal inner structures in volume visualization. The technique builds on the concept of radial opacity change but adapts it for zooming interactions, where the user focuses on a specific point and the opacity is modulated accordingly. The MIDA extension, described in "Enhanced perception in volume visualization" [S2, unofficial], further refines this approach by integrating importance-based opacity modulation.

## Sources
[S1] Dynamic Focus + Context for Volume Rendering, https://doi.org/10.2312/pe/vmv/vmv10/331-338, fetched 2026-10-26 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
