# Texture Painting

Also called: local texture editing

## Overview
Texture painting is a method for locally editing the appearance of a 3D avatar or mesh by fine-tuning specific latent codes to match a 2D input painting [S2, unofficial]. It solves the problem of transferring 2D artistic intent to a 3D neural representation while preserving the geometry and appearance of unedited regions [S1]. Unlike global texture replacement, it operates on a sparse set of anchor nodes, allowing for independent, spatial-aware local edits [S2, unofficial].

## How it works
**Inputs:** A 2D painting image, a binary mask indicating the region to edit, the current mesh scaffold (vertices and faces), and the pre-trained neural implicit field parameters (shading network, signed height indicator, and latent codes).
**Outputs:** Updated latent codes for the affected anchor nodes.

**Data Structures:**
*   **Mesh Scaffold:** Defined by vertices $V$ and faces $F$. Each vertex $v_i$ has an associated latent code $c_i$.
*   **Binary Mask:** A 2D image $M$ where $M(u,v)=1$ if the pixel $(u,v)$ is painted, 0 otherwise.
*   **Latent Codes:** Vector $c_i$ stored at each anchor node (mesh vertex).

**Steps:**
1.  **Ray Casting:** Shoot probing rays from the camera through the painted pixels in the 2D painting image towards the mesh scaffold [S1].
2.  **Hit Detection:** Identify the faces hit by these rays. Collect all vertices of the hit faces to determine the set of affected anchor nodes $V_{affected}$ [S1].
3.  **Mask Dilation:** Dilate the binary mask $M$ slightly to create a boundary region [S2, unofficial].
4.  **Training Ray Sampling:** Sample training rays using the dilated mask. This ensures smooth texture boundaries by including pixels adjacent to the painted region in the optimization [S2, unofficial].
5.  **Optimization Setup:**
    *   Fix the shading network parameters.
    *   Fix the latent codes for all nodes $v_j \notin V_{affected}$.
    *   Enable gradient updates only for latent codes $c_i$ where $v_i \in V_{affected}$ [S1][S2, unofficial].
6.  **Loss Function:** Minimize the difference between the rendered pixel values and the target painting pixel values. The pixel value is determined by the interpolated latent code, the local signed height indicator, and the shading factor [S2, unofficial].
7.  **Learning Rate Adjustment:** Use a higher learning rate for the latent codes compared to other parameters to prevent the non-trainable signed height indicator from dominating the optimization [S2, unofficial]. Specifically, increase the learning rate gap further during texture painting compared to standard training [S2, unofficial].
8.  **Iteration:** Repeat steps 5-7 for a fixed number of iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_painting(painting_image, binary_mask, mesh, latent_codes, shading_network, signed_height_indicator, num_iterations=8000):
    # 1. Identify affected nodes
    hit_faces = shoot_rays_through_masked_pixels(painting_image, binary_mask, mesh)
    affected_vertices = set()
    for face in hit_faces:
        affected_vertices.update(face.vertices)
    
    # 2. Prepare optimizer
    # Only latent codes for affected vertices are trainable
    trainable_codes = [latent_codes[v] for v in affected_vertices]
    
    # 3. Dilate mask for smooth boundaries
    dilated_mask = dilate(binary_mask, kernel_size=3)
    
    # 4. Optimization loop
    for i in range(num_iterations):
        # Sample rays based on dilated mask
        rays = sample_training_rays(dilated_mask)
        
        # Render current state
        # Note: shading_network and signed_height_indicator are fixed
        rendered_pixels = render_mesh(mesh, trainable_codes, shading_network, signed_height_indicator, rays)
        
        # Get target pixels from painting
        target_pixels = get_pixels_from_painting(rays, painting_image)
        
        # Compute loss
        loss = mse_loss(rendered_pixels, target_pixels)
        
        # Backpropagate only to trainable_codes
        # Use higher learning rate for codes to counter signed height influence
        loss.backward()
        
        # Update codes
        for code in trainable_codes:
            code -= learning_rate_code * code.grad
            
        # Zero gradients
        for code in trainable_codes:
            code.grad = 0
            
    return latent_codes
```

**Parameters:**
*   **Learning Rate (Latent Codes):** Significantly higher than other parameters (e.g., 10 times higher than standard training) [S2, unofficial].
*   **Iterations:** ~8000 iterations [S1].
*   **Optimization Duration:** ~1 hour on the reported hardware [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting to Signed Height:** The texture decoder may overfit to the non-trainable signed height indicator during single-image fine-tuning, generating artifacts at other positions [S2, unofficial]. This is mitigated by increasing the learning rate gap [S2, unofficial].
*   **Boundary Artifacts:** Without mask dilation, texture boundaries may appear sharp or aliased [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of optimization iterations and the number of affected pixels/vertices. The reported duration is ~1 hour for 8000 iterations [S1].
*   **Memory Complexity:** Low, as only a subset of latent codes are updated and stored in memory during optimization.
*   **Strengths:** Allows precise, local 2D-to-3D texture transfer; preserves geometry and unedited appearance; supports novel view synthesis of edited textures [S2, unofficial].
*   **Limitations:** Requires a pre-trained neural implicit field; optimization can be slow (~1 hour); risk of artifacts if learning rates are not tuned correctly [S2, unofficial].
*   **When to use:** When local appearance editing is required without retraining the entire model or altering geometry [S2, unofficial].
*   **When not to use:** When global texture changes are needed or when real-time editing is required [S2, unofficial].

## Background
Texture painting was introduced in the context of Unified Volumetric Avatars (UVA) and NeuMesh for editing human avatars and 3D objects [S1][S2, unofficial]. It addresses the challenge of disentangling geometry and appearance in neural implicit fields to allow for fine-grained editing [S2, unofficial]. Previous methods relied on neural texture map replacement or separate implicit fields, which lacked the flexibility of this approach [S2, unofficial].

## Sources
[S1] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S2] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
