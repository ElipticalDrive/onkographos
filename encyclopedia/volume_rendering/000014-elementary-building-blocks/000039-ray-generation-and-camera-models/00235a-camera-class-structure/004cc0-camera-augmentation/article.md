# Camera Augmentation
Also called: Intrinsic/Extrinsic perturbation, Perspective Augmentation

## Overview
Camera augmentation perturbs camera parameters during training to improve robustness against small image misalignments and promote diversity in generated views [S1, S2]. It deviates from the fixed camera assumptions of EG3D by introducing stochastic variations in intrinsic and extrinsic properties [S1]. While EG3D assumes a fixed camera radius of 2.7, focal length of 18.83, zero roll, and a central principal point, this method samples these values from specific distributions [S1].

## How it works
**Inputs:** Base camera parameters (focal length $f$, radius $r$, principal point $(c_x, c_y)$, roll angle $\theta_{\text{roll}}$), dataset type (FFHQ or AFHQ), current training epoch $e$, and maximum schedule epoch $E_{\text{max}}$ (typically 100) [S1, S2].
**Outputs:** Perturbed camera parameters used for ray generation [S1, S2].

**Data Structures:**
The method modifies the following scalar parameters:
- $f$: Focal length (float) [S1].
- $r$: Camera radius (float) [S1].
- $c_x, c_y$: Principal point coordinates (float) [S1].
- $\theta_{\text{roll}}$: Camera roll angle in degrees (float) [S1].
- $\theta_{\text{persp}}$: Perspective augmentation angle in degrees (float) [S2].

**Steps:**

1. **Determine Augmentation Schedule (Perspective Augmentation):**
   Calculate the current maximum angle range based on the training epoch [S2]. The range expands linearly from $[0^\circ, 0^\circ]$ to $[-15^\circ, 15^\circ]$ over 100 epochs [S2].
   $$ \theta_{\max}(e) = 15^\circ \times \min\left(1, \frac{e}{E_{\text{max}}}\right) $$
   Randomly and independently sample an angle $\theta_{\text{persp}}$ from the current range $[-\theta_{\max}, \theta_{\max}]$ [S2]. This angle is applied as a perspective distortion along the height axis [S2]. The corresponding rotation matrix $R_z$ is defined as:
   $$ R_z = \begin{pmatrix} \cos \theta_z & -\sin \theta_z & 0 \\ \sin \theta_z & \cos \theta_z & 0 \\ 0 & 0 & 1 \end{pmatrix} $$
   where $\theta_z$ denotes the sampled perspective angle [S2]. Direct optimization of Euler angles is avoided in SinGRAF due to discontinuities at $\theta=0$; instead, cosine and sine values are carried for rotation parameterization [S2].

2. **Sample Intrinsic and Extrinsic Perturbations:**
   For each training step, sample the following parameters from normal distributions [S1]. The parameters depend on the dataset [S1].
   **For FFHQ:**
   - Focal length $f \sim \mathcal{N}(18.83, 1.0^2)$ [S1].
   - Camera radius $r \sim \mathcal{N}(2.7, 0.1^2)$ [S1].
   - Principal point $c_x, c_y \sim \mathcal{N}(256, 14^2)$ [S1].
   - Camera roll $\theta_{\text{roll}} \sim \mathcal{N}(0^\circ, 2^\circ)$ [S1].
   **For AFHQ:**
   - Focal length $f \sim \mathcal{N}(18.83, 1.5^2)$ [S1].
   - Camera radius $r \sim \mathcal{N}(2.7, 0.1^2)$ [S1].
   - Principal point $c_x, c_y \sim \mathcal{N}(256, 25^2)$ [S1].
   - Camera roll $\theta_{\text{roll}} \sim \mathcal{N}(0^\circ, 6^\circ)$ [S1].

3. **Apply Perturbations:**
   Combine the sampled parameters to form the augmented camera matrix [S1]. Apply the perspective distortion along the height axis using the sampled angle [S2]. The final camera pose and intrinsics are then used for ray generation [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_camera_augmentation(dataset_type: str, epoch: int, max_epochs: int = 100) -> dict:
    """
    Samples augmented camera parameters based on dataset and training progress.
    
    Args:
        dataset_type: 'FFHQ' or 'AFHQ'
        epoch: Current training epoch
        max_epochs: Total epochs for linear schedule (default 100)
        
    Returns:
        dict with keys: 'focal_length', 'radius', 'principal_point', 'roll', 'perspective_angle'
    """
    base_focal = 18.83
    base_radius = 2.7
    base_principal = 256.0
    
    if dataset_type == 'FFHQ':
        std_focal = 1.0
        std_radius = 0.1
        std_principal = 14.0
        std_roll = 2.0
    elif dataset_type == 'AFHQ':
        std_focal = 1.5
        std_radius = 0.1
        std_principal = 25.0
        std_roll = 6.0
    else:
        raise ValueError("Unknown dataset type")

    focal_length = random.gauss(base_focal, std_focal)
    radius = random.gauss(base_radius, std_radius)
    principal_point = random.gauss(base_principal, std_principal)
    roll = random.gauss(0.0, std_roll)

    current_max_angle = 15.0 * min(1.0, epoch / max_epochs)
    perspective_angle = random.uniform(-current_max_angle, current_max_angle)

    return {
        'focal_length': focal_length,
        'radius': radius,
        'principal_point': principal_point,
        'roll': roll,
        'perspective_angle': perspective_angle
    }
```

**Edge Cases and Pitfalls:**
- The linear schedule for perspective augmentation assumes a fixed `max_epochs` of 100 [S2]. If training exceeds this, the angle remains capped at $15^\circ$ [S2].
- Principal point sampling is centered at 256 [S1]. Not given in the sources.
- Direct optimization of Euler angles is avoided in SinGRAF due to discontinuities at $\theta=0$ [S2].

## Subtopics

### [Pose Augmentation for FFHQ](20236-pose-augmentation-for-ffhq.md)
A training technique that perturbs camera poses during the generation of 3D-aware models to address the lack of capability to model precise 3D structures from different vertical angles. It sets itself apart from standard fixed-camera training by explicitly oversampling vertical perspectives, ensuring better coverage of varied vertical perspectives than the base dataset distribution. Choose it when vertical angle coverage is critical for the target dataset.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Improves robustness to misalignment and increases view diversity without requiring additional data [S1, S2].
- **Limitations:** Not given in the sources.
- **When to use:** Use when training 3D-aware generative models on single images where ground-truth multi-view data is unavailable [S1, S2].
- **When not to use:** Not given in the sources.

## Background
Camera augmentation was introduced to address the limitations of fixed camera models in generative radiance fields, specifically EG3D [S1]. It was adopted in Real-Time Radiance Fields for Single-Image Portrait View Synthesis to handle real image misalignments [S1]. SinGRAF also employs a similar perspective augmentation technique to promote diversity in 3D generative radiance fields [S2]. The technique is particularly relevant for single-image view synthesis where ground-truth multi-view data is unavailable [S1, S2].

## Sources
[S1] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
[S2] SinGRAF: Learning a 3D Generative Radiance Field for a Single Scene, https://doi.org/10.1109/cvpr52729.2023.00822, fetched 2026-10-29
