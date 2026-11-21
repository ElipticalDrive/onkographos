# Volume SDF

Also called: Isocontour SDF Generation, SDF Rebuild, Volume to SDF Conversion

## Overview
The Volume SDF operation rebuilds a signed distance field (SDF) from a given isocontour within a volumetric dataset [S1]. It converts a scalar volume (such as a fog density field) into a valid distance field where the zero-level set corresponds to the specified iso-surface, enabling fast distance queries and renormalization of distorted SDFs [S1]. Unlike [SDF to Density Conversion](1282-sdf-to-density-conversion.md), which maps distance to opacity, this method explicitly enforces the Eikonal equation constraint $|\nabla d(\mathbf{x})| = 1$ to ensure the output is a geometrically valid distance field [S1, Parent]. It differs from [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md) by operating on existing scalar volumes rather than generating SDFs from primitive geometry or point clouds [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A standard volumetric primitive containing scalar values. The method does not support VDB volumes [S1].
*   **Output:** A new volume primitive of identical resolution and dimensions, where each voxel stores the signed Euclidean distance to the target isocontour [S1].

**Data Structures:**
The input and output are dense 3D grids. Let $V_{in}$ be the input volume and $V_{out}$ be the output volume. Both are indexed by integer coordinates $(i, j, k)$ corresponding to voxel centers. The spatial resolution is defined by the voxel dimensions $\Delta x, \Delta y, \Delta z$. The smallest voxel dimension is denoted as $\Delta_{min} = \min(\Delta x, \Delta y, \Delta z)$ [S1].

**Parameters:**
1.  **Iso Surface ($I_{iso}$):** The scalar value in $V_{in}$ that defines the zero-level set of the output SDF. Typical values: $0.5$ for fog volumes, $0$ for existing SDF sources [S1].
2.  **Invert ($b_{inv}$):** Boolean flag. If true, the sign of the distance is flipped relative to the input gradient direction. Default is false (higher input values yield positive distances) [S1].
3.  **Use Maximum Distance ($b_{max}$):** Boolean flag. If true, computation is limited to a band around the isocontour [S1].
4.  **Maximum Distance ($D_{max}$):** The object-space distance threshold. Voxels with computed distance $|d| > D_{max}$ are clamped to $\pm D_{max}$ [S1].
5.  **Rebuild With Fast Iterative Method ($b_{fim}$):** Boolean flag. Selects the Fast Iterative Method (FIM) if true, otherwise uses the Fast Marching Method (FMM) [S1].
6.  **FIM Tolerance ($\tau_{frac}$):** Fraction of $\Delta_{min}$ used for convergence checking. Absolute tolerance $\tau = \tau_{frac} \cdot \Delta_{min}$ [S1].
7.  **Max FIM Iterations ($N_{max}$):** Hard limit on iterations for FIM. Should be several times larger than the bandwidth size in voxels [S1].

**Algorithm:**
The process identifies the zero-crossing of the isocontour $I_{iso}$ within $V_{in}$ and propagates distance values outward. The propagation method depends on $b_{fim}$.

**Case 1: Fast Marching Method (Default)**
FMM is a Dijkstra-like algorithm that solves the Eikonal equation $|\nabla d|^2 = 1$ by processing voxels in increasing order of distance.
1.  Initialize $V_{out}$ with $\infty$.
2.  Identify the "narrow band" of voxels adjacent to the isocontour $I_{iso}$. Set their initial distances based on linear interpolation between grid points where the sign of $(V_{in} - I_{iso})$ changes.
3.  Insert these voxels into a min-heap priority queue $Q$, keyed by their current distance estimate.
4.  While $Q$ is not empty:
    a. Extract voxel $v$ with minimum distance $d_v$ from $Q$.
    b. Mark $v$ as "accepted" (finalized).
    c. For each neighbor $u$ of $v$ not yet accepted:
        i. Compute candidate distance $d_u$ using the upwind finite difference approximation of the Eikonal equation.
        ii. If $d_u < V_{out}[u]$, update $V_{out}[u] = d_u$ and update $u$'s position in $Q$.

**Case 2: Fast Iterative Method (FIM)**
FIM is an iterative relaxation method that is generally faster but less accurate than FMM [S1].
1.  Initialize $V_{out}$ with $\infty$.
2.  Identify the isocontour voxels (where $V_{in} \approx I_{iso}$). Set their distance to $0$.
3.  Define the active set $A$ as all voxels within distance $D_{max}$ of the isocontour (if $b_{max}$ is true) or all voxels (if $b_{max}$ is false).
4.  Iterate $n$ from $0$ to $N_{max}$:
    a. Create a temporary copy $V_{temp} = V_{out}$.
    b. For each voxel $v \in A$:
        i. If $v$ is on the isocontour, $V_{temp}[v] = 0$.
        ii. Else, compute the new distance estimate $d_{new}$ by averaging the distances of its neighbors, constrained by the gradient magnitude condition $|\nabla d| \le 1$. A common approximation is:
        $$ d_{new} = \min_{u \in \text{neighbors}(v)} (V_{out}[u] + \|\mathbf{x}_v - \mathbf{x}_u\|) $$
        iii. Update $V_{temp}[v] = d_{new}$.
    c. Check for convergence: If $\max_{v \in A} |V_{temp}[v] - V_{out}[v]| < \tau$, break loop.
    d. $V_{out} = V_{temp}$.

**Sign Handling:**
If $b_{inv}$ is false, the sign of $V_{out}$ follows the gradient of $V_{in}$ relative to $I_{iso}$. If $b_{inv}$ is true, $V_{out} = -V_{out}$. For fog volumes, inversion is typically required so that high-density (white) regions are negative (inside) [S1].

**Clamping:**
If $b_{max}$ is true, after computation, for all voxels $v$:
$$ V_{out}[v] = \text{clamp}(V_{out}[v], -D_{max}, D_{max}) $$

**Edge Cases and Pitfalls:**
*   **VDB Incompatibility:** The method does not work with VDB volumes; it requires standard dense volume primitives [S1].
*   **Boundary Conditions:** The output volume boundary condition is set to SDF by this operation. To change it to constant or streak, a separate Primitive SOP is required [S1].
*   **Convergence:** If $N_{max}$ is too low, the FIM may not converge, resulting in inaccurate distances far from the surface. $N_{max}$ should be several times the bandwidth size in voxels [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Fast Marching Method (FMM): $O(N \log N)$, where $N$ is the number of voxels in the narrow band or volume. FMM is exact but slower due to heap operations [S1].
    *   Fast Iterative Method (FIM): $O(k \cdot N)$, where $k$ is the number of iterations until convergence. FIM is generally faster than FMM but approximate [S1].
*   **Memory Complexity:** $O(N)$ for storing the output volume and auxiliary data structures (heap for FMM, active set for FIM). The output resolution matches the input [S1].
*   **Performance:** FIM is preferred when performance is prioritized over accuracy [S1]. FMM is used for higher accuracy.
*   **Strengths:** Fast distance computation for ray marching or collision detection; renormalizes distorted SDFs; handles arbitrary scalar volumes (e.g., fog) [S1].
*   **Limitations:** Does not support VDBs; FIM accuracy degrades with complex topology or large bandwidths; requires dense volume storage [S1].

## Background
The Volume SDF operation was introduced in Houdini version 10.0 [S1]. It is part of the SOP (Surface Operator) network for geometry processing. The underlying algorithms, Fast Marching Method and Fast Iterative Method, are well-established numerical techniques for solving the Eikonal equation [S1]. The specific implementation in Houdini allows for the conversion of procedural or simulated volume data (like fog) into geometric distance fields for efficient rendering and simulation.

## Sources
[S1] Volume SDF - SideFX, https://www.sidefx.com/docs/houdini/nodes/sop/volumesdf.html, fetched 2026-10-27
