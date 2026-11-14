# Unified Volumetric Avatar (UVA)

## Overview
Unified Volumetric Avatar (UVA) is a method for representing dynamic human avatars that enables local and independent editing of both geometry and texture while supporting novel view synthesis and novel pose rendering [S1, unofficial]. It distinguishes itself from pre-defined UV mapping methods, which only allow 3D-to-2D direction mapping and hinder local texture editing, by anchoring structured latent codes to a deformable mesh in canonical space [S1, unofficial]. Unlike two-stage distilling methods that require manual parameter tuning and are limited to static objects, UVA is end-to-end trainable and jointly optimized with the skinning field from scratch [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters (ray origin $\mathbf{o}$ and direction $\mathbf{d}$), target pose parameters $\theta_t$, and a deformable canonical mesh $M$ with anchor nodes.
*   **Outputs:** Final pixel color $\mathbf{c}(\mathbf{r})$ for the rendered ray $\mathbf{r}$.

**Data Structures:**
1.  **Deformable Canonical Mesh ($M$):** A mesh in canonical space with $N$ anchor nodes. Each node $i$ stores:
    *   Position $\mathbf{p}_i$ in canonical space.
    *   Geometry latent code $\mathbf{l}_{geo}^i \in \mathbb{R}^{d_{geo}}$.
    *   Appearance latent code $\mathbf{l}_{rgb}^i \in \mathbb{R}^{d_{rgb}}$.
    *   UV coordinates $(u_i, v_i)$ associated with the node.
2.  **Neural Deformation Field:** An MLP $\phi_{def}$ that predicts a non-rigid deformation offset $\Delta \mathbf{x}$ given a point in canonical space and pose parameters.
3.  **Shading MLP:** An MLP $\phi_{shade}$ that predicts a pose-dependent shading factor $s$.
4.  **Radiance MLP:** An MLP $\phi_{rad}$ that takes interpolated latent codes and a signed height indicator to predict volume density $\sigma$ and base color $\mathbf{c}_0$.

**Numbered Steps:**

1.  **Ray Sampling:** For a given camera pose and target pose $\theta_t$, sample points $\mathbf{x}_t$ along the ray $\mathbf{r}$ in the target (observation) space.

2.  **LBS Weight Calculation:** For each target point $\mathbf{x}_t$, determine its Linear Blending Skinning (LBS) weights $\mathbf{w}_b$ by interpolating the weights of its Top-$K$ nearest neighbors (NNs) on the mesh $M$ [S1, unofficial].
    *   Find the $K$ nearest mesh nodes to $\mathbf{x}_t$.
    *   Interpolate the pre-defined or learned LBS weights of these neighbors to obtain $\mathbf{w}_b$ for $\mathbf{x}_t$.

3.  **Canonical Space Transformation:** Transform the target point $\mathbf{x}_t$ to the canonical space $\mathbf{x}_c$ using the inverted LBS transformation and a neural deformation field [S1, unofficial].
    *   Apply the inverse LBS transformation using $\mathbf{w}_b$ and the skeleton motion defined by $\theta_t$ to get an initial canonical position.
    *   Refine this position using the neural deformation field: $\mathbf{x}_c = \text{LBS}^{-1}(\mathbf{x}_t, \mathbf{w}_b, \theta_t) + \phi_{def}(\mathbf{x}_c, \theta_t)$ [S1, unofficial]. Note: The exact composition of the inverse LBS and the deformation field offset $\Delta \phi$ is described as combining the NN motion field with the neural deformation field [S1, unofficial].

4.  **Latent Code Interpolation:** Calculate the latent codes for the canonical point $\mathbf{x}_c$ by interpolating the latent codes anchored on the $K$ nearest neighbors of $\mathbf{x}_c$ on the mesh $M$ [S1, unofficial].
    *   Find the $K$ nearest mesh nodes to $\mathbf{x}_c$.
    *   Interpolate $\mathbf{l}_{geo}$ and $\mathbf{l}_{rgb}$ from these nodes to get $\mathbf{l}_{geo}(\mathbf{x}_c)$ and $\mathbf{l}_{rgb}(\mathbf{x}_c)$.

5.  **Signed Height Indicator Calculation:** Compute the local signed height indicator $\tilde{\mathbf{h}}$ for $\mathbf{x}_c$ [S1, unofficial].
    *   $\tilde{\mathbf{h}} = [u, v, \bar{d}]$, where $(u, v)$ are the UV coordinates of the nearest mesh node to $\mathbf{x}_c$, and $\bar{d}$ is the point-to-surface distance relative to the anchored mesh [S1, unofficial]. This indicator addresses spatial ambiguity in code interpolation [S1, unofficial].

6.  **Radiance Field Prediction:** Use the Radiance MLP $\phi_{rad}$ to predict volume density $\sigma$ and base color $\mathbf{c}_0$ [S1, unofficial].
    *   Input to $\phi_{rad}$: Concatenation of $\mathbf{l}_{geo}(\mathbf{x}_c)$, $\mathbf{l}_{rgb}(\mathbf{x}_c)$, and $\tilde{\mathbf{h}}$.
    *   Output: $\sigma(\mathbf{x}_c)$ and $\mathbf{c}_0(\mathbf{x}_c)$.

7.  **Shading Modulation:** Predict a pose-dependent shading factor $s$ using the Shading MLP $\phi_{shade}$ [S1, unofficial].
    *   Input to $\phi_{shade}$: Pose parameters $\theta_t$ and potentially other context.
    *   Output: Shading factor $s$.
    *   Modulate base color: $\mathbf{c}_{mod} = \mathbf{c}_0 \otimes s$ (element-wise or appropriate modulation) [S1, unofficial].

8.  **Volume Rendering:** Integrate the modulated colors along the target ray $\mathbf{r}$ to produce the final color $\mathbf{c}(\mathbf{r})$ using standard volumetric rendering [S1, unofficial].
    *   $\mathbf{c}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_{mod}(\mathbf{x}_i)$, where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$.

**Parameters:**
*   **Top-K:** The number of nearest neighbors used for interpolating LBS weights and latent codes. Typical values are not explicitly given in the sources, but Table 1 in [S1, unofficial] lists "Local Tex." and "Local Geo." with values like 8, 4, 8, 4, 4, 8, 4, 8, 8, 4. It is unclear if these correspond to Top-K or other hyperparameters. The text explicitly mentions "Top-K nearest neighbors" [S1, unofficial].

**Pitfalls:**
*   **Spatial Ambiguity:** Using discrete latent codes defined on mesh nodes diffused via nearest-neighbor interpolation can lead to spatial query indistinguishability [S1, unofficial]. This is addressed by the signed height indicator $\tilde{\mathbf{h}}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def uva_render(ray_origin, ray_direction, target_pose, mesh, mesh_nodes, 
               lgeo_nodes, lrgb_nodes, uv_nodes, k_neighbors, 
               def_mlp, shade_mlp, rad_mlp, num_samples):
    """
    Renders a single ray using the UVA method.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, camera direction.
        target_pose: array, pose parameters theta_t.
        mesh: Mesh object with methods for finding nearest nodes and LBS weights.
        mesh_nodes: (N, 3) array, positions of mesh nodes in canonical space.
        lgeo_nodes: (N, d_geo) array, geometry latent codes for mesh nodes.
        lrgb_nodes: (N, d_rgb) array, appearance latent codes for mesh nodes.
        uv_nodes: (N, 2) array, UV coordinates for mesh nodes.
        k_neighbors: int, Top-K for interpolation.
        def_mlp: Neural deformation field MLP.
        shade_mlp: Shading MLP.
        rad_mlp: Radiance MLP.
        num_samples: int, number of samples along the ray.
        
    Returns:
        final_color: (3,) array, rendered color.
    """
    # Sample points along the ray in target space
    t = np.linspace(0, 1, num_samples) # Simplified sampling
    target_points = ray_origin[:, np.newaxis] + ray_direction[:, np.newaxis] * t[np.newaxis, :]
    target_points = target_points.T # (num_samples, 3)
    
    final_color = np.zeros(3)
    transmittance = 1.0
    
    for i in range(num_samples):
        x_t = target_points[i]
        
        # Step 2: Calculate LBS weights
        # Find K nearest neighbors on the mesh for x_t
        # Note: In practice, this might be done in target space or mapped first.
        # The source says "interpolating the weights of its Top-K nearest neighbors (NNs) on the mesh"
        # Assuming mesh nodes are in canonical space, we need to map x_t to canonical space first?
        # Or find NNs in target space if mesh is also in target space?
        # The source says "transform... to canonical space... using skinning motion field"
        # and "determine its LBS weights wb by interpolating... on the mesh".
        # It implies the mesh is in canonical space, but LBS is applied to target points.
        # Let's assume we find NNs in the space where x_t resides, or map x_t to canonical space
        # to find NNs on the canonical mesh. The source says "Given a sampled point xt in the target space...
        # determine its LBS weights... by interpolating... on the mesh".
        # This suggests the mesh might be in target space or the NN search is done in target space.
        # However, the latent codes are anchored on the canonical mesh.
        # Let's assume the mesh nodes provided are in canonical space, and we need to find
        # the corresponding nodes in target space for LBS, or the mesh is deformed to target space.
        # The source says "deformable canonical mesh M".
        # Let's assume we have a function to get LBS weights for a target point.
        w_b = mesh.get_lbs_weights(x_t, k_neighbors)
        
        # Step 3: Transform to canonical space
        # Inverse LBS + Neural Deformation
        # x_c = LBS^-1(x_t, w_b, theta_t) + def_mlp(x_c, theta_t)
        # This is iterative or implicit. Let's approximate.
        x_c_init = mesh.inverse_lbs(x_t, w_b, target_pose)
        x_c = x_c_init + def_mlp(x_c_init, target_pose)
        
        # Step 4: Interpolate Latent Codes
        # Find K nearest neighbors on the canonical mesh for x_c
        # mesh_nodes are in canonical space
        idx = mesh.find_k_nearest_nodes(x_c, k_neighbors)
        
        # Interpolate lgeo and lrgb
        l_geo = interpolate(lgeo_nodes[idx], w_b_interpolation) # Weights from NN interpolation
        l_rgb = interpolate(lrgb_nodes[idx], w_b_interpolation)
        
        # Step 5: Signed Height Indicator
        # Find nearest node for UV and distance
        nearest_node_idx = idx[0] # Assuming first is nearest
        u, v = uv_nodes[nearest_node_idx]
        # Point-to-surface distance d_bar
        # This requires a surface representation. Assume mesh provides this.
        d_bar = mesh.point_to_surface_distance(x_c)
        h_tilde = np.array([u, v, d_bar])
        
        # Step 6: Radiance Field Prediction
        # Input: l_geo, l_rgb, h_tilde
        sigma, c_0 = rad_mlp(l_geo, l_rgb, h_tilde)
        
        # Step 7: Shading Modulation
        s = shade_mlp(target_pose)
        c_mod = c_0 * s # Element-wise multiplication
        
        # Step 8: Volume Rendering
        alpha = 1.0 - np.exp(-sigma) # Assuming delta_t = 1 for simplicity
        T_i = transmittance
        final_color += T_i * alpha * c_mod
        transmittance *= (1.0 - alpha)
        
        if transmittance < 1e-4:
            break
            
    return final_color

def interpolate(values, weights):
    """Simple weighted interpolation."""
    return np.sum(values * weights[:, np.newaxis], axis=0)
```

## Subtopics

### Local Geometry Editing
UVA allows local geometry deformation by altering the latent codes $\mathbf{l}_{geo}$ anchored on specific mesh nodes [S1, unofficial].

### Local Texture Editing
UVA allows local texture manipulation by altering the latent codes $\mathbf{l}_{rgb}$ anchored on specific mesh nodes [S1, unofficial].

### Novel View Synthesis
UVA supports rendering novel views by integrating colors along rays from different camera poses [S1, unofficial].

### Novel Pose Rendering
UVA supports rendering avatars in new poses by using the target pose parameters $\theta_t$ in the deformation and shading fields [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Depends on the number of samples, the cost of finding nearest neighbors, and the MLP evaluations.
*   **Memory Complexity:** Not given in the sources. Depends on the number of mesh nodes and the size of latent codes.
*   **Measured Performance:** Experiments on multiple human avatars demonstrate competitive results in novel view synthesis and novel pose rendering [S1, unofficial].
*   **Strengths:** Enables local and independent editing of geometry and texture; end-to-end trainable; jointly optimized with the skinning field [S1, unofficial].
*   **Limitations:** Spatial ambiguity in code interpolation requires the signed height indicator; relies on a deformable mesh [S1, unofficial].
*   **When to use:** When local editing of geometry and texture is required for dynamic avatars.
*   **When not to use:** For static objects where two-stage distilling might be sufficient, or when mesh-based representation is not desirable.

## Background
Unified Volumetric Avatar (UVA) was proposed in the paper "UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing" [S1, unofficial]. It addresses the limitations of existing methods that either rely on pre-defined UV mapping (hindering local editing) or use two-stage distilling (requiring manual tuning and being limited to static objects) [S1, unofficial]. The method builds on the idea of encoding geometry and appearance in separate fields anchored on a deformable mesh in canonical space [S1, unofficial].

## Sources
[S1] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
