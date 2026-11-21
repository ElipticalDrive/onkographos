# SWAGS

Also called: SWAGS [S1]

## Overview
SWAGS is a dynamic 3D Gaussian Splatting method that addresses the challenge of modeling complex, non-rigid motion in dynamic scenes by partitioning the temporal sequence into distinct windows based on motion intensity [S1][S2, unofficial]. Unlike sibling methods that track a single global deformation field or flow across the entire sequence, SWAGS trains separate dynamic 3DGS models for each window, each with its own canonical space and deformation field, thereby isolating local motion characteristics [S1][S2, unofficial]. This approach improves reconstruction fidelity for highly dynamic segments while maintaining temporal consistency through fine-tuning on overlapping frames [S1][S2, unofficial].

## How it works
SWAGS extends the 3D Gaussian Splatting framework to 4D by introducing time-varying parameters. Each Gaussian is defined by a position $\boldsymbol{\mu}_t$, rotation $q_t$, scaling $S_t$, and opacity $\alpha_t$ at time $t$ [S1][S2, unofficial]. The scaling matrix in 4D space is formed by adding a scaling factor for the time dimension on the diagonal of the standard 3D scaling matrix [S1][S2, unofficial]. The view-dependent color is represented by a 4D extension of spherical harmonics (SH), expressed as a combination of 3D SH with 1D basis functions over time [S1][S2, unofficial].

The core innovation lies in the window-based decomposition and deformation modeling:

1.  **Window Partitioning**: The dynamic sequence is divided into multiple windows. The partitioning criterion is based on the amount of motion within each segment [S1][S2, unofficial].
2.  **Separate Model Training**: For each window, a separate dynamic 3DGS model is trained [S1][S2, unofficial]. Each model possesses:
    *   A unique canonical space (the reference geometry at a canonical time).
    *   A dedicated deformation field.
3.  **Deformation Field**: The deformation field is implemented as a tunable Multi-Layer Perceptron (MLP) [S1][S2, unofficial]. This MLP is designed to focus specifically on modeling the dynamic part of the scene, distinguishing it from static background elements [S1][S2, unofficial].
4.  **Fine-Tuning for Consistency**: To ensure temporal consistency across window boundaries, the models are fine-tuned using overlapping frames between adjacent windows [S1][S2, unofficial]. During this fine-tuning phase, the MLP parameters are fixed, and only the canonical representation (the 3DGS attributes in the canonical space) is optimized [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def swags_train(sequence_frames, motion_thresholds):
    """
    Trains SWAGS models for a dynamic sequence.
    
    Args:
        sequence_frames: List of image frames with known camera poses and timestamps.
        motion_thresholds: Thresholds or criteria to determine window boundaries based on motion.
        
    Returns:
        A list of trained Dynamic3DGSModel objects, one per window.
    """
    # Step 1: Divide sequence into windows based on motion amount
    windows = partition_sequence_by_motion(sequence_frames, motion_thresholds)
    
    models = []
    
    for i, window in enumerate(windows):
        # Step 2: Train separate dynamic 3DGS model for this window
        # Initialize with canonical space and deformation field
        model = Dynamic3DGSModel(
            canonical_space=initialize_canonical_space(window),
            deformation_field=TunableMLP() # Focuses on dynamic part
        )
        
        # Train the model for this window
        model.train(window.frames, window.camera_poses)
        
        models.append(model)
        
    # Step 4 & 5: Fine-tune for temporal consistency using overlapping frames
    for i in range(len(models) - 1):
        # Identify overlapping frames between current window i and next window i+1
        overlap_frames = find_overlap_frames(models[i].window, models[i+1].window)
        
        if overlap_frames:
            # Fix the MLP, optimize only canonical representation
            models[i].freeze_mlp()
            models[i+1].freeze_mlp()
            
            # Optimize canonical representations to align across overlap
            fine_tune_canonical_representation(
                models[i], 
                models[i+1], 
                overlap_frames
            )
            
    return models

class Dynamic3DGSModel:
    def __init__(self, canonical_space, deformation_field):
        self.canonical_space = canonical_space
        self.deformation_field = deformation_field
        self.gaussians = [] # List of Gaussian attributes in canonical space
        
    def freeze_mlp(self):
        """Freeze the deformation field MLP parameters."""
        self.deformation_field.requires_grad = False
        
    def train(self, frames, camera_poses):
        """Standard 3DGS optimization loop for the window."""
        for frame, pose in zip(frames, camera_poses):
            # Render frame using 4D GS (time-varying pos, rot, scale, SH)
            rendered_image = render_4d_gaussians(
                self.gaussians, 
                pose, 
                frame.timestamp,
                self.deformation_field,
                self.canonical_space
            )
            # Compute loss and backpropagate
            loss = compute_loss(rendered_image, frame.image)
            optimize(loss)

def render_4d_gaussians(gaussians, camera_pose, time_t, deformation_field, canonical_space):
    """
    Renders a frame at time t using 4D Gaussian Splatting.
    
    - Transforms canonical Gaussians to world space using deformation_field.
    - Applies time-varying scaling and 4D SH.
    """
    # Get time-varying parameters
    # Position: mu_t = canonical_mu + deformation_field(canonical_mu, time_t)
    # Rotation: q_t (time-varying)
    # Scaling: S_t (diagonal matrix with time scaling factor)
    # Color: C_t (4D SH: 3D SH combined with 1D time basis)
    
    world_gaussians = []
    for g in gaussians:
        # Apply deformation
        t_mu = g.canonical_mu + deformation_field(g.canonical_mu, time_t)
        
        # Construct 4D scaling matrix
        # S_4d = diag(S_3d, S_time)
        S_4d = construct_4d_scaling_matrix(g.scale_3d, g.scale_time)
        
        world_gaussians.append(Gaussian(
            mu=t_mu,
            q=g.q_t, # Time-varying rotation
            S=S_4d,
            alpha=g.alpha,
            sh_4d=g.sh_4d # 4D spherical harmonics
        ))
        
    return rasterize(world_gaussians, camera_pose)
```

## Operating characteristics
*   **Complexity**: The method introduces overhead due to training multiple models and managing window boundaries. Memory usage scales with the number of windows and the size of each window's Gaussian set.
*   **Strengths**: By isolating motion into windows, SWAGS can handle scenes with varying motion intensities more effectively than global deformation models. The focus of the MLP on dynamic parts allows for more efficient representation of motion.
*   **Limitations**: Requires careful selection of window boundaries based on motion. Temporal consistency relies on the fine-tuning step and overlapping frames; poor overlap or insufficient fine-tuning can lead to discontinuities.
*   **Performance**: Not given in the sources.

## Background
SWAGS was introduced as a method to improve dynamic 3D Gaussian Splatting by addressing the limitations of global deformation fields in handling complex motion [S1][S2, unofficial]. It builds on the parent method [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md) by introducing a window-based approach [S1][S2, unofficial]. The method has been applied in medical fields, such as markerless motion reconstruction for infants and neonates, where additional mask and depth supervisions are used [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
