# Object-Centric Voxelization for Dynamic Scenes

Also called: DynaVol, Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization

## Overview
DynaVol is an unsupervised 3D dynamic scene decomposition method that unifies geometric structures and object-centric learning within a differentiable volume rendering framework [S1, unofficial]. It solves the multi-view mismatch of 2D object segmentation by ensuring 3D consistency, enabling novel view synthesis, and allowing direct scene editing [S1, unofficial]. Unlike 2D image decomposition methods such as SAVi, DynaVol infers a probability distribution over objects at individual spatial locations via object-centric voxelization, facilitating fine-grained separation without geometric priors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial timestamp images and poses $\{I_v^{t=1}, T_v^{t=1}\}_{v=1}^V$ and monocular video frames $\{I_t, T_t\}_{t=2}^T$ [S1, unofficial]. The method supports scenarios where $V=1$ (only one initial camera pose) and monocular camera trajectories [S1, unofficial].
*   **Outputs:** Synthesized target pixel values for novel views and time steps, along with decomposed object-centric representations $\{(c_n, \sigma_n)\}_{n=1}^N$ [S1, unofficial].

**Data Structures:**
*   **Object-centric 4D Voxel Representation:** $V_t \in \mathbb{R}^{N \times N_x \times N_y \times N_z}$ [S1, unofficial]. This structure maintains the time-varying density $\{\sigma_n\}_{n=1}^N$ for each possible object across spatial dimensions $N_x, N_y, N_z$ and time $t$ [S1, unofficial].
*   **Slots:** $N$ is a predefined number of object slots, assumed to be larger than the actual number of objects in the scene to handle unknown counts [S1, unofficial].

**Algorithm Steps:**
1.  **Sampling:** Sample 3D points $\mathbf{x} = (x, y, z)$ and view directions $\mathbf{d}$ for each timestamp $t$ [S1, unofficial].
2.  **Object-Centric Voxelization:** Learn the object-centric projection of $(\mathbf{x}, \mathbf{d}, t)$ to $\{(c_n, \sigma_n)\}_{n=1}^N$ [S1, unofficial]. This infers the probability distribution over objects at individual spatial locations [S1, unofficial].
3.  **Dynamics Module:** Learn transitions of voxel grid features in canonical space over time [S1, unofficial]. The voxel features evolve through a canonical-space deformation function [S1, unofficial].
4.  **Volume Slot Attention:** Aggregate voxel representations to refine object-level, time-invariant global features [S1, unofficial]. This module progressively refines global features by aggregating local voxel representations [S1, unofficial].
5.  **Compositional NeRF Decoder:** Estimate density and color for each object at view direction $\mathbf{d}$ using both local voxel features and global object representations [S1, unofficial].
6.  **Rendering:** Re-combine $\{(c_n, \sigma_n)\}_{n=1}^N$ to synthesize target pixel values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynavol_render(initial_images, initial_poses, video_frames, N, Nx, Ny, Nz):
    # Initialize object-centric 4D voxel representation V_t in canonical space
    V_t = torch.zeros((N, Nx, Ny, Nz))
    global_features = torch.zeros((N, 256)) # Object-level, time-invariant global features
    T = len(video_frames)
    rendered_images = []

    for t in range(1, T + 1):
        # Sample 3D points x and view directions d for each timestamp t
        points, directions = sample_rays(video_frames[t], initial_poses if t == 1 else None)

        # Object-centric voxelization: learn projection of (x, d, t) to per-slot features
        # Infers probability distribution over objects at individual spatial locations
        # [G] Standard MLP/convolutional projection from observation to canonical voxel features
        voxel_features = object_centric_voxelization(points, directions, t, V_t)

        # Dynamics module: learn transitions of voxel grid features in canonical space over time
        # [G] Standard recurrent or temporal convolutional update for voxel grid features
        V_t = dynamics_module(V_t, t)

        # Volume slot attention: aggregate voxel representations to refine global features
        # [G] Standard attention mechanism aggregating spatial voxels into slot vectors
        global_features = volume_slot_attention(voxel_features, global_features)

        # Compositional NeRF decoder: estimate density and color for each object at view direction d
        densities = []
        colors = []
        for n in range(N):
            # [G] Standard MLP combining local voxel features, global slot features, and view direction
            sigma_n, c_n = compositional_nerf_decoder(
                voxel_features[n], global_features[n], directions
            )
            densities.append(sigma_n)
            colors.append(c_n)

        # Re-combine per-slot densities and colors to synthesize target pixel values
        # [G] Standard differentiable volume rendering integrating per-slot contributions along rays
        rendered_image = volume_rendering(densities, colors, directions)
        rendered_images.append(rendered_image)

    return rendered_images, V_t, global_features
