# Texture Swapping

Also called: Texture transfer

## Overview
Texture swapping transfers appearance latent codes from a source mesh area to a target mesh area within trained neural mesh representations [S2, unofficial]. It solves the problem of reusing textures between objects with slightly different geometries by performing non-rigid 3D alignment of anchor nodes [S2, unofficial]. Unlike [Texture Painting](12730-texture-painting.md), which modifies appearance directly on a single model, swapping operates between two distinct trained models or areas, leveraging pre-blended latent codes for better 3D consistency [S2, unofficial].

## How it works
**Inputs:**
*   Source mesh area: A set of vertices $V_S$ and their associated latent codes $C_S$ from a trained source model [S1].
*   Target mesh area: A set of vertices $V_T$ and their associated latent codes $C_T$ from a trained target model [S1].
*   Correspondences: A set of $K$ point pairs $(p_i^S, p_i^T)$ where $p_i^S \in V_S$ and $p_i^T \in V_T$, with $K \in [4, 9]$ [S1].

**Outputs:**
*   Updated latent codes $C_T'$ for the target vertices, allowing the target area to display the source texture [S2, unofficial].

**Data Structures:**
*   Mesh vertices: 3D coordinates $(x, y, z)$ [G].
*   Latent codes: Vectors $c$ stored at anchor nodes (mesh vertices) [Parent].

**Steps:**
1.  **Selection:** Select source and target areas by choosing mesh vertices in a 3D editor (e.g., Blender) [S1].
2.  **Annotation:** Annotate $K$ point correspondences between the source and target areas [S1].
3.  **Initial Alignment (Umeyama):** Compute a scaled rigid transformation $T_{scale}$ that aligns the source correspondences to the target correspondences using the Umeyama algorithm [S1][S2, unofficial]. This minimizes the least-squares error between the point sets [G].
4.  **Non-Rigid Deformation (ARAP):** Calculate the residual transformation for each corresponding point pair after applying $T_{scale}$. Use As-Rigid-As-Possible (ARAP) deformation to deform the entire source mesh area $V_S$ to match the target shape, producing deformed source vertices $V_S'$ [S1][S2, unofficial].
5.  **Code Interpolation:** For each target vertex $v_j \in V_T$, identify the 4 nearest neighbors in the deformed source vertices $V_S'$ [S1].
6.  **Code Assignment:** Compute the new latent code $c_j'$ for $v_j$ using inverse distance weighting (IDW) of the latent codes from the 4 nearest deformed source vertices [S1].
    $$ c_j' = \frac{\sum_{k=1}^{4} w_k c_k'}{\sum_{k=1}^{4} w_k} $$
    where $c_k'$ is the latent code of the $k$-th nearest deformed source vertex, and $w_k = \frac{1}{d_k}$ is the weight based on the Euclidean distance $d_k$ between $v_j$ and the $k$-th source vertex [S1].
7.  **Decoding:** Decode the updated latent codes $C_T'$ using the source decoder to render the swapped texture, as the pre-blended method is chosen for better 3D consistency [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_swap(source_verts, target_verts, source_codes, target_codes, src_indices, tgt_indices, correspondences):
    # correspondences: list of (src_idx, tgt_idx)
    # Step 3: Scaled rigid alignment (Umeyama)
    src_pts = [source_verts[i] for i, _ in correspondences]
    tgt_pts = [target_verts[i] for _, i in correspondences]
    scale, rot, trans = umeyama(src_pts, tgt_pts)
    
    # Apply rigid transform to all source vertices
    deformed_verts = []
    for v in source_verts:
        r_v = [sum(r * c for r, c in zip(row, v)) for row in rot]
        deformed_verts.append([r_v[j] + trans[j] for j in range(3)])
    
    # Step 4: Non-rigid deformation (ARAP)
    residuals = []
    for src_i, tgt_i in correspondences:
        t_src = deformed_verts[src_i]
        tgt_v = target_verts[tgt_i]
        residuals.append([tgt_v[j] - t_src[j] for j in range(3)])
    deformed_verts = arap(deformed_verts, source_verts, residuals, correspondences)
    
    # Step 5 & 6: Code interpolation
    updated_codes = []
    for i, tgt_v in enumerate(target_verts):
        neighbors = k_nearest(tgt_v, deformed_verts, k=4)
        if not neighbors:
            updated_codes.append(target_codes[i])
            continue
        weights = []
        codes = []
        for n_idx in neighbors:
            d = dist(tgt_v, deformed_verts[n_idx])
            weights.append(1.0 / (d + 1e-6))
            codes.append(source_codes[n_idx])
        w_sum = sum(weights)
        new_c = [0.0] * len(source_codes[0])
        for w, c in zip(weights, codes):
            for d in range(len(c)):
                new_c[d] += w * c[d]
        for d in range(len(new_c)):
            new_c[d] /= w_sum
        updated_codes.append(new_c)
    return updated_codes

def dist(a, b): return sum((x - y) ** 2 for x, y in zip(a, b)) ** 0.5

def k_nearest(p, pts, k):
    return [i for _, i in sorted([(dist(p, pt), i) for i, pt in enumerate(pts)])[:k]]

def umeyama(src, tgt):
    # Standard closed-form solution for scale, rotation, translation
    # Minimizes least-squares error between point sets
    # Returns (scale, rotation_matrix, translation_vector)
    pass

def arap(current, original, residuals, correspondences):
    # Standard ARAP deformation solving sparse linear system
    # Distributes residuals to preserve local rigidity
    # Returns updated vertex positions
    pass
```

**Parameters:**
*   Number of correspondences: 4 to 9 points [S1].
*   Nearest neighbors for interpolation: 4 [S1].
*   Weighting scheme: Inverse distance weighting [S1].

**Edge Cases:**
*   Precision: Ensure distance calculations use sufficient precision to avoid division by zero in IDW [G].
*   Boundaries: The method assumes the source and target areas can be reasonably aligned [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the ARAP deformation and nearest neighbor search. ARAP is typically solved iteratively. Nearest neighbor search is $O(N \log N)$ or $O(N)$ with spatial indexing [G].
*   **Memory Complexity:** Requires storing latent codes for all vertices in the source and target areas [G].
*   **Strengths:** Provides better 3D consistency than pose-dependent inference methods by using pre-blended codes [S2, unofficial]. Allows local editing without global shape parameter changes [S2, unofficial].
*   **Limitations:** Requires manual annotation of correspondences (though automation is possible) [S1].

## Background
Texture swapping was introduced in the context of Unified Volumetric Avatars (UVA) and NeuMesh for geometry and texture editing [S1][S2, unofficial]. The method leverages Umeyama alignment for initial rigid transformation and ARAP for non-rigid deformation [S1].

## Sources
[S1] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S2] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
