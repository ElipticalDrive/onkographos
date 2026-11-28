# Voting Marching Cubes

Also called: Noise-robust MC, Volumetric Marching Cubes

## Overview
Voting Marching Cubes (VMC) is a variant of the standard Marching Cubes algorithm that mitigates surface reconstruction errors caused by noise in silhouette images [S1]. While standard Marching Cubes requires unanimous agreement across all views to classify a voxel corner as inside, VMC employs a voting threshold to tolerate individual classification errors, thereby reducing the occurrence of lost octants where noise prevents surface generation [S1].

## How it works
**Inputs:**
- $n$: Total number of silhouette images.
- $\theta_i$: Rotational angle of the $i$-th viewpoint.
- $P(\theta_i)$: Projection matrix for the $i$-th viewpoint.
- $S_i$: Binary silhouette mask for the $i$-th image ($1$ if inside object, $0$ if outside).
- $\vec{x}_{3d}$: 3D vertex position in the voxel grid.
- $v_{th}$: Voting threshold parameter.

**Outputs:**
- Boolean classification of $\vec{x}_{3d}$ as "inside" or "outside" for the Marching Cubes lookup table [S1].

**Data Structures:**
- Standard voxel grid and binary silhouette masks [S1].

**Steps:**
1. Compute the cost function $c_i(\theta_i, \vec{x}_{3d})$ for each silhouette $i \in \{0, \dots, n-1\}$:
   $$c_i(\theta_i, \vec{x}_{3d}) = \begin{cases} 1 & \text{if } P(\theta_i)\vec{x}_{3d} \in S_i \\ 0 & \text{otherwise} \end{cases}$$
   This checks if the projection of the 3D vertex falls within the object's silhouette in the $i$-th image [S1].
2. Sum the cost values across all $n$ silhouette images:
   $$C = \sum_{i=0}^{n-1} c_i(\theta_i, \vec{x}_{3d})$$
   (Note: Source notation uses $\sum_{i=0}^{n}$, but $n$ denotes the total count of images, so indices run $0$ to $n-1$ [S1].)
3. Subtract the voting threshold $v_{th}$ from the sum:
   $$R = C - v_{th}$$
4. Classify the vertex: If $R \le 0$, classify $\vec{x}_{3d}$ as inside; otherwise, classify as outside [S1].

**Formula:**
$$\sum_{i=0}^{n-1} c_i(\theta_i, \vec{x}_{3d}) - v_{th} \le 0, \quad 0 \le v_{th} < n$$
where $n$ is the total number of silhouette images [S1].

**Parameters:**
- $v_{th}$: Voting threshold for outside classification. Must satisfy $0 \le v_{th} < n$ [S1]. Value is not specified and requires careful selection [S1].
- Typical values: For 10% noise, the optimal threshold is approximately 90% of $n$ [S1]. For 15% noise, a smaller threshold minimizes lost octants [S1].

**Edge Cases and Pitfalls:**
- Output quality varies with the threshold level even for convex objects, making threshold selection non-trivial [S1].
- If $v_{th}$ is too small, an intersection octant may have 8 inside corners. Standard MC then treats it as an inside octant and generates no surface, resulting in a lost octant [S1].
- Standard MC loses a significant number of surface patches with 10% noise, whereas VMC reduces this lost octant ratio with an appropriate threshold [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voting_marching_cubes_classify(vertex_3d, silhouettes, v_th):
    """
    Classifies a 3D vertex using Voting Marching Cubes logic.
    
    Args:
        vertex_3d: tuple or list of 3 floats (x, y, z)
        silhouettes: list of n dictionaries, each containing:
            'proj_matrix': 3x3 or 4x4 numpy array (projection matrix P)
            'mask': 2D numpy array (binary silhouette S)
        v_th: int, voting threshold (0 <= v_th < n)
        
    Returns:
        bool: True if classified inside, False if outside
    """
    n = len(silhouettes)
    inside_votes = 0
    
    for i in range(n):
        # Project 3D vertex to 2D using the i-th camera matrix
        # Assuming homogeneous coordinates: proj_2d = P @ [x, y, z, 1]
        proj_2d = silhouettes[i]['proj_matrix'] @ [vertex_3d[0], vertex_3d[1], vertex_3d[2], 1.0]
        # Normalize to image coordinates (divide by w)
        if proj_2d[2] != 0:
            u, v = proj_2d[0] / proj_2d[2], proj_2d[1] / proj_2d[2]
        else:
            u, v = 0.0, 0.0
            
        # Check if projection falls inside the binary silhouette mask
        # Mask dimensions: height x width
        h, w = silhouettes[i]['mask'].shape
        if 0 <= int(v) < h and 0 <= int(u) < w and silhouettes[i]['mask'][int(v), int(u)] == 1:
            inside_votes += 1
            
    # Revised classification condition: sum - v_th <= 0 => inside
    if inside_votes - v_th <= 0:
        return True  # inside
    else:
        return False # outside
```

## Operating characteristics
- **Time Complexity:** $O(n \cdot V)$, where $n$ is the number of silhouettes and $V$ is the number of voxel vertices [G]. Each vertex requires $n$ projection and mask lookup operations.
- **Memory Complexity:** $O(V + n \cdot S)$, where $S$ is the size of the silhouette masks [G].
- **Strengths:** Significantly reduces the lost octant ratio caused by noise in silhouette images compared to standard Marching Cubes [S1].
- **Limitations:** Reconstruction quality varies with the threshold level even for convex objects [S1]. Surface continuity may degrade compared to ideal Marching Cubes results, as surfaces may not always connect correctly [S1]. Selecting an optimal $v_{th}$ is difficult and non-trivial [S1].
- **When to use:** Reconstructing surfaces from noisy silhouette images where standard Marching Cubes produces excessive holes or unattached segments [S1].
- **When not to use:** When high precision and strict surface continuity are required with minimal noise, or when the optimal voting threshold cannot be determined [S1].

## Background
Voting Marching Cubes was introduced to address the noise sensitivity of standard Marching Cubes in silhouette-based volumetric reconstruction [S1]. The method modifies the vertex classification decision function to tolerate individual view errors by counting inside votes across multiple silhouettes rather than requiring unanimous agreement [S1]. It was described in the context of local hull-based surface construction techniques [S1].

## Sources
[S1] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
