# Image Transformation Methods

Also called: Drebin/Hanrahan Transformation, Two-Pass Image Transformation, Three-Pass Image Transformation, Forward Projection via Transformation, Feed-forward Rotate-and-Draw

## Overview
Image Transformation Methods are feed-forward (projective) volume rendering techniques where the volume data is transformed into an intermediate space aligned with the image plane, allowing standard 2D compositing or projection [S2, unofficial]. Unlike image-order methods that trace rays through the volume, these methods enumerate voxels, apply a viewing transformation, and then project the result [S2, unofficial]. They are distinguished from their siblings by their specific decomposition of the transformation into incremental beam and slice updates to exploit spatial coherency and parallelism [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume $V$ represented as a rectilinear grid of resolution $N \times N \times N$ (total $N^3$ voxels) [S2, unofficial]. A 4x4 homogeneous transformation matrix $M$ mapping object space to image space [S2, unofficial].
*   **Output:** A transformed intermediate volume (or set of slices) ready for 2D projection/compositing [S2, unofficial].

**Data Structures:**
*   **Volume Grid:** A 3D array of voxels.
*   **Slices:** A slice $S_{z:k}$ is the set of grid points where $z=k$, i.e., $S_{z:k} = \{(x,y,z) \mid z=k, 0 \le x,y < N\}$ [S2, unofficial].
*   **Beams:** A beam $B_{x:(j,k)}$ is the set of grid points where $y=j$ and $z=k$, i.e., $B_{x:(j,k)} = \{(x,y,z) \mid y=j, z=k, 0 \le x < N\}$ [S2, unofficial].

**Transformation Equation:**
The position $v$ in object space and $v'$ in image space are related by:
$$ [v_x, v_y, v_z, 1] = [v'_x, v'_y, v'_z, 1] M $$
where $M$ is a 4x4 matrix [S2, unofficial].

**Algorithm Steps (Incremental Update Scheme):**
The method reduces the cost of transforming $N^3$ voxels from $16N^3$ multiplications to $3N^3$ additions by exploiting spatial coherency [S2, unofficial]. The transformation is decomposed into four phases:

1.  **Seed Phase:**
    *   Transform the first grid point (e.g., the lowest point of the first slice, $v_{0,0,0}$) using a full matrix-vector multiplication [S2, unofficial].
    *   This result serves as the seed for subsequent incremental updates [S2, unofficial].

2.  **Vertical Beam Phase:**
    *   Transform the first vertical beam $B_{y:(0,k)}$ (along the y-axis for fixed $x=0, z=k$) using the seed [S2, unofficial].
    *   For adjacent points along the beam, the update is a scalar addition of the second column of $M$ ($[m_{01}, m_{11}, m_{21}, 0]^T$) [S2, unofficial].
    *   Formula: $v'_{next} = v'_{prev} + [m_{01}, m_{11}, m_{21}, 0]^T$ [S2, unofficial].

3.  **Horizontal Beam Phase:**
    *   Transform the remaining beams $B_{x:(j,k)}$ for $0 < j < N$ [S2, unofficial].
    *   Use the already transformed beam $B_{y:(j,k)}$ to update the next beam $B_{y:(j+1,k)}$? *Correction based on source text:* The source states "Beam $B_{y:(j,k)}$ is used to update beam $B_{y:(j+1,k)}$". However, standard incremental logic along x-beams implies updating along the x-axis. The source text says: "The first column of matrix M is used for incremental updates, since the enumeration is along the x-axis." This implies updating points *within* a beam along x, or updating beams based on x-index.
    *   Re-reading [S2, unofficial]: "Horizontal Beam Phase: In the third phase beams $B_{x:(j,k)}, 0 < j < N$ are updated. Beam $B_{y:(j,k)}$ is used to update beam $B_{y:(j+1,k)}$." This phrasing is contradictory to the label "Horizontal Beam" if it updates y-beams. However, it continues: "The first column of matrix M is used for incremental updates, since the enumeration is along the x-axis."
    *   *Interpretation:* The source likely describes updating the *next* beam in the x-direction (index $j+1$) using the current beam (index $j$). The vector added is the first column of $M$ ($[m_{00}, m_{10}, m_{20}, 0]^T$) [S2, unofficial].
    *   Formula: $v'_{beam_{j+1}} = v'_{beam_j} + [m_{00}, m_{10}, m_{20}, 0]^T$ [S2, unofficial].
    *   At the end of this phase, slice $S_{z:k}$ is completely transformed [S2, unofficial].

4.  **Slice Phase:**
    *   Incrementally update the next slice $S_{z:k+1}$ using the updated slice $S_{z:k}$ [S2, unofficial].
    *   The update vector is the third column of $M$ ($[m_{02}, m_{12}, m_{22}, 0]^T$) [S2, unofficial].
    *   Formula: $v'_{slice_{k+1}} = v'_{slice_k} + [m_{02}, m_{12}, m_{22}, 0]^T$ [S2, unofficial].
    *   This is a vector addition with vector length $N^2$ [S2, unofficial].

**Parameters:**
*   **Resolution $N$:** The grid dimension. Total voxels $N^3$ [S2, unofficial].
*   **Transformation Matrix $M$:** Constructed from Euler angles. Drebin's formulation uses:
    $$ T = P_z Z_e R_z(\psi) R_y(\theta) R_z(\phi) $$
    where $P$ is perspective, $Z_e$ is a translation, $R_z$ is rotation about Z, and $R_y$ is rotation about Y [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Artifacts:** Direct application of the transformation matrix introduces artifacts in the image [S2, unofficial]. Reconstruction or interpolation in 3D is required to annul these effects [S2, unofficial].
*   **Precision:** Incremental updates accumulate floating-point error. For large $N$, periodic re-computation of the seed or full matrix multiplication may be required to prevent drift [G].
*   **Boundary Conditions:** The incremental scheme assumes regular grid spacing. Non-uniform grids require different handling [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_volume_incremental(volume, M, N):
    """
    Transforms a volume of size N x N x N using incremental updates.
    M is a 4x4 transformation matrix.
    Returns transformed_volume.
    """
    # Extract columns of M for incremental updates
    # M is defined as [v] = [v'] * M in the source, but standard matrix mult is [v'] = M * [v] or [v] = M * [v'].
    # Source [S2, unofficial] says: [v_x, v_y, v_z, 1] = [v'_x, v'_y, v'_z, 1] M.
    # This implies row vector on left.
    # Let's define M_col0, M_col1, M_col2 as the first, second, third columns of M.
    # Note: The source says "Add the first column... for x-axis enumeration".
    # If v' = v * M, then v'_next = (v + delta) * M = v*M + delta*M.
    # delta = [1, 0, 0, 0] for x-step. delta*M is the first row of M?
    # Source says: "Add the first column of matrix M".
    # This implies the source might be using column vectors v' = M * v, or there is a transpose convention difference.
    # Given "Add the first column... for x-axis", if v' = M * v, then v'_next = M*(v+[1,0,0,0]) = M*v + M*[1,0,0,0] = v' + col0.
    # So we assume column vector convention for the update logic described: v' = M * v.
    
    col0 = [M[0][0], M[1][0], M[2][0], 0]
    col1 = [M[0][1], M[1][1], M[2][1], 0]
    col2 = [M[0][2], M[1][2], M[2][2], 0]
    
    transformed_volume = empty_array(N, N, N)
    
    # Seed Phase
    # Transform first voxel (0,0,0)
    v_seed = [0, 0, 0, 1]
    v_prime_seed = multiply_matrix_vector(M, v_seed)
    transformed_volume[0][0][0] = v_prime_seed
    
    # Vertical Beam Phase
    # Transform first vertical beam B_y:(0,k) for all k? 
    # Source says: "first vertical beam, B_y:(0,k) is updated using the seed."
    # This implies for a fixed k, we transform the beam along y.
    # But the loop structure usually iterates k, then j, then i.
    # Let's follow the source's phase description strictly.
    
    for k in range(N):
        # Vertical Beam Phase: Transform B_y:(0,k)
        # Start from seed of the slice? Or global seed?
        # Source: "first vertical beam... is updated using the seed."
        # Assuming seed for the slice k is transformed first.
        # Let's assume we transform the first point of the beam (0,0,k) using full mult?
        # Or use the global seed? The global seed is (0,0,0).
        # For k>0, we need a starting point for the beam.
        # The Slice Phase updates slices. So we should process slices in order.
        
        # Re-structuring loops to match phases:
        # 1. Seed: (0,0,0)
        # 2. Vertical Beam: Beam y at x=0, z=0. (0, y, 0) for y in 1..N-1
        # 3. Horizontal Beam: Beams x at z=0. (x, y, 0) for x in 1..N-1, y in 0..N-1?
        #    Source: "Beam B_y:(j,k) is used to update beam B_y:(j+1,k)".
        #    This updates the *next* y-beam using the *current* y-beam.
        #    This implies we iterate j (x-index of the beam).
        #    So for a fixed k, we have beams B_y:(0,k), B_y:(1,k)...
        #    We transform B_y:(0,k) (Vertical Beam Phase).
        #    Then we use B_y:(0,k) to transform B_y:(1,k) (Horizontal Beam Phase).
        #    Then B_y:(1,k) to B_y:(2,k), etc.
        
        # Let's implement for slice k=0 first.
        pass

    # Corrected Loop Structure based on Source Phases:
    
    # 1. Seed Phase: Transform (0,0,0)
    v_prime_000 = multiply_matrix_vector(M, [0, 0, 0, 1])
    transformed_volume[0][0][0] = v_prime_000
    
    # 2. Vertical Beam Phase: Transform B_y:(0,0)
    # Beam along y at x=0, z=0.
    # Start with (0,0,0). Next is (0,1,0).
    # Update: add col1.
    for y in range(1, N):
        v_prev = transformed_volume[0][y-1][0]
        v_curr = add_vectors(v_prev, col1)
        transformed_volume[0][y][0] = v_curr
        
    # 3. Horizontal Beam Phase: Transform remaining beams in slice k=0
    # Beams B_y:(j,0) for j=1 to N-1.
    # Use B_y:(j-1,0) to update B_y:(j,0).
    # Update: add col0.
    for j in range(1, N):
        # Beam B_y:(j-1,0) is fully transformed.
        # Beam B_y:(j,0) starts at (j,0,0).
        # Wait, the source says "Beam B_y:(j,k) is used to update beam B_y:(j+1,k)".
        # This implies the *entire* beam is shifted by col0.
        # So v_prime(x, j+1, k) = v_prime(x, j, k) + col0?
        # No, that would mean the beam is just translated.
        # The source says: "The first column of matrix M is used for incremental updates, since the enumeration is along the x-axis."
        # This is confusing. If we are moving from beam j to j+1 (changing x), the step in object space is [1,0,0,0].
        # The change in image space is M * [1,0,0,0]^T = col0.
        # So yes, the entire beam B_y:(j+1, k) is obtained by adding col0 to the corresponding points in B_y:(j, k).
        
        for y in range(N):
            v_prev = transformed_volume[j-1][y][0]
            v_curr = add_vectors(v_prev, col0)
            transformed_volume[j][y][0] = v_curr
            
    # 4. Slice Phase: Transform remaining slices k=1 to N-1
    # Use slice k-1 to update slice k.
    # Update: add col2.
    for k in range(1, N):
        for j in range(N):
            for y in range(N):
                v_prev = transformed_volume[j][y][k-1]
                v_curr = add_vectors(v_prev, col2)
                transformed_volume[j][y][k] = v_curr

    return transformed_volume
```

## Subtopics

### Two-Pass Image Transformation
The original method by Drebin et al. It transforms the volume in two stages. It is less efficient and produces lower quality for general views compared to the three-pass method [S1, unofficial].

### Three-Pass Image Transformation
Generalized by Hanrahan. It decomposes the transformation into three passes (often corresponding to shears or specific axis alignments) to reduce resampling stages and improve efficiency and quality for general views [S1, unofficial].

### Warp-Space Affine Mixture
A sibling method that uses affine transformations to warp slices or volumes. It differs in how it handles the mixture of substances and the specific warping strategy [Link to sibling].

### Volume Rotation via Affine Transform
A sibling method focusing on rotation. It may use different decomposition strategies for the rotation matrix [Link to sibling].

## Operating characteristics
*   **Time Complexity:** Standard transformation requires $16N^3$ multiplications and $12N^3$ additions. Using the incremental update scheme with spatial coherency, the complexity is reduced to $3N^3$ additions (after one initial matrix-vector multiply) [S2, unofficial].
*   **Memory Complexity:** Requires storage for the input volume and the transformed intermediate volume, i.e., $O(N^3)$ [S2, unofficial].
*   **Parallelism:** The method exhibits high parallelism.
    *   **Level 1:** Slices can be transformed independently, allowing distribution to multiple processors with no communication [S2, unofficial].
    *   **Level 2:** Within slices, beams can be processed using vector processing or pipelining due to coherency [S2, unofficial].
*   **Strengths:**
    *   Linear scalability on parallel machines [S2, unofficial].
    *   Simple implementation (feed-forward rotate-and-draw paradigm) [S2, unofficial].
    *   Efficient on vector processors due to regular memory access patterns [S2, unofficial].
*   **Limitations:**
    *   Artifacts from direct transformation require post-processing (interpolation/reconstruction) [S2, unofficial].
    *   Quality for general views is lower than the three-pass variant [S1, unofficial].
    *   Not suitable for non-rectilinear grids without modification [S2, unofficial].
*   **When to Use:** When rendering on parallel or vector architectures where linear scalability is critical, and the view angles are not extremely general (or the three-pass variant is used) [S2, unofficial].
*   **When Not to Use:** When high-quality anti-aliased results are required without post-processing, or for non-uniform grids [S2, unofficial].

## Background
*   **Origin:** The method was introduced by Drebin, Cole, and Hurst in 1988 [S1, unofficial]. Hanrahan generalized it to a three-pass method in 1990 [S1, unofficial].
*   **Development:** The algorithm was developed to address the inefficiency of direct matrix multiplication for large volumes. It was specifically designed to uncover inherent parallelism schemes for moderately parallel MIMD machines [S2, unofficial].
*   **Usage:** It is a foundational feed-forward volume rendering technique, often cited in the context of parallel volume rendering and shear-warp factorization [S2, unofficial].

## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19930023133: Feed-forward volume rendering algorithm for moderately parallel MIMD machines, https://archive.org/details/NASA_NTRS_Archive_19930023133, fetched 2026-10-26 (unofficial)
