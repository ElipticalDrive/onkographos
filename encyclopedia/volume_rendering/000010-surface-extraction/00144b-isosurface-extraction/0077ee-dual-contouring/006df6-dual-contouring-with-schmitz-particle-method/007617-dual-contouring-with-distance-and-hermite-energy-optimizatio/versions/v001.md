# Dual Contouring with Distance and Hermite Energy Optimization

Also called: Iterative Dual Contouring, Local SDF Reconstruction

## Overview
This method reconstructs a quad mesh from Signed Distance Function (SDF) data on a regular grid by locally updating vertex positions within each cell to minimize a combined energy of distance to SDF samples and Hermite consistency [S1]. It avoids global geometric flow by using local updates, which removes the complexity of signed energy treatment required by methods like Sellán et al. [2023] that rely on global optimization [S1]. Unlike its parent method, which uses a Schmitz particle approach for initial vertex placement, this method iteratively corrects Hermite data to recover sharp features without requiring exact Hermite data or access to the SDF function derivatives [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A uniform regular grid with SDF values $s_i$ at vertices $u_i \in \mathbb{R}^3$.
*   **Output:** A quad mesh $M = (V, Q)$ approximating the zero level set $\Omega$.

**Data Structures:**
*   **Interesting Edges:** Edges of the grid whose endpoints have SDF values with different signs. These are known to intersect the true surface.
*   **Interesting Cells ($C$):** The set of cells containing any interesting edge. $C = \bigcup c_i$.
*   **Hermite Data:** For each interesting edge $j$, a point $\mathbf{h}_j$ and a normal $\mathbf{n}_j$.
*   **Cell Vertices:** For each interesting cell $i$, a single vertex position $\mathbf{x}_i$.

**Algorithm Steps:**

1.  **Initialization:**
    *   Identify all interesting edges where $s_{u_a} \cdot s_{u_b} < 0$.
    *   Tag all cells containing these edges as interesting cells $C$.
    *   Initialize the vertex position for each interesting cell $i$ to its centroid: $\mathbf{x}_i^0 = \mathbf{c}_i$.
    *   Initialize Hermite data for each interesting edge $j$. The sources do not specify the exact initialization formula for $\mathbf{h}_j$ and $\mathbf{n}_j$, but imply they are estimated from the discrete SDF samples [S1]. [G: A common initialization is to set $\mathbf{h}_j$ to the midpoint of the edge and $\mathbf{n}_j$ to the gradient of the SDF interpolated at the midpoint, or simply the average of the normals of adjacent cells if available. If not, set $\mathbf{n}_j$ to the edge direction or a default normal.]

2.  **Outer Iterative Loop:**
    Repeat for $k = 0, 1, \dots, K_{max}$ (default $K_{max}=100$):
    
    2.1. **Construct Global Mesh $M^k$:**
    *   For each interesting edge $j$, connect the four cell vertices $\{\mathbf{x}_i\}$ of the cells sharing edge $j$ to form a quad mesh. This forms the global mesh $M^k$ for the current iteration.

    2.2. **Inner Iterative Loop (Cell Optimization):**
    *   For each interesting cell $i$, solve for the optimal vertex position $\mathbf{x}_i$ by minimizing the local energy. This is done in parallel for all cells using OpenMP [S1].
    *   Initialize inner iteration $r=0$ with $\mathbf{x}_i^{k,0} = \mathbf{x}_i^k$.
    *   Repeat for $r = 0, 1, \dots, R_{max}$ (default $R_{max}=100$):
        *   **Compute Distance Energy Term:**
            *   For each SDF sample $j$ associated with cell $i$ (typically the vertices of the cell or nearby grid points):
                *   Let $u_j$ be the location and $s_j$ be the SDF value.
                *   Interpret $(u_j, s_j)$ as a sphere centered at $u_j$ with radius $|s_j|$.
                *   Find $\mathbf{t}_j$, the closest point on the current global mesh $M^k$ (specifically the local mesh inside cell $i$) to $u_j$.
                *   Find $\mathbf{q}_j$, the closest point on the sphere (center $u_j$, radius $|s_j|$) to $\mathbf{t}_j$.
                *   Compute $\mathbf{d}_j$, the normalized vector from $u_j$ to $\mathbf{q}_j$: $\mathbf{d}_j = \frac{\mathbf{q}_j - u_j}{\|\mathbf{q}_j - u_j\|}$.
                *   The linearized distance energy contribution is based on the projection of the vector $(\mathbf{t}_j - \mathbf{q}_j)$ onto $\mathbf{d}_j$. The term is $((\mathbf{t}_j - \mathbf{q}_j) \cdot \mathbf{d}_j)^2$. Note: $\mathbf{t}_j$ depends on $\mathbf{x}_i$, making this a non-linear problem, but it is solved iteratively [S1].
        *   **Compute Hermite Energy Term:**
            *   For each interesting edge $j$ shared by cell $i$:
                *   The Hermite energy encourages the vertex $\mathbf{x}_i$ to lie on the plane defined by the Hermite data $(\mathbf{h}_j, \mathbf{n}_j)$ of the shared edges.
                *   $E_H^k(\mathbf{x}_i) = \sum_{j \in \text{edges}(i)} ((\mathbf{x}_i - \mathbf{h}_j^k) \cdot \mathbf{n}_j^k)^2$.
        *   **Minimize Total Energy:**
            *   Minimize the following energy function with respect to $\mathbf{x}_i$:
                $$ E(\mathbf{x}_i) = \tilde{E}_d^{k,r}(\mathbf{x}_i) + w_H^2 E_H^k(\mathbf{x}_i) + \mu \|\mathbf{x}_i - \mathbf{x}_i^{k,r}\|^2 $$
                where:
                *   $\tilde{E}_d^{k,r}(\mathbf{x}_i)$ is the linearized distance energy term described above.
                *   $w_H$ is the weight balancing distance and Hermite energy (default 0.02) [S1].
                *   $\mu$ is the regularization parameter (default 0.1, range $[10^{-2}, 10^{-1}]$) [S1].
            *   This is a quadratic optimization problem (QEF) and can be solved using least squares.
        *   **Update Vertex Position:**
            *   $\mathbf{x}_i^{k,r+1} = \arg \min_{\mathbf{x}_i} E(\mathbf{x}_i)$.
        *   **Check Convergence:**
            *   If $\|\mathbf{x}_i^{k,r+1} - \mathbf{x}_i^{k,r}\| < \theta$ (threshold) or $r = R_{max}$, break inner loop.
    *   Update cell vertex: $\mathbf{x}_i^{k+1} = \mathbf{x}_i^{k, r_{final}}$.

    2.3. **Update Hermite Data:**
    *   For each interesting edge $j$ shared by four cells:
        *   Perform Principal Component Analysis (PCA) on the four vertices $\{\mathbf{x}_i\}$ of the cells sharing edge $j$ to obtain the best-fit plane normal $\mathbf{n}$ and intersection point $\mathbf{y}$ with the edge.
        *   Update Hermite point $\mathbf{h}_j$ and normal $\mathbf{n}_j$ using linear interpolation with update weight $w_u$ (default 0.2) [S1]:
            $$ \mathbf{h}_j^{k+1} = \mathbf{h}_j^k + w_u(\mathbf{y} - \mathbf{h}_j^k) $$
            $$ \mathbf{n}_j^{k+1} = \frac{\mathbf{n} + w_u\mathbf{n}_j^k}{\|\mathbf{n} + w_u\mathbf{n}_j^k\|} $$
    *   If $k = K_{max}$, terminate outer loop.

3.  **Final Mesh Construction:**
    *   Join together the final cell vertices $V = \{\mathbf{x}_i\}$ and the connectivity defined by the interesting cells to form the output mesh $M$ [S1].

**Parameters:**
*   $w_H$: Weight for Hermite energy. Default: 0.02 [S1].
*   $\mu$: Regularization parameter. Default: 0.1. Range: $[10^{-2}, 10^{-1}]$ [S1].
*   $w_u$: Update weight for Hermite data. Default: 0.2 [S1].
*   $R_{max}$: Max inner iterations. Default: 100 [S1].
*   $K_{max}$: Max outer iterations. Default: 100 [S1].
*   Batch Size: If using all SDF data at high resolution, random batching is used. Max batch size: 200,000 [S1].

**Edge Cases and Pitfalls:**
*   **Vertex Movement:** Vertices are allowed to move outside their corresponding grid cells. This is crucial for reconstructing sharp features [S1]. Constraining vertices inside the cell degrades sharp feature recovery.
*   **Linearization:** The distance energy is linearized by measuring distance only in the direction towards the sphere center $\mathbf{d}_j$, not tangentially. This prevents the vertex from sliding tangentially to satisfy other spheres' distances [S1].
*   **Sign Ignorance:** The method ignores the sign of the SDF distance locally. This is valid because the local updates and Hermite consistency ensure topological separation of positive and negative SDF data by construction [S1].

## Subtopics

### Iterative Dual Contouring with Discrete SDF
An iterative algorithm that reconstructs surfaces from discrete SDF samples by progressively correcting an initial estimate of Hermite data to optimize vertex placement in each cell [S1]. It identifies interesting edges (vertices with different sign SDF values) and tags cells containing them as interesting cells. It solves energy minimization to update vertex position $x_i^{k+1}$, repeating until convergence [S1]. This variant emphasizes the use of purely discrete SDF samples without requiring continuous SDF function access or derivatives [S1].

## Operating characteristics
*   **Time Complexity:** Runtime scales linearly with input data size, number of inner iterations, and number of outer iterations [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On a Macbook Pro with M4 Pro chip and 48 GB memory, the method is efficient due to parallelization using OpenMP for vertex optimizations per cell [S1].
*   **Scaling:** The method is agnostic to input structure. At low resolutions, all grid information can be used. At higher resolutions, a narrow band around the zero level set can be used to reduce redundancy. If all data is used at high resolution, random batching (max 200,000 samples) can be employed [S1].
*   **Strengths:** Recovers sharp features nearly as well as Ju et al. [2002] without requiring exact Hermite data or SDF function access. Removes complexity of signed energy treatment compared to global methods like Sellán et al. [2023] [S1].
*   **Limitations:** Requires iterative optimization which can be computationally expensive if many iterations are needed. Sensitive to the choice of $w_H$ [S1].
*   **When to Use:** When reconstructing surfaces from discrete SDF data with sharp features, and when global optimization is too expensive or complex.
*   **When Not to Use:** When real-time performance is critical and iterative convergence is too slow. When the input SDF is smooth and does not require sharp feature recovery.

## Background
This method was introduced in "Dual Contouring of Signed Distance Data" [S1]. It builds on the contouring strategy proposed by Ju et al. [2002], which optimizes vertex placements using a quadratic error function (QEF). However, Ju et al. [2002] requires exact Hermite data. This method exploits information in discrete SDF samples to iteratively correct initial Hermite data, allowing it to recover sharp features without restrictive assumptions on the input [S1]. It contrasts with global geometric flow methods like Sellán et al. [2023], which require sign agreement metrics and global optimization [S1].

## Sources
[S1] Dual Contouring of Signed Distance Data, https://arxiv.org/pdf/2604.00157, fetched 2026-10-26
