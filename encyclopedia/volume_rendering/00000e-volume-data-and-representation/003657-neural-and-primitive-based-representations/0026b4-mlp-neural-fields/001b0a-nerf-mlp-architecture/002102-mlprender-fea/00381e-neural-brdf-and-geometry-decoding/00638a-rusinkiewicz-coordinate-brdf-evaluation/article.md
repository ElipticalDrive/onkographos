# Rusinkiewicz Coordinate BRDF Evaluation

Also called: Local Frame BRDF, Rusinkiewicz BRDF

## Overview
This method evaluates a learned Bidirectional Reflectance Distribution Function (BRDF) by transforming incident light and view directions into a local coordinate frame aligned with the surface normal, converting them to Rusinkiewicz coordinates, and querying a neural network [S1, unofficial]. It differs from sibling methods like [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md) by explicitly utilizing the geometric symmetry of the Rusinkiewicz parameterization to reduce the input dimensionality and improve sampling efficiency for the BRDF latent code [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface normal $\mathbf{n}$ (world space), view direction $\mathbf{v}$ (world space, pointing from surface to camera), light direction $\mathbf{l}$ (world space, pointing from surface to light), BRDF latent code $\mathbf{Z}$, albedo $\mathbf{c}_{albedo}$, and a neural network model $\mathcal{N}$ with an embedding function $E(\cdot)$.
*   **Outputs:** Specular BRDF value $f_{spec}$ and final reflectance $f_{total}$.

**Data Structures:**
*   **Rusinkiewicz Coordinates ($\mathbf{r}$):** A 3D vector representing the direction in Rusinkiewicz space. Computed from local view and light directions [S1, unofficial].
*   **Concatenated Input ($\mathbf{z}_{rusink}$):** A tensor of shape $[N, 3 + z_{dim}]$, where the first 3 dimensions are the Rusink coordinates and the remaining $z_{dim}$ dimensions are the BRDF latent $\mathbf{Z}$ [S1, unofficial]. Here $z_{dim}$ is the dimension of the BRDF latent code $\mathbf{Z}$ stored in `brdf_prop` [S1, unofficial].

**Steps:**
1.  **Local Frame Construction:** Generate a world-to-local rotation matrix $R_{world \to local}$ from the surface normal $\mathbf{n}$ [S1, unofficial]. This matrix aligns the local Z-axis with $\mathbf{n}$ [G].
2.  **Direction Transformation:** Transform the view direction $\mathbf{v}$ and light direction $\mathbf{l}$ into the local frame using the rotation matrix:
    $$ \mathbf{v}_{local} = R_{world \to local} \mathbf{v} $$
    $$ \mathbf{l}_{local} = R_{world \to local} \mathbf{l} $$
    In implementation, this is often done via einsum operations on batched tensors [S1, unofficial].
3.  **Rusinkiewicz Conversion:** Flatten the local directions and convert them to Rusinkiewicz coordinates $\mathbf{r}$:
    $$ \mathbf{r} = \text{dir2rusink}(\mathbf{l}_{local}, \mathbf{v}_{local}) $$
    The standard Rusinkiewicz parameterization maps two vectors to a 3D vector, exploiting the symmetry of the BRDF [S1, unofficial].
4.  **Front-Face Masking:** Calculate the cosine of the angle between the light direction and the normal in the local frame:
    $$ \cos = \mathbf{l}_{local} \cdot \mathbf{n}_{local} $$
    Create a boolean mask `front_lit` where `front_lit` is true if $\cos > 0$ [S1, unofficial]. Backlit directions are masked out to prevent invalid BRDF queries [S1, unofficial].
5.  **Embedding and Concatenation:** Embed the Rusinkiewicz coordinates using an embedding function $E(\cdot)$:
    $$ \mathbf{r}_{emb} = E(\mathbf{r}) $$
    Concatenate the embedded coordinates with the BRDF latent code $\mathbf{Z}$:
    $$ \mathbf{z}_{rusink} = \text{concat}(\mathbf{Z}, \mathbf{r}_{emb}) $$
    The shape of $\mathbf{z}_{rusink}$ is $[N, 3 + z_{dim}]$ [S1, unofficial]. Note: If $\mathbf{Z}$ is tiled to match the number of light directions, ensure dimensions align [S1, unofficial].
6.  **Neural Network Query:** Pass $\mathbf{z}_{rusink}$ through the BRDF neural network $\mathcal{N}$:
    $$ \mathbf{brdf}_{raw} = \mathcal{N}(\mathbf{z}_{rusink}) $$
7.  **Scattering and Masking:** Scatter the results back to the original positions, applying the `front_lit` mask. For backlit directions, the specular component is typically zero or handled via the mask [S1, unofficial].
8.  **Final Reflectance:** Combine the specular output with the Lambertian term:
    $$ f_{spec} = \mathbf{brdf}_{raw} $$
    $$ f_{total} = \frac{\mathbf{c}_{albedo}}{\pi} + f_{spec} $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_rusinkiewicz_brdf(normal, pts2c, pts2l, z, albedo, embedder, brdf_model):
    """
    Evaluate BRDF using Rusinkiewicz coordinates.
    
    Args:
        normal: Surface normals, shape [N, 3]
        pts2c: View directions (surface to camera), shape [N, 3]
        pts2l: Light directions (surface to light), shape [N, L, 3]
        z: BRDF latent code, shape [N, z_dim]
        albedo: Surface albedo, shape [N, 3]
        embedder: Embedding function for Rusink coordinates
        brdf_model: Neural network for BRDF evaluation
        
    Returns:
        brdf: Total BRDF value, shape [N, L, 3]
    """
    # Step 1: Generate world-to-local rotation matrix from normal
    # gen_world2local creates a rotation matrix aligning local Z with normal
    world2local = gen_world2local(normal)  # Shape [N, 3, 3]
    
    # Step 2: Transform view and light directions to local frame
    # vdir: [N, 3], ldir: [N, L, 3]
    vdir = torch.einsum('nij, nj -> ni', world2local, pts2c)
    ldir = torch.einsum('nij, njk -> nik', world2local, pts2l)
    
    # Step 3: Flatten directions and convert to Rusink coordinates
    # ldir_flat: [N*L, 3], vdir_flat: [N, 3] (broadcasted or tiled)
    ldir_flat = ldir.reshape(-1, 3)
    vdir_flat = vdir.unsqueeze(1).expand(-1, ldir.shape[1], -1).reshape(-1, 3)
    
    rusink = dir2rusink(ldir_flat, vdir_flat)  # Shape [N*L, 3]
    
    # Step 4: Mask out backlit directions
    # local_normal is [0, 0, 1] in local frame
    local_normal = torch.tensor([0.0, 0.0, 1.0], device=normal.device)
    cos = ldir_flat @ local_normal  # Shape [N*L]
    front_lit = cos > 0  # Shape [N*L]
    
    # Step 5: Tile BRDF latent Z to match light count
    # z: [N, z_dim] -> z_tiled: [N*L, z_dim]
    z_tiled = z.unsqueeze(1).expand(-1, ldir.shape[1], -1).reshape(-1, z.shape[1])
    
    # Step 6: Embed Rusink coordinates and concatenate with Z
    rusink_embed = embedder(rusink)  # Shape [N*L, 3] (or embedded dim)
    z_rusink = torch.cat((z_tiled, rusink_embed), dim=1)  # Shape [N*L, z_dim + 3]
    
    # Step 7: Pass through network
    brdf_fl = brdf_model.net(z_rusink)  # Shape [N*L, 3]
    
    # Step 8: Scatter results back and apply mask
    # For backlit directions, set specular to 0
    brdf_fl_masked = torch.where(front_lit.unsqueeze(1), brdf_fl, torch.zeros_like(brdf_fl))
    
    # Reshape back to [N, L, 3]
    brdf_spec = brdf_fl_masked.reshape(-1, ldir.shape[1], 3)
    
    # Step 9: Combine with Lambertian term
    # albedo: [N, 3]
    brdf_lambertian = albedo.unsqueeze(1) / torch.tensor(np.pi, dtype=torch.float32, device=albedo.device)
    brdf = brdf_lambertian + brdf_spec
    
    return brdf
```

**Parameters:**
*   $z_{dim}$: Dimension of the BRDF latent code $\mathbf{Z}$ [S1, unofficial]. Not given in the sources.
*   Embedder: Typically a positional encoding or simple MLP [G].

**Edge Cases and Pitfalls:**
*   **Backlit Directions:** The mask `front_lit` must be applied to prevent querying the BRDF for invalid light directions ($\cos \le 0$) [S1, unofficial].
*   **Precision:** Ensure consistent data types and device placement for tensors, especially when combining albedo and specular terms [S1, unofficial].
*   **Degenerate Normals:** If the normal is zero or near-zero, the rotation matrix generation may fail [G].

## Subtopics

### [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md)
The parent method that decodes appearance features into BRDF values, normals, and colors. Rusinkiewicz evaluation is a specific technique used within this framework for efficient BRDF querying.

### [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md)
Uses tensor factorization for feature decoding. Rusinkiewicz evaluation is distinct in its explicit use of geometric coordinate transformation for BRDF input.

### [Neural Surface Rendering](9960-neural-surface-rendering.md)
A broader category of rendering methods. Rusinkiewicz evaluation is a specific component for handling material appearance.

### [Neural SDF and Material MLPs](10268-neural-sdf-and-material-mlps.md)
Focuses on SDF and material representation. Rusinkiewicz evaluation is a technique for evaluating the material (BRDF) part.

### [Reflectance NeRF](14040-reflectance-nerf.md)
A method for neural radiance fields with reflectance. Rusinkiewicz evaluation can be used as the BRDF evaluation component in such frameworks.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of points and $L$ is the number of light directions. The dominant cost is the neural network query and direction transformations [S1, unofficial].
*   **Memory Complexity:** $O(N \cdot L \cdot (3 + z_{dim}))$ for storing the concatenated input tensor $\mathbf{z}_{rusink}$ [S1, unofficial].
*   **Strengths:** Reduces the input dimensionality for the BRDF network by exploiting geometric symmetry, potentially improving sampling efficiency and generalization [S1, unofficial].
*   **Limitations:** Requires accurate surface normals for the local frame transformation [S1, unofficial]. Backlit directions must be handled explicitly [S1, unofficial].
*   **When to Use:** When evaluating BRDFs for surface-based rendering with explicit normals and multiple light directions [S1, unofficial].
*   **When Not to Use:** When normals are unavailable or unreliable, or when the BRDF does not benefit from the Rusinkiewicz parameterization [G].

## Background
The method is implemented in the L-Tracing repository [S1, unofficial]. It builds on the concept of Rusinkiewicz coordinates for efficient BRDF representation and evaluation, adapted for neural field-based rendering [S1, unofficial].

## Sources
[S1] ziyc/L-Tracing, https://github.com/ziyc/L-Tracing, fetched 2026-10-26 (unofficial)
