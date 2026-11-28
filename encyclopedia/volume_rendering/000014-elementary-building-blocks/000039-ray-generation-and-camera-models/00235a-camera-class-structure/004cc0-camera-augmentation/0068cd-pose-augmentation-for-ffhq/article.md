# Pose Augmentation for FFHQ
Also called: Pose Distribution Augmentation
## Overview
A training technique that perturbs camera poses during the generation of 3D-aware models to address the lack of capability to model precise 3D structures from different vertical angles [S1]. It sets itself apart from standard fixed-camera training by explicitly oversampling vertical perspectives, ensuring better coverage of varied vertical perspectives than the base dataset distribution [S1].
## How it works
**Inputs and Outputs:**
- Input: The standard FFHQ dataset pose statistics (standard deviations for yaw and pitch).
- Output: A modified training loop where camera poses are sampled from an augmented distribution with probability 0.5.

**Data Structures:**
- Pose parameters: Yaw ($\theta_y$) and Pitch ($\theta_p$).
- Distribution parameters: $\sigma_{yaw}$ and $\sigma_{pitch}$, derived from the FFHQ dataset statistics.

**Steps:**
1. During training, for each batch or sample, generate a random probability $p \in [0, 1]$.
2. If $p < 0.5$, sample the pose from the augmented distribution defined below. Otherwise, use the original dataset pose distribution.
3. Sample the yaw angle $\theta_y$ from a Normal distribution:
   $$ \theta_y \sim \mathcal{N}(0, \sigma_{yaw}) $$
4. Sample the pitch angle $\theta_p$ from a Uniform distribution:
   $$ \theta_p \sim \mathcal{U}(-3\sigma_{pitch}, 3\sigma_{pitch}) $$
5. Construct the camera pose using the sampled $\theta_y$ and $\theta_p$.

**Parameters:**
- Probability of augmentation: 0.5 [S1].
- Yaw distribution: Normal $\mathcal{N}(0, \sigma_{yaw})$ [S1].
- Pitch distribution: Uniform $\mathcal{U}(-3\sigma_{pitch}, 3\sigma_{pitch})$ [S1].
- Typical range for multi-view synthesis evaluation: Yaw in $[-0.4, 0.4]$ radians and Pitch in $[-0.4, 0.1]$ radians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_augmented_pose(sigma_yaw, sigma_pitch):
    """
    Samples a pose from the augmented distribution.
    
    Args:
        sigma_yaw (float): Standard deviation of yaw from FFHQ dataset.
        sigma_pitch (float): Standard deviation of pitch from FFHQ dataset.
        
    Returns:
        tuple: (yaw, pitch) sampled from the augmented distribution.
    """
    # Sample yaw from Normal distribution N(0, sigma_yaw)
    yaw = np.random.normal(loc=0.0, scale=sigma_yaw)
    
    # Sample pitch from Uniform distribution U(-3*sigma_pitch, 3*sigma_pitch)
    lower_bound = -3.0 * sigma_pitch
    upper_bound = 3.0 * sigma_pitch
    pitch = np.random.uniform(low=lower_bound, high=upper_bound)
    
    return yaw, pitch

def get_training_pose(sigma_yaw, sigma_pitch):
    """
    Determines the pose for a training step based on augmentation probability.
    
    Args:
        sigma_yaw (float): Standard deviation of yaw from FFHQ dataset.
        sigma_pitch (float): Standard deviation of pitch from FFHQ dataset.
        
    Returns:
        tuple: (yaw, pitch) for the current training step.
    """
    # Probability of augmentation is 0.5
    if np.random.random() < 0.5:
        return sample_augmented_pose(sigma_yaw, sigma_pitch)
    else:
        # Return original dataset pose distribution
        # Note: The specific implementation of the original distribution is not given in the sources.
        # This is a placeholder for the standard FFHQ pose sampling logic.
        pass
```

**Edge Cases and Pitfalls:**
- The sources do not specify how the original dataset pose distribution is sampled when augmentation is not applied [S1].
- The uniform distribution for pitch extends to $\pm 3\sigma_{pitch}$, which may exceed typical natural head rotation ranges if $\sigma_{pitch}$ is large, potentially leading to unrealistic poses if not clamped [S1, G].

## Subtopics

### 3D-Aware Adaptive Discriminator Augmentation
A modification of Adaptive Discriminator Augmentation (ADA) where both the input image and the camera pose are augmented with the same geometric transformation (scale, translation, rotation), and the discriminator is conditioned on the augmented pose [S2]. It is chosen to prevent artifacts caused by 2D-only augmentation in 3D-aware models [S2]. Standard ADA is not 3D-aware because augmentations are only carried out in image space while the discriminator is conditioned on the original camera pose, leading to artifacts such as objects appearing concave [S2].

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Measured Performance:** The method ensures better coverage of images with varied vertical perspectives [S1].
- **Scaling:** Not given in the sources.
- **Strengths:** Addresses the lack of capability to model precise 3D structures from different vertical angles [S1].
- **Limitations:** Not given in the sources.
- **When to use:** When training on the FFHQ dataset to improve 3D consistency in vertical perspectives [S1].
- **When not to use:** Not given in the sources.

## Background
Originated in GSGAN (Gaussian Splatting GAN) for the FFHQ dataset [S1]. The authors observed that training on FFHQ lacked the capability to accurately model precise structures of 3D scenes, particularly for structures viewed from different vertical angles [S1]. The method was introduced as a training trick to address this specific limitation [S1].

## Sources
[S1] GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats, https://doi.org/10.52202/079017-2171, fetched 2026-10-29
[S2] Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion, https://doi.org/10.1109/cvpr52729.2023.00427, fetched 2026-10-26
