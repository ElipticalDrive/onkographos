# Froxel Depth Distribution

Also called: Exponential Froxel Distribution, Non-linear Froxel Depth Partitioning

## Overview
Froxel Depth Distribution is a non-linear depth partitioning scheme for froxel grids that allocates higher spatial resolution near the camera and lower resolution in the distance [S1, unofficial]. It solves the problem of insufficient depth precision in view-space voxelization, where linear spacing causes large froxels in the distance that fail to capture volumetric detail, while wasting resolution on distant, visually insignificant volumes [S1, unofficial]. It sets itself apart from linear depth partitioning by ensuring the visual angle subtended by a voxel remains relatively uniform across the depth range, prioritizing near-field fidelity [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: Normalized clip-space depth $z \in [0, 1]$, exponential distribution factor $s$.
- Output: Transformed depth value $z'$ used to index or map the froxel grid along the view ray.

**Data Structures**
The froxel grid is a 3D texture or buffer aligned with the camera frustum.
- Dimensions: $N_x \times N_y \times N_z$.
- Typical Resolution: $160 \times 90 \times 128$ [S1, unofficial].
- X and Y axes: Map linearly to normalized device coordinates (screen space) [S2, unofficial].
- Z axis: Distributed non-linearly according to the depth distribution function [S1, unofficial].

**Formula**
The depth transformation is defined as:
$$z'(z) = z^s$$
Where:
- $z$ is the normalized depth in clip space, $z \in [0, 1]$ [S1, unofficial].
- $s$ is the exponential distribution factor [S1, unofficial].
- $z'$ is the transformed depth value, representing the relative position within the froxel grid's Z-dimension [S1, unofficial].

**Parameters**
- Distribution factor $s$: A value of $s = 2.0$ provides a reasonable quality trade-off between nearby and far-away froxels [S1, unofficial]. This value was used consistently in the referenced Godot engine tests [S1, unofficial].

**Algorithm**
1. Retrieve the normalized clip-space depth $z$ for the current ray or froxel boundary.
2. Apply the exponential transformation: $z' = z^s$.
3. Map $z'$ to the discrete froxel index $k$ along the Z-axis:
   $$k = \lfloor z' \cdot N_z \rfloor$$
   Where $N_z$ is the number of froxels along the depth axis [S1, unofficial].
4. Use $(x, y, k)$ to access the froxel data [S2, unofficial].

**Edge Cases and Pitfalls**
- **Precision Loss**: As $z \to 0$, the derivative $dz'/dz = s z^{s-1}$ becomes large for $s > 1$, meaning small changes in clip space depth result in larger changes in grid index. This is desirable for near-field detail but requires careful handling of floating-point precision near the camera [S1, unofficial].
- **Boundary Conditions**: Ensure $z=0$ maps to the first froxel and $z=1$ maps to the last. Clamping $z'$ to $[0, 1]$ prevents out-of-bounds access [G].
- **Alternative Mapping**: Some implementations use a logarithmic mapping for the Z-dimension instead of power-law: $w = \frac{\ln(z / z_{near})}{\ln(z_{far} / z_{near})}$ [S2, unofficial]. This maps view-space depth to normalized texture coordinates. The power-law method operates on normalized clip-space depth directly [S1, unofficial].

## Subtopics

### [Logarithmic Depth Partitioning](3335-logarithmic-depth-partitioning.md)
Logarithmic Depth Partitioning is a non-linear depth distribution scheme for froxel grids that enforces a constant minimum angular resolution for reconstructed surface normals. It solves the problem of insufficient depth precision near the camera during voxelization, which causes discontinuities in extracted boundary surfaces. It sets itself apart from linear depth partitioning by keeping the ratio of cell depth to column width constant, ensuring that the visual angle subtended by a voxel remains uniform across the entire depth range. See [Logarithmic Depth Partitioning](3335-logarithmic-depth-partitioning.md).

### Froxel Volume (Frustum Grid)
A view-frustum-aligned 3D texture grid used to store volumetric fog data, replacing axis-aligned world-space grids to optimize for view-dependent rendering [S2, unofficial]. It maps view-space depth $z$ to a normalized texture coordinate $w$ using the formula:
$$w = \frac{\ln(z / z_{near})}{\ln(z_{far} / z_{near})}$$
Typical dimensions are $160 \times 90 \times 64$ or $256 \times 144 \times 128$, where X and Y map linearly to screen space and Z is distributed exponentially between near and far planes [S2, unofficial]. See [Froxel Volume](3336-froxel-volume.md).

## Operating characteristics
- **Time Complexity**: $O(1)$ per froxel for depth mapping. Grid construction is $O(N_x N_y N_z)$ [S1, unofficial].
- **Memory Complexity**: $O(N_x N_y N_z)$ for the froxel grid storage [S2, unofficial].
- **Performance**: The resolution $160 \times 90 \times 128$ balances performance and quality in real-time applications [S1, unofficial]. Higher resolutions increase memory bandwidth usage but improve near-field detail [S2, unofficial].
- **Strengths**: Improves visual quality near the camera where volumetric detail is most perceptible [S1, unofficial]. Reduces aliasing artifacts in depth-dependent rendering [S1, unofficial].
- **Limitations**: Non-uniform sampling may complicate certain volume integration algorithms that assume uniform spacing [G]. Requires careful handling of depth precision near the camera [S1, unofficial].
- **When to Use**: Real-time volumetric rendering where near-field detail is critical and performance constraints limit total froxel count [S1, unofficial].
- **When Not to Use**: Scenes with uniform volumetric distribution where linear spacing is sufficient, or when strict world-space consistency is required for global illumination [G].

## Background
Froxel Depth Distribution originated in the context of real-time volumetric rendering in game engines, specifically Godot, to improve the visual quality of participating media [S1, unofficial]. The method was developed to address the limitations of linear froxel grids, which wasted resolution on distant, less visible volumes [S1, unofficial]. The exponential distribution factor $s=2.0$ was empirically determined to provide a good balance between quality and performance [S1, unofficial]. The concept is closely related to view-frustum-aligned volume grids, which have been used in OpenGL-based volumetric fog rendering [S2, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
