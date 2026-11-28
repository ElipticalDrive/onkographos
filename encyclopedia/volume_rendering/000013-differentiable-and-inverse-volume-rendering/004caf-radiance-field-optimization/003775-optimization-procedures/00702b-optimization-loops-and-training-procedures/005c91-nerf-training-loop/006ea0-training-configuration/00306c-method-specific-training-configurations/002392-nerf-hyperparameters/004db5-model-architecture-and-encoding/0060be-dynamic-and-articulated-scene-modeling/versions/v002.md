# Dynamic and Articulated Scene Modeling

Also called: Dynamic NeRF, Articulated NeRF, Human NeRF

## Overview
Dynamic and Articulated Scene Modeling extends static radiance fields to handle time-varying geometry and non-rigid deformations, primarily in human performance capture. It solves the ill-posed problem of mapping dynamic 3D points to a static representation by introducing canonical spaces, skeletal constraints, or surface-relative encodings. Unlike static methods, these approaches must jointly optimize or infer pose, deformation, and appearance from monocular or sparse multi-view video inputs.

## Shared principles
All methods in this family map a 3D query point $\mathbf{x}$ observed at time $t$ to a canonical space $\mathbf{x}_c$ where the radiance field is static. The rendering process remains volume rendering, but the density $\sigma$ and color $c$ are conditioned on the deformation state.

1.  **Canonical Transformation:** A transformation function $T(\mathbf{x}, t)$ maps observation space to canonical space. Common strategies include:
    *   **Skeleton-based:** Using inverse Linear Blend Skinning (LBS) or skeletal relative coordinates [S1, 18108-a-nerf, unofficial].
    *   **Surface-based:** Using signed distances to nearest neighbors in a reconstructed surface mesh [18160-selfnerf].
    *   **Residual-based:** Predicting radiance residuals in the canonical space conditioned on pose [16197-moco-nerf].
2.  **Conditioning:** The radiance field MLP $F_\omega$ takes the canonical point $\mathbf{x}_c$, view direction $\mathbf{v}$, and often a pose or identity code as input.
3.  **Optimization:** Training typically involves minimizing photometric loss $\| C_{rendered} - I_{gt} \|$. Articulated methods often include regularization terms for pose smoothness and consistency with initial pose estimates [18108-a-nerf].

## Subtopics

### [MoCo-NeRF](16197-moco-nerf.md)
MoCo-NeRF models non-rigid body motions as radiance residual fields rather than spatial offsets [16197-moco-nerf]. It separates the rigid canonical representation from pose-dependent non-rigid residuals, allowing for direct color supervision of the dynamic components. Unlike sibling methods that rely on spatial deformation fields, MoCo-NeRF reduces learning complexity by predicting residuals in the radiance space conditioned on a rigid prior [16197-moco-nerf]. It uses a skeletal weights volume decoder to predict canonical skinning weights $w^c_k$ for $K$ joints [16197-moco-nerf]. The canonical point is computed via inverse linear blend skinning:
$$ \mathbf{x}_c = \sum_{k=1}^K w^o_k (R_k \mathbf{x} + \mathbf{t}_k) $$
where observation weights $w^o_k$ are derived from canonical weights [16197-moco-nerf]. Non-rigid features are generated using cross-attention between a learnable base code and sinusoidal-encoded joint positions [16197-moco-nerf].

### [A-NeRF](18108-a-nerf.md)
A-NeRF learns a user-specific neural 3D body model and underlying skeleton pose from unlabelled monocular videos by extending NeRFs with a skeleton to apply to time-varying and articulated motion [18108-a-nerf, unofficial]. It solves the ill-posed inverse problem of mapping dynamic 3D points to a static radiance field by defining spatial latent variables relative to the pose of body parts, effectively overparameterizing the representation with one embedding per bone [18108-a-nerf, unofficial]. Unlike explicit surface models that use forward kinematics, A-NeRF learns an implicit volumetric model that jointly refines pose and appearance without requiring ground truth labels for shape or pose [18108-a-nerf, unofficial]. It employs a "Rel. Ray" encoding combining bone-relative distance, direction, and the angle between the ray and the joint vector [18108-a-nerf, unofficial].

### SelfNeRF
SelfNeRF is a novel view synthesis method for human performance that trains from scratch using monocular self-rotating video inputs, achieving high-fidelity results in approximately twenty minutes [18160-selfnerf, unofficial]. It distinguishes itself from sibling methods like MoCo-NeRF by utilizing a surface-relative hash encoding to aggregate inter-frame information, enabling rapid convergence without multi-view inputs [18160-selfnerf, unofficial]. It relies on external reconstruction methods (e.g., VideoAvatar) to recover surface shape point clouds $\{T_t\}$ and maps query points to canonical space via K-nearest neighbor signed distances [18160-selfnerf, unofficial].

### Neural Human Performer
A feed-forward method for synthesizing novel-view videos of arbitrary human performers from sparse multi-view camera inputs (3-4 views) by learning generalizable neural radiance fields using a parametric 3D body model (SMPL) and transformer-based aggregation of spatio-temporal features [S1, unofficial]. It constructs time-augmented skeletal features by leveraging SMPL and aggregates temporal information using a temporal transformer, followed by a multi-view transformer for cross-attention [S1, unofficial].

### HOSNeRF
A novel 360° free-view rendering method capable of reconstructing dynamic human-object-scene neural radiation fields from a single monocular video [S2]. It estimates complex object motion during human-object interaction by introducing new object skeletons into the traditional human skeletal hierarchy and uses two learnable object state embeddings for different interaction contexts [S2]. It significantly outperforms existing techniques on challenging datasets, with a 40% to 50% improvement on the LPIPS metric [S2].

### MonoNeRF
A method for learning generic dynamic radiation fields from monocular video [S2]. It learns both point features and scene streams through video feature extraction, implicit velocity fields, and stream-based feature aggregation modules [S2]. It supports new applications such as scene editing, unseen frame synthesis, and fast new scene adaptation [S2].

## Background
The extension of NeRF to dynamic scenes originated with the need to capture human performance and articulated objects from limited camera views. Early works like [18108-a-nerf] addressed the ill-posed nature of monocular dynamic reconstruction by introducing skeletal-relative encodings and joint optimization of pose and appearance. Subsequent methods like [16197-moco-nerf] and [18160-selfnerf] refined this by decoupling rigid and non-rigid components or leveraging surface geometry for faster convergence. Generalizable approaches like Neural Human Performer [S1, unofficial] and HOSNeRF [S2] further expanded the domain to sparse multi-view and single-view human-object interactions, respectively.

## Sources
[S1] Neural Human Performer: Learning Generalizable Radiance Fields for Human Performance Rendering, https://doi.org/10.48550/arxiv.2109.07448, fetched 2026-10-30 (unofficial)
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
