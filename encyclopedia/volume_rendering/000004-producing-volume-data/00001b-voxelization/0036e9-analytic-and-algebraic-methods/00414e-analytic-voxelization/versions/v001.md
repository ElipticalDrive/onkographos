# Analytic Voxelization

Also called: Digitization, Digitization (T_o

## Overview
Analytic Voxelization converts parametrically defined continuous geometric objects into digital volume representations by assigning each voxel a membership value in $[0, 1]$ reflecting the fraction of the voxel's total volume occupied by the object [S1]. Unlike binary voxelization methods that use all-or-none occupancy flags, this approach retains the inaccuracies and boundary ambiguities of the digitization process naturally, creating a grey-scale scene representation suitable for applications like prosthesis design and radiation therapy planning [S1]. It sets itself apart from [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) by providing exact volumetric overlap fractions rather than distance metrics, and from [Surface-Aware Discretization](6820-surface-aware-discretization.md) by focusing on the continuous-to-discrete mapping of volume rather than surface approximation [S1].

## How it works
**Inputs:** A parametric continuous object description $Q_c$ (e.g., a sphere defined by center and radius) and a target voxel grid resolution.
**Outputs:** A digital object representation $D_p$ where each voxel $v$ holds a scalar value $m(v) \in [0, 1]$.

**Data Structures:**
*   **Voxel Grid:** A 3D array of floating-point values.
*   **Continuous Object:** Defined by parametric equations or implicit functions.

**Steps:**
1.  **Define Digitization Transform:** The process is characterized by the transform $T_o: Q_c \to D_p$, mapping the set of parametric object descriptions to the digital representation [S1].
2.  **Iterate Voxels:** For each voxel in the target grid, determine its spatial relationship with the continuous object.
3.  **Calculate Membership Value:**
    *   If the voxel is completely inside the object, assign $m(v) = 1$.
    *   If the voxel is completely outside the object, assign $m(v) = 0$.
    *   If the voxel intersects the boundary of the object, calculate the exact fraction of the voxel's volume occupied by the object. Assign this fraction to $m(v)$ [S1].

**Formula:**
For a voxel $v$ with volume $V_v$ and the intersection of the continuous object $Q_c$ with the voxel $V_{int} = Q_c \cap v$, the membership value is:
$$ m(v) = \frac{\text{Volume}(V_{int})}{V_v} $$
where $\text{Volume}(V_{int})$ is the analytic volume of the intersection [S1].

**Example:**
For a sphere, the region of the resulting scene is a cube that encloses the sphere. Voxels completely inside or outside are assigned 1 or 0, respectively. Intersecting voxels are assigned the fraction of the voxel volume intersected by the sphere [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Calculating the exact intersection volume for complex shapes may require robust geometric clipping or numerical integration.
*   **Boundary:** The method explicitly retains boundary inaccuracies, which is a feature for soft segmentation but requires care if hard boundaries are needed later [S1].

## Subtopics

### Precomputed Obstacle Voxelization
Precomputed Obstacle Voxelization is a variant for scenarios involving obstacles undergoing simple analytic transformations [S2, unofficial]. It involves precomputing the voxelization of the obstacle mesh once. During runtime, when accessing inside-outside or obstacle velocity textures, the texture coordinates are multiplied by the inverse of the transformation matrix to retrieve the correct precomputed values [S2, unofficial]. This method is limited to simple analytic transformations (e.g., rigid translation and rotation) and does not support complex skinning operations [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The complexity depends on the cost of calculating the intersection volume for each voxel. For simple shapes like spheres, this can be done analytically in constant time per voxel. For complex meshes, it may require clipping or sampling. Memory usage is proportional to the number of voxels in the grid [S1].

**Strengths and Limitations:**
*   **Strengths:** Retains geometric accuracy at boundaries; suitable for applications requiring soft transitions or physical conservation (implied by the fraction-based approach) [S1].
*   **Limitations:** Computationally expensive for complex shapes if exact intersection volumes are required; not suitable for real-time simulation of complex deformable objects without precomputation or simplification [S2, unofficial].

**When to Use:**
Use when visualizing or analyzing continuous parametric objects in medical or engineering contexts where boundary accuracy and volumetric fractions are critical [S1]. Use for static or rigidly moving obstacles in fluid simulations to avoid per-frame voxelization costs [S2, unofficial].

## Background
The concept of digitizing continuous geometric objects into digital representations was introduced by Upupa and Goncalves [S1]. They characterized the digitization operation as a transform from parametric object descriptions to digital sets, emphasizing the retention of inaccuracies through fractional membership values. This approach bridges the gap between continuous parametric models and discrete digital entities, enabling structure-related operations like visualization and manipulation [S1]. The application of precomputed voxelization for transformed obstacles was later discussed in the context of real-time fluid simulation [S2, unofficial].

## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
