# 4D Hybrid Planar-Grid Representation
Also called: 4D Hybrid Planar-Grid

## Overview
The 4D Hybrid Planar-Grid Representation is a neural field method for large-scale dynamic scene reconstruction that combines coarse-resolution multi-planar features with high-resolution hash grids to balance efficiency and accuracy [S1]. It addresses the scalability limits of dense 3D grids and the visual artifacts of pure hash grids by decomposing the 4D space into static and dynamic components, leveraging the smoothness of planar factorization for global structure and the precision of hash mapping for fine details [S1]. It builds on [K-planes](3317-k-planes.md) by extending planar factorization to 4D and incorporating hash grids, distinguishing itself from [TriPlane NeRF](6932-triplane-nerf.md) by explicitly separating temporal dynamics into dedicated dynamic planes and hash structures for large-scale driving scenarios [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 4D coordinates $(x, y, z, t)$ representing spatial position and time.
*   **Output:** Feature vector $\mathbf{f}$ used to predict scene properties (e.g., density, color, LiDAR intensity).

**Data Structures:**
The representation consists of two main components: Planar Features and Hash Features.

1.  **Planar Feature Grid ($\mathbf{V}$):**
    *   Layout: Stores features for 6 orthogonal planes: static $(xy, xz, yz)$ and dynamic $(xt, yt, zt)$.
    *   Dimensions: Spatial resolution $M$ for static planes; spatial resolution $M$ and temporal resolution $H$ for dynamic planes.
    *   Size: $\mathbf{V} \in \mathbb{R}^{(3 M^{2} + 3 M H) C}$, where $C$ is the number of channels [S1].
    *   Interpolation: Bilinear interpolation [S1].

2.  **Hash Feature Grid ($\mathbf{G}$):**
    *   Layout: Stores features for static 3D space and dynamic 3D subspaces involving time.
    *   Dimensions: Static component uses resolution $M$; dynamic components use resolution $M$ (spatial) and $H$ (temporal).
    *   Size: $\mathbf{G} \in \mathbb{R}^{(M^{3} + 3 M^{2} H) C}$ [S1].
    *   Compression: The dense grid is compressed via hash mapping to reduce parameters [S1].
    *   Interpolation: Trilinear interpolation [S1].

**Parameters:**
*   $M$: Spatial resolution for planar features and base hash grid resolution.
*   $H$: Temporal resolution for dynamic planar features and dynamic hash grids.
*   $C$: Number of feature channels.
*   Typical values: Not given in the sources.

**Algorithm Steps:**
1.  **Project to Planar Features:**
    *   Project 4D coordinates $(x, y, z, t)$ into the 6 orthogonal planes: $xy, xz, yz$ (static) and $xt, yt, zt$ (dynamic).
    *   Retrieve features from $\mathbf{V}$ using bilinear interpolation.
    *   Compute static planar feature $\mathbf{f}_{planar}^{static}$ by combining $xy, xz, yz$ features.
    *   Compute dynamic planar feature $\mathbf{f}_{planar}^{dynamic}$ by combining $xt, yt, zt$ features.
    *   Apply Hadamard product ($\odot$) to multiply static features together and dynamic features together [S1].
    *   Concatenate multiscale features in a coarse-to-fine manner [S1].

2.  **Project to Hash Features:**
    *   Project 4D coordinates into static 3D hash grid ($xyz$) and dynamic 3D hash grids ($xyt, xzt, yzt$).
    *   Retrieve features from $\mathbf{G}$ using trilinear interpolation.
    *   Compute static hash feature $\mathbf{f}_{hash}^{static}$ from the $xyz$ grid.
    *   Compute dynamic hash feature $\mathbf{f}_{hash}^{dynamic}$ from the $xyt, xzt, yzt$ grids.
    *   Apply Hadamard product to multiply dynamic hash features [S1].

3.  **Combine Features:**
    *   Concatenate or sum the planar and hash features to form the final feature vector $\mathbf{f}$ [S1].

**Formulas:**
The planar feature is defined as [S1]:
$$
\mathbf {f}_\mathrm {planar} = {\mathcal S}\left (\mathbf {V}, \left (x,y,z,t\right )\right ), \quad \mathbf {V} \in \mathbb {R}^{(3 M^{2} + 3 M H) C}
$$
where $\mathcal{S}$ projects coordinates into planes and interpolates bilinearly.

The hash feature is defined as [S1]:
$$
\mathbf {f}_\mathrm {hash} = {\mathcal S}\left (\mathbf {G}, \left (x, y, z, t\right )\right ), \quad \mathbf {G} \in \mathbb {R}^{(M^{3} + 3 M^{2} H) C}
$$
where $\mathcal{S}$ projects coordinates into hash grids and interpolates trilinearly.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_4d_hybrid_features(x, y, z, t, V, G, M, H, C):
    """
    Retrieves hybrid planar-hash features for a 4D coordinate.
    
    Args:
        x, y, z, t: Float coordinates in [0, 1] or normalized space.
        V: Planar feature grid tensor of shape (3*M^2 + 3*M*H, C).
           Layout: [xy, xz, yz, xt, yt, zt] planes.
        G: Hash feature grid (accessed via hash map).
           Layout: Static xyz grid and dynamic xyt, xzt, yzt grids.
        M: Spatial resolution.
        H: Temporal resolution.
        C: Channel dimension.
        
    Returns:
        f: Combined feature vector.
    """
    # 1. Planar Features
    # Static planes: xy, xz, yz
    f_xy = bilinear_interpolate(V[0], x, y)
    f_xz = bilinear_interpolate(V[1], x, z)
    f_yz = bilinear_interpolate(V[2], y, z)
    
    # Dynamic planes: xt, yt, zt
    f_xt = bilinear_interpolate(V[3], x, t)
    f_yt = bilinear_interpolate(V[4], y, t)
    f_zt = bilinear_interpolate(V[5], z, t)
    
    # Hadamard product for static and dynamic components
    f_planar_static = f_xy * f_xz * f_yz
    f_planar_dynamic = f_xt * f_yt * f_zt
    
    # Concatenate multiscale features (coarse-to-fine)
    # Note: Specific multiscale implementation details are not provided in sources [S1]
    f_planar = concatenate(f_planar_static, f_planar_dynamic)

    # 2. Hash Features
    # Static 3D grid: xyz
    f_hash_static = trilinear_interpolate(G.static_xyz, x, y, z)
    
    # Dynamic 3D grids: xyt, xzt, yzt
    f_hash_xt = trilinear_interpolate(G.dynamic_xt, x, t) # Assuming 2D hash grid for xt? 
    # Source says "dynamic 3D hash grids (xyt, xzt, yzt)" and "trilinear interpolation"
    # This implies the hash grids store 3D data, but the input is projected.
    # Standard hash grid lookup for 3D subspace:
    f_hash_xt = trilinear_interpolate(G.dynamic_xt, x, y, t) # Wait, xt plane is 2D. 
    # Source text: "projected into static (xyz) and dynamic (xyt, xzt, yzt) multi-level hash grids"
    # This implies the hash grids are 3D structures indexed by 3 coordinates.
    # For xyt, we use x, y, t. For xzt, we use x, z, t. For yzt, we use y, z, t.
    
    f_hash_xt = trilinear_interpolate(G.dynamic_xt, x, y, t)
    f_hash_xz = trilinear_interpolate(G.dynamic_xz, x, z, t) # Assuming G has these keys
    f_hash_yz = trilinear_interpolate(G.dynamic_yz, y, z, t)
    
    # Hadamard product for dynamic hash features
    f_hash_dynamic = f_hash_xt * f_hash_xz * f_hash_yz
    
    # Concatenate hash features
    f_hash = concatenate(f_hash_static, f_hash_dynamic)

    # 3. Combine
    # Source does not specify exact combination (sum/concat). 
    # Assuming concatenation as is common in hybrid representations.
    f = concatenate(f_planar, f_hash)
    
    return f

def bilinear_interpolate(grid, u, v):
    # Standard bilinear interpolation
    # grid shape: (H, W, C)
    # u, v in [0, 1]
    pass

def trilinear_interpolate(grid, u, v, w):
    # Standard trilinear interpolation
    # grid shape: (D, H, W, C)
    # u, v, w in [0, 1]
    pass
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Pure hash grids suffer from collisions, but the sparsity of LiDAR point clouds in large-scale scenes substantially avoids adverse effects [S1].
*   **Visual Artifacts:** Pure hash grid representation suffers from visual artifacts and noisy reconstruction results, impeding accurate object geometry construction [S1]. The hybrid approach mitigates this by using planar features for smooth global representation.
*   **Scalability:** Dense 3D grids (e.g., TiNeuVox) are unscalable for large-scale scenarios due to cubic complexity growth [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The planar component scales with $O(M^2 + MH)$ per channel, and the hash component scales with $O(M^3 + M^2H)$ per channel, but the hash grid is compressed, reducing effective memory usage compared to dense grids [S1].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Efficient and effective reconstruction for large-scale dynamic driving scenarios; high accuracy and efficiency by combining smooth planar features with detailed hash grids; reduces parameter quantities compared to dense 3D grids [S1].
*   **Limitations:** Pure hash grids alone suffer from artifacts; requires careful balancing of planar and hash resolutions [S1].
*   **When to Use:** Large-scale dynamic scenes, particularly LiDAR point cloud reconstruction, where both global smoothness and fine detail are required [S1].
*   **When Not to Use:** Small-scale static scenes where simpler representations like [TriPlane NeRF](6932-triplane-nerf.md) might suffice, or when computational resources for hash lookups are extremely limited.

## Background
The 4D Hybrid Planar-Grid Representation was introduced in the paper "LiDAR4D: Dynamic Neural Fields for Novel Space-Time View LiDAR Synthesis" [S1]. It was developed to address the limitations of existing neural fields for large-scale dynamic driving scenarios, specifically the unscalability of dense 3D grids and the artifacts of pure hash grids. The method builds on the planar factorization concept from K-planes [S1] and incorporates hash grids from Instant-NGP [S1] to achieve a hybrid representation. It is used for novel space-time view LiDAR synthesis, modeling geometric depth, reflection intensity, and ray-drop probability of LiDAR point clouds [S1].

## Sources
[S1] LiDAR4D: Dynamic Neural Fields for Novel Space-Time View LiDAR Synthesis, https://doi.org/10.1109/cvpr52733.2024.00492, fetched 2026-10-30