```

**Parameters:**
*   $N$: Predefined number of object slots. Must be larger than the actual number of objects [S1, unofficial].
*   $N_x, N_y, N_z$: Spatial dimensions of the voxel grid [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Monocular Input:** The method handles scenarios with only one initial camera pose ($V=1$) and monocular camera trajectories smoothly, avoiding multi-view mismatch issues common in 2D approaches [S1, unofficial].
*   **Slot Count:** $N$ is a hyperparameter that must be set conservatively high to account for unknown object counts, potentially leading to unused slots [S1, unofficial].

## Subtopics

### MoCo-NeRF
Uses residual-based canonical transformation and pose-conditioned radiance residuals. Choose when pose information is available and deformation is complex. [MoCo-NeRF](16197-moco-nerf.md)

### Neural Scene Graphs
Explicitly models scene objects and their relationships. Choose when explicit scene structure and reasoning are required. [Neural Scene Graphs](17706-neural-scene-graphs.md)

### A-NeRF
Uses skeleton-based inverse Linear Blend Skinning (LBS). Choose for articulated human or rigid-body scenes with known skeletons. [A-NeRF](18108-a-nerf.md)

### Pixel-aligned Volumetric Avatars (PVA)
Uses pixel-aligned features for high-fidelity avatar rendering. Choose for detailed human avatar synthesis. [PVA](18139-pixel-aligned-volumetric-avatars-pva.md)

### SelfNeRF
Uses surface-based signed distances to nearest neighbors. Choose when surface reconstruction is a viable prior. [SelfNeRF](18160-selfnerf.md)

### CLA-NeRF
Uses canonical space alignment with local appearance. Choose for scenes with varying lighting conditions. [CLA-NeRF](20285-cla-nerf.md)

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on simulated 3D dynamic scenes with varying numbers of objects, diverse motions, shapes (cubes, spheres, real-world shapes), and materials (rubber, metal) [S1, unofficial]. Demonstrated on real-world videos for novel view synthesis and editing [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Ensures 3D consistency, enables novel view synthesis, allows direct scene editing (object removal, replacement, trajectory modification) by manipulating voxel grids or deformation functions without further training [S1, unofficial]. Handles monocular inputs and unknown object counts [S1, unofficial].
*   **Limitations:** Requires predefined number of slots $N$ which may be larger than actual objects [S1, unofficial].
*   **When to use:** Unsupervised dynamic scene decomposition, novel view synthesis, and scene editing where 3D consistency is critical [S1, unofficial].
*   **When not to use:** When explicit geometric priors are available and preferred, or when the number of objects is known and small [S1, unofficial].

## Background
DynaVol was presented at ICLR 2024 [S1, unofficial]. It addresses the challenge of unsupervised learning of object-centric representations in dynamic visual scenes [S1, unofficial]. Unlike previous approaches that decompose 2D images, DynaVol introduces a 3D scene generative model that unifies geometric structures and object-centric learning [S1, unofficial]. It builds on the principles of dynamic and articulated scene modeling by mapping 3D query points to a canonical space [G].

## Sources
[S1] DynaVol: Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization, https://doi.org/10.48550/arxiv.2305.00393, fetched 2026-10-26 (unofficial)
