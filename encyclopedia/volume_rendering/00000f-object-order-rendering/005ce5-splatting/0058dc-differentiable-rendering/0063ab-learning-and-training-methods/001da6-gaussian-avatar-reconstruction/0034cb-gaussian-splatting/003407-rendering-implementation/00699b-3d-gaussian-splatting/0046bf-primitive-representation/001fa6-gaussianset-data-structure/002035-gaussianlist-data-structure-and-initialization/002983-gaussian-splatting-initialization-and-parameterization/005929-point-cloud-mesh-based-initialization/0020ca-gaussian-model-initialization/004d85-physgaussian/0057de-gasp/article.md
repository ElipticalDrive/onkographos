# GASP

Also called: GASP (Gaussian Splatting for Physic-Based Simulations), GASP pipeline

## Overview
GASP is a pipeline that integrates 3D Gaussian Splatting with black-box physics engines by converting flat Gaussian components into triangle faces, simulating the vertices using the Material Point Method (MPM), and reconstructing Gaussians from the deformed triangles [S1, unofficial]. It solves the problem of simulating deformable objects in Gaussian representations without requiring modifications to the physics engine or complex mesh extraction strategies like marching cubes [S1, unofficial]. Unlike PhysGaussian, which approximates deformation via first-order linear transformations, GASP operates directly on the geometry of flat Gaussians using the GaMeS parameterization, ensuring stable simulations and avoiding artifacts associated with treating points as discrete entities [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A collection of flat Gaussians $G = \{(N(m_i, R_i, S_i), \sigma_i, c_i)\}_{i=1}^p$, where $m_i$ is the mean, $R_i$ is the rotation matrix, $S_i = \text{diag}(s_{i1}, s_{i2}, s_{i3})$ is the scaling matrix with $s_{i1} = \epsilon$ (making the Gaussian effectively 2D), $\sigma_i$ is opacity, and $c_i$ is color [S1, unofficial].
*   **Output:** A time-evolved collection of Gaussians $G_t = \{(N(m'_i, R'_i, S'_i), \sigma_i, c_i)\}_{i=1}^p$ reflecting the physics simulation [S1, unofficial].

**Data Structures:**
*   **Gaussian:** Defined by mean $m \in \mathbb{R}^3$, rotation $R \in \mathbb{R}^{3 \times 3}$, scaling $S \in \mathbb{R}^{3 \times 3}$ (diagonal), opacity $\sigma \in \mathbb{R}$, and color $c \in \mathbb{R}^3$ [S1, unofficial].
*   **Triangle Face (GaMeS):** A triplet of vertices $V = [v_1, v_2, v_3]$ representing a flat Gaussian [S1, unofficial].
*   **Triangle Soup:** A collection of vertices $V$ and edges $E$ derived from the Gaussians [S1, unofficial].

**Parameters:**
*   $\epsilon$: A small constant such that $s_1 = \epsilon$ [S1, unofficial].
*   $\alpha$: A hyperparameter for the correction algorithm, controlling the threshold for scaling correction [S1, unofficial].

**Steps:**

1.  **Initialization (GaMeS Conversion):**
    Convert each flat Gaussian $N(m, R, S)$ into a triangle face $V = [v_1, v_2, v_3]$ using the mapping $T(\cdot)$ [S1, unofficial]:
    $$v_1 = m$$
    $$v_2 = m + s_2 r_2$$
    $$v_3 = m + s_3 r_3$$
    where $r_2$ and $r_3$ are the second and third columns of the rotation matrix $R$ [S1, unofficial]. This creates a triangle soup $(V, E)$ [S1, unofficial].

2.  **Physics Simulation:**
    Apply the physics engine (e.g., MPM) to the vertices $V$ individually to obtain deformed vertices $V_t$ [S1, unofficial]:
    $$V_t = \{\phi(v, t) : v \in V\}$$
    where $\phi(\cdot, t)$ is the deformation map provided by the black-box physics engine (such as Genesis, Blender, or Taichi Elements) [S1, unofficial]. The edges $E$ are preserved to maintain the triangle structure [S1, unofficial].

3.  **Reconstruction (Inverse GaMeS):**
    Recover the Gaussian parameters from the deformed triangle vertices $V_t = [v'_1, v'_2, v'_3]$ using the inverse mapping $T^{-1}(\cdot)$ [S1, unofficial]:
    *   **Mean:** $\hat{m} = v'_1$ [S1, unofficial].
    *   **Rotation Axis 1:** $\hat{r}_1 = \frac{(v'_2 - v'_1) \times (v'_3 - v'_1)}{\|(v'_2 - v'_1) \times (v'_3 - v'_1)\|}$ (normalized face normal) [S1, unofficial].
    *   **Rotation Axis 2:** $\hat{r}_2 = \frac{v'_2 - v'_1}{\|v'_2 - v'_1\|}$ (normalized edge vector) [S1, unofficial].
    *   **Rotation Axis 3:** $\hat{r}_3 = \text{orth}(v'_3 - v'_1; \hat{r}_1, \hat{r}_2)$ (Gram-Schmidt orthogonalization) [S1, unofficial].
    *   **Scaling:**
        $$\hat{s}_1 = \epsilon$$
        $$\hat{s}_2 = \|v'_2 - v'_1\|$$
        $$\hat{s}_3 = \langle v'_3 - v'_1, \hat{r}_3 \rangle$$
    The new rotation matrix is $\hat{R} = [\hat{r}_1, \hat{r}_2, \hat{r}_3]$ and scaling is $\hat{S} = \text{diag}(\hat{s}_1, \hat{s}_2, \hat{s}_3)$ [S1, unofficial].

4.  **Correction Algorithm:**
    To mitigate artifacts from independent vertex processing, check if the distance between transformed nodes exceeds a threshold relative to original nodes [S1, unofficial]. For each Gaussian face with original vertices $V = [v_1, v_2, v_3]$ and deformed vertices $V_t = [v'_1, v'_2, v'_3]$:
    If $\|v'_1 - v'_i\| > \alpha \|v_1 - v_i\|$ for $i \in \{2, 3\}$, then correct the scaling parameter $\hat{s}_i$ to $\alpha \cdot s_i$ (where $s_i$ is the original scaling parameter before simulation) [S1, unofficial]. Note: The source states "si should be identical to that of the original value times α" [S1, unofficial].

5.  **Hierarchical Simulation (Optional):**
    *   Use BIRCH clustering to group Gaussians into CoreGaussians and SubGaussians [S1, unofficial].
    *   Simulate only the CoreGaussians using the steps above.
    *   Transfer the movement pattern (deformation) from CoreGaussians to their associated SubGaussians [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gasp_pipeline(gaussians, physics_engine, alpha=1.5):
    """
    gaussians: List of dicts with keys 'm', 'R', 'S', 'sigma', 'c'
    physics_engine: Object with method 'simulate(points)' returning deformed points
    alpha: Hyperparameter for correction
    """
    # Step 1: Convert Gaussians to Triangle Soup
    triangles = []
    for g in gaussians:
        m = g['m']
        R = g['R']
        S = g['S']
        s2 = S[1, 1]
        s3 = S[2, 2]
        r2 = R[:, 1]
        r3 = R[:, 2]
        
        v1 = m
        v2 = m + s2 * r2
        v3 = m + s3 * r3
        
        triangles.append({
            'v1': v1, 'v2': v2, 'v3': v3,
            'original_s2': s2, 'original_s3': s3,
            'sigma': g['sigma'], 'c': g['c']
        })
    
    # Extract all unique vertices for physics engine
    # Note: In a real implementation, vertices shared by multiple triangles 
    # must be handled carefully (e.g., by merging indices). 
    # The source describes treating points individually, implying a point cloud approach.
    # We assume physics_engine takes a list of points and returns deformed points.
    # To preserve topology, we might need to pass the full set of vertices.
    
    all_vertices = []
    for t in triangles:
        all_vertices.append(t['v1'])
        all_vertices.append(t['v2'])
        all_vertices.append(t['v3'])
        
    # Physics simulation
    deformed_vertices = physics_engine.simulate(all_vertices)
    
    # Step 2 & 3 & 4: Reconstruct Gaussians
    new_gaussians = []
    tri_idx = 0
    vert_idx = 0
    
    # Re-associate deformed vertices to triangles
    # Assuming the order of deformed_vertices matches the order of all_vertices
    for t in triangles:
        v1 = deformed_vertices[vert_idx]
        v2 = deformed_vertices[vert_idx + 1]
        v3 = deformed_vertices[vert_idx + 2]
        vert_idx += 3
        
        # Inverse GaMeS
        m_hat = v1
        
        # Rotation axes
        edge1 = v2 - v1
        edge2 = v3 - v1
        
        r1_hat = np.cross(edge1, edge2)
        norm_r1 = np.linalg.norm(r1_hat)
        if norm_r1 < 1e-8:
            # Degenerate triangle, fallback or skip [G]
            r1_hat = np.array([0, 0, 1])
        else:
            r1_hat = r1_hat / norm_r1
            
        r2_hat = edge1 / np.linalg.norm(edge1)
        
        # Gram-Schmidt for r3
        r3_hat = edge2 - np.dot(edge2, r1_hat) * r1_hat - np.dot(edge2, r2_hat) * r2_hat
        norm_r3 = np.linalg.norm(r3_hat)
        if norm_r3 < 1e-8:
            r3_hat = np.cross(r1_hat, r2_hat) # Fallback [G]
        else:
            r3_hat = r3_hat / norm_r3
            
        R_hat = np.column_stack((r1_hat, r2_hat, r3_hat))
        
        # Scaling
        s2_hat = np.linalg.norm(edge1)
        s3_hat = np.dot(edge2, r3_hat)
        
        # Correction Algorithm
        # Compare deformed distances to original distances
        orig_v1 = t['v1']
        orig_v2 = t['v2']
        orig_v3 = t['v3']
        orig_s2 = t['original_s2']
        orig_s3 = t['original_s3']
        
        dist_v1_v2_orig = np.linalg.norm(orig_v1 - orig_v2)
        dist_v1_v3_orig = np.linalg.norm(orig_v1 - orig_v3)
        
        dist_v1_v2_deform = np.linalg.norm(v1 - v2)
        dist_v1_v3_deform = np.linalg.norm(v1 - v3)
        
        # Correction logic based on source: if deformed dist > alpha * original dist
        # Then scale parameter is corrected to original value * alpha
        # Note: The source says "si should be identical to that of the original value times alpha"
        # This implies capping the growth.
        
        if dist_v1_v2_deform > alpha * dist_v1_v2_orig:
            s2_hat = alpha * orig_s2
            
        if dist_v1_v3_deform > alpha * dist_v1_v3_orig:
            s3_hat = alpha * orig_s3
            
        # s1 is always epsilon
        s1_hat = 1e-6 # epsilon
        
        new_gaussians.append({
            'm': m_hat,
            'R': R_hat,
            'S': np.diag([s1_hat, s2_hat, s3_hat]),
            'sigma': t['sigma'],
            'c': t['c']
        })
        
    return new_gaussians
```

## Subtopics

### GaMeS (Gaussian Meshes)
A parameterization of flat Gaussians as triangle faces. It sets one scaling axis to $\epsilon$, allowing Gaussians to be represented as 2D triangles in 3D space. It is chosen when stable, triangle-based physics simulation is required for Gaussian splats. [Link](6003-games.md)

### Material Point Method (MPM)
A particle-based physics simulation method used in GASP to compute vertex trajectories. It is chosen for its ability to handle large deformations and complex material behaviors (cloth, sand, etc.) as a black-box engine. [Link](../../volume-rendering/differentiable-and-inverse-volume-rendering/radiance-field-optimization/3d-gaussian-splatting/mpm.md)

### PhysGaussian
A method that adapts Newtonian dynamics to 3D Gaussians using first-order approximation of deformation maps. GASP differs by using GaMeS triangles and black-box physics, avoiding the need for engine modifications or linear approximations. [Link](6002-physgaussian.md)

## Operating characteristics
*   **Time Complexity:** The conversion to triangles and back is linear with respect to the number of Gaussians $p$. The physics simulation cost depends on the engine (e.g., MPM complexity). Hierarchical simulation reduces the number of points simulated, accelerating computation [S1, unofficial].
*   **Memory Complexity:** Linear in the number of Gaussians, storing vertices and edges for the triangle soup [S1, unofficial].
*   **Performance:** The pipeline is described as "notably rapid" and does not increase rendering costs compared to standard GS [S1, unofficial].
*   **Strengths:** Compatible with any black-box physics engine (Genesis, Blender, Taichi); avoids artifacts from discrete point processing; avoids complex mesh extraction (marching cubes) [S1, unofficial].
*   **Limitations:** Relies on the quality of the black-box physics engine; artifacts can still occur if the correction threshold $\alpha$ is not tuned; hierarchical simulation requires clustering overhead [S1, unofficial].
*   **When to use:** When simulating deformable objects (cloth, jelly, sand) with Gaussian Splatting and you want to leverage existing physics engines without modifying them.
*   **When not to use:** When high-fidelity continuous deformation fields are required beyond the triangle approximation, or when the physics engine cannot handle point clouds efficiently.

## Background
GASP was introduced in the paper "GASP: Gaussian Splatting for Physic-Based Simulations" by Waczja et al. (2024) [S1, unofficial]. It builds upon the GaMeS parameterization of flat Gaussians [S1, unofficial]. The method was demonstrated using the Genesis engine, Blender software, and the Taichi Elements library [S1, unofficial]. It addresses limitations in previous methods like PhysGaussian (which requires engine modifications) and mesh-based approaches (which suffer from extraction artifacts) [S1, unofficial].

## Sources
[S1] GASP: Gaussian Splatting for Physic-Based Simulations, https://doi.org/10.48550/arXiv.2409.05819, fetched 2026-10-30 (unofficial)
