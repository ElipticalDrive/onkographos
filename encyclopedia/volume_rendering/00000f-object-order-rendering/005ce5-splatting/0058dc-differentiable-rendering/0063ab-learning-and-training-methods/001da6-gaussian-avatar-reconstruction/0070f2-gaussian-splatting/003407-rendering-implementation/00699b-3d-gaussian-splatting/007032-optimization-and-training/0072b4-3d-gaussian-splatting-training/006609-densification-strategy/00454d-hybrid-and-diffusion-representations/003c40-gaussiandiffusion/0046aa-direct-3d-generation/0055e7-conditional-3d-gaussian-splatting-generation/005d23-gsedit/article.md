# GSEdit

Also called: Gaussian Editing via Instruct-Pix2Pix and SDS Loss

## Overview
GSEdit is a method for editing pre-trained 3D Gaussian Splatting (3DGS) representations or texture meshes using text or image instructions [S1, unofficial]. It solves the problem of applying 2D diffusion-based edits to 3D scenes while maintaining geometric consistency [S1, unofficial]. It sets itself apart from sibling methods like [GaussianObject](17321-gaussianobject.md) or [Promptable Explicit Scene Representation](15906-promptable-explicit-scene-representation.md) by directly utilizing Instruct-Pix2Pix and Score Distillation Sampling (SDS) loss to update the input representation, rather than relying on complex multi-view consistency constraints or explicit texture mapping networks like [Texture-GS](18279-appearance-aware-3d-gaussian-splatting-aags.md) [S1, unofficial].

## How it works
**Inputs:** A pre-trained 3D Gaussian Splatting representation or a texture mesh [S1, unofficial]. Text or image instructions for the desired edit [S1, unofficial].

**Outputs:** An edited 3D Gaussian Splatting representation or texture mesh [S1, unofficial].

**Data Structures:**
The method operates on the standard 3DGS data structure: a set of $N$ Gaussians, each defined by position $\mathbf{\mu}_i$, covariance $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and spherical harmonic coefficients for color $\mathbf{c}_i$ [S1, unofficial]. For mesh inputs, the structure includes vertex positions, face connectivity, and UV coordinates [S1, unofficial].

**Steps:**
1.  **Input Preparation:** Load the pre-trained 3DGS or texture mesh [S1, unofficial].
2.  **2D Edit Generation:** Use Instruct-Pix2Pix to generate edited 2D images based on the input instructions [S1, unofficial]. This involves rendering views from the current 3DGS/mesh, passing them through Instruct-Pix2Pix with the edit prompt, and obtaining the edited 2D images [S1, unofficial].
3.  **Loss Calculation:** Compute the Score Distillation Sampling (SDS) loss between the rendered views of the current 3DGS/mesh and the edited 2D images from Instruct-Pix2Pix [S1, unofficial]. The SDS loss guides the optimization to match the visual style and content of the 2D edits [S1, unofficial].
4.  **Optimization:** Update the parameters of the 3DGS (positions, covariances, opacities, colors) or the texture mesh (vertex positions, UVs) using gradient descent to minimize the SDS loss [S1, unofficial].
5.  **Iteration:** Repeat steps 2-4 for a fixed number of iterations or until convergence [S1, unofficial].

**Parameters:**
-   Number of optimization iterations: Not given in the sources.
-   Learning rate: Not given in the sources.
-   Instruct-Pix2Pix model weights: Standard pretrained weights [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Inconsistency:** Direct use of SDS loss can lead to geometric inconsistencies across views [S1, unofficial]. GSEdit relies on the inherent consistency of the 3DGS representation and the 2D edit, but may suffer from flickering or distortion if the edit is complex [S1, unofficial].
-   **Precision:** The optimization is sensitive to the learning rate and number of iterations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GSEdit(input_3dgs_or_mesh, edit_instruction, num_iterations):
    """
    Edits a 3D Gaussian Splatting or texture mesh using Instruct-Pix2Pix and SDS loss.
    
    Args:
        input_3dgs_or_mesh: Pre-trained 3DGS or texture mesh object.
        edit_instruction: Text or image instruction for the edit.
        num_iterations: Number of optimization steps.
        
    Returns:
        Edited 3DGS or texture mesh.
    """
    # Initialize optimizer with the parameters of input_3dgs_or_mesh
    optimizer = torch.optim.Adam(input_3dgs_or_mesh.parameters(), lr=0.01)
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Render current views from the 3DGS or mesh
        rendered_views = render_views(input_3dgs_or_mesh)
        
        # Apply Instruct-Pix2Pix to each rendered view
        edited_views = []
        for view in rendered_views:
            # Instruct-Pix2Pix takes the original image and the edit instruction
            edited_view = InstructPix2Pix(view, edit_instruction)
            edited_views.append(edited_view)
            
        # Compute SDS loss between rendered views and edited views
        # SDS loss is computed for each view and averaged
        loss = compute_sds_loss(rendered_views, edited_views)
        
        # Backpropagate and update parameters
        loss.backward()
        optimizer.step()
        
    return input_3dgs_or_mesh
```

## Subtopics

### Albedo Editing via Proxy Plane Spawning
A variant that focuses on editing the albedo (color) of 3DGS by initializing new Gaussians on a textured proxy plane [S2]. It places these Gaussians in 3D space using proxy depth and spawns additional albedo Gaussians by randomly sampling 20% of the pixels covered by the edit in the edited view [S2]. The augmented set of albedo Gaussians is then re-optimized to match the edited albedo images rendered from multiple viewpoints [S2]. This method is chosen when fine-grained albedo editing is required without altering geometry [S2].

### GaussianPrediction
A variant for dynamic scene modeling and future scenario synthesis [S3, unofficial]. It encodes position at a certain frequency and feeds it with a time coefficient into an MLP to predict the deformation of Gaussians [S3, unofficial]. It uses another MLP to derive the lifespan of a Gaussian indirectly to improve rendering quality [S3, unofficial]. It initializes key points using clustering techniques to drive Gaussians with motion similarity and spatial proximity, adding key points by localizing places with greater Gaussian gradient norm [S3, unofficial]. It learns the weight each key point contributes to Gaussians for deformation, a time-independent process, and uses a Graph Convolution Network to extract relational features between key points across frames, passing through a single-layer MLP to predict future motion and calculate predicted Gaussians [S3, unofficial]. This method is chosen for dynamic scene prediction and reconstruction [S3, unofficial].

### Video-3DGS
A variant for video reconstruction and editing [S3, unofficial]. It leverages an off-the-shelf open-vocabulary video object segmentation network to conduct Spatial Decomposition, extracting foreground moving objects (Frg-3DGS) and background points (Bkg-3DGS) [S3, unofficial]. It conducts Temporal Decomposition to divide the video sequence into several clips to minimize foreground motion for COLMAP process management [S3, unofficial]. It introduces MC-COLMAP to estimate point clouds from videos, recursively invoking COLMAP on certain clips in order of increasing frames until success [S3, unofficial]. Bkg-3DGS and Frg-3DGS are initialized and deformed by distinct deformation networks, and the frame is rendered by the learnable combination of these two kinds of Gaussians [S3, unofficial]. This method is chosen for dynamic video scene reconstruction [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** Not given in the sources. Depends on the number of Gaussians, iterations, and the cost of Instruct-Pix2Pix inference.
-   **Memory Complexity:** Not given in the sources. Depends on the number of Gaussians and the memory required for Instruct-Pix2Pix.
-   **Measured Performance:** Not given in the sources.
-   **Strengths:** Simple to implement, leverages powerful 2D diffusion models, can handle complex edits [S1, unofficial].
-   **Limitations:** Can suffer from geometric inconsistencies, requires careful tuning of optimization parameters [S1, unofficial].
-   **When to use:** When editing 3DGS with text or image instructions is required, and geometric consistency is not the primary concern [S1, unofficial].
-   **When not to use:** When high geometric consistency is required, or when editing dynamic scenes [S1, unofficial].

## Background
GSEdit was introduced as part of the recent advances in 3D Gaussian Splatting editing [S1, unofficial]. It builds on previous NeRF editing work like InstructNeRF2NeRF [S1, unofficial]. It is cited in the context of other independent research like GaussianEditor [S1, unofficial]. The method is part of the broader trend of integrating 2D diffusion models with 3D representations for editing and generation [S1, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S2] Intrinsic decomposition and editing of 3D Gaussian splats, https://doi.org/10.1145/3804495, fetched 2026-10-27
[S3] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
