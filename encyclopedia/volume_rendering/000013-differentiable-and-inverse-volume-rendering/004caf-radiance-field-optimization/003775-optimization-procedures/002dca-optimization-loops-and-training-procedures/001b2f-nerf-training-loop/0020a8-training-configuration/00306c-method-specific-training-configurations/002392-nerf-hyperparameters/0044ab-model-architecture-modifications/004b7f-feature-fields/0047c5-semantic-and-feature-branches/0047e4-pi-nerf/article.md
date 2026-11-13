# PI-NeRF

Also called: Partial-Invertible Neural Radiance Fields

## Overview
PI-NeRF is a framework that integrates NeRF with invertible neural networks (INN) to directly output the 6-DoF pose of a given image without pose initialization or iterative optimization, solving the inverse problem of view synthesis [S1]. It differs from existing NeRF-based pose estimation methods (like iNeRF and pNeRF) by directly outputting the pose without iterative optimization or initial pose, whereas existing methods rely on synthesizing images from a number of poses or iterative optimization, which is time-consuming and sensitive to initial poses [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single RGB image $I$ captured by a camera with known intrinsic parameters $K$.
*   **Output:** The 6-DoF camera pose $(R, t)$, where $R$ is the rotation matrix and $t$ is the translation vector, relative to the 3D scene represented by the NeRF.

**Data Structures:**
1.  **NeRF-Feat:** A modified NeRF model. Instead of outputting RGB color and density, it outputs a high-dimensional feature vector $f \in \mathbb{R}^D$ and density $\sigma$.
    *   **MLP:** Maps 3D position $x$ and view direction $d$ to feature $f$ and density $\sigma$.
    *   **Volume Rendering:** Aggregates features along the ray.
2.  **Invertible Neural Network Module (INM):** Consists of two Invertible Neural Networks (INNs) operating in forward and reverse directions.
    *   **Forward Path:** Input: Ray feature $f_r$ and ray representation $r_{rep}$. Output: Pixel feature $f_p$.
    *   **Reverse Path:** Input: Pixel feature $f_p$. Output: Ray feature $f_r$ and ray representation $r_{rep}$.
3.  **ResUNet:** A convolutional neural network used to extract pixel features from the input image.

**Numbered Steps:**

1.  **NeRF-Feat Construction:**
    Modify the standard NeRF architecture [S1]. Use an MLP to map each 3D point $x_i$ (sampled along a ray) to a high-dimensional feature vector $f(x_i)$ instead of RGB color. Apply the volume rendering function as a weighted feature aggregation method along the ray to obtain a high-dimensional feature $f_r$ for each ray:
    $$
    f_r = \sum_{i=1}^{N} T_i \alpha_i f(x_i)
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\alpha_i = (1 - \exp(-\sigma_i \delta_i))$ is the opacity weight [S1]. This modified NeRF is called NeRF-Feat.

2.  **Invertible Neural Network Module (INM) Design:**
    Design an INM consisting of two INNs that can input and output in two directions [S1].
    *   **Forward Path:** Takes the ray feature $f_r$ from NeRF-Feat and a representation of the ray $r_{rep}$ (derived from the camera pose) as input, and outputs a pixel feature $f_p$.
    *   **Reverse Path:** If the pixel feature $f_p$ is fed in, it recovers the corresponding ray feature $f_r$ and ray representation $r_{rep}$.

3.  **Training Phase:**
    *   Use ResUNet to extract pixel features $f_p^{gt}$ for the given observed image $I$ [S1].
    *   Feed these pixel features $f_p^{gt}$ into the reverse path of INM to recover the ray representation $r_{rep}^{rec}$ and ray feature $f_r^{rec}$.
    *   Use the recovered ray representation $r_{rep}^{rec}$ to generate synthetic ray features via NeRF-Feat (forward pass of NeRF-Feat) to get $f_r^{syn}$.
    *   Feed $f_r^{syn}$ into the forward path of INM to get reconstructed pixel features $f_p^{rec}$.
    *   Compute the Barlow Twins (BT) loss between the corresponding pixel features produced by INM in the forward path ($f_p^{rec}$) and the features extracted by ResUNet ($f_p^{gt}$) [S1]. This ensures that the ResUNet learns to generate features suitable to be fed into INM in the reverse path to recover the ray representations.

4.  **Inference Phase:**
    *   Use ResUNet to extract a feature map from the observed image $I$ [S1].
    *   Sample a set of features from the feature map and feed them into INM in the reverse path to obtain ray representations $r_{rep}$ [S1].
    *   Based on these ray representations, directly use the Perspective-n-Point (PnP) algorithm to calculate the camera pose $(R, t)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class PINeRF:
    def __init__(self, nerf_feat, inm, resunet, camera_intrinsics):
        """
        Initialize PI-NeRF components.
        :param nerf_feat: NeRF-Feat model (MLP for feature/density + volume renderer)
        :param inm: Invertible Neural Network Module (forward and reverse paths)
        :param resunet: ResUNet model for pixel feature extraction
        :param camera_intrinsics: Camera intrinsic matrix K
        """
        self.nerf_feat = nerf_feat
        self.inm = inm
        self.resunet = resunet
        self.K = camera_intrinsics

    def train_step(self, image):
        """
        Perform one training step.
        :param image: Input RGB image
        :return: Barlow Twins loss
        """
        # 1. Extract pixel features from image using ResUNet
        f_p_gt = self.resunet(image)
        
        # 2. Feed pixel features into reverse path of INM to recover ray representation
        # Note: The source states feeding pixel features into reverse path recovers 
        # ray feature and ray representation. We assume the reverse path takes f_p and outputs (f_r, r_rep).
        f_r_rec, r_rep_rec = self.inm.reverse_path(f_p_gt)
        
        # 3. Use recovered ray representation to generate synthetic ray features via NeRF-Feat
        # This involves sampling rays based on r_rep_rec and rendering features
        f_r_syn = self.nerf_feat.render_ray_features(r_rep_rec)
        
        # 4. Feed synthetic ray features into forward path of INM to get reconstructed pixel features
        f_p_rec = self.inm.forward_path(f_r_syn, r_rep_rec)
        
        # 5. Compute Barlow Twins loss between f_p_rec and f_p_gt
        loss = barlow_twins_loss(f_p_rec, f_p_gt)
        
        return loss

    def infer_pose(self, image):
        """
        Estimate camera pose from a single image.
        :param image: Input RGB image
        :return: Camera pose (R, t)
        """
        # 1. Extract pixel features from image using ResUNet
        f_p = self.resunet(image)
        
        # 2. Sample a set of features from the feature map
        # Note: The source mentions sampling a set of features. This likely refers to 
        # selecting specific pixel locations or aggregating features to form ray representations.
        # For implementation, we assume we sample N pixel features corresponding to N 3D points.
        sampled_f_p = sample_features(f_p, num_samples=1000)
        
        # 3. Feed sampled features into INM in reverse path to obtain ray representations
        # This recovers the ray representations (which encode the camera pose)
        r_rep_list = []
        for f_p_sample in sampled_f_p:
            _, r_rep = self.inm.reverse_path(f_p_sample)
            r_rep_list.append(r_rep)
            
        # 4. Use PnP algorithm to calculate camera pose based on ray representations
        # The ray representations contain information about the 3D points and their corresponding 2D projections.
        # We need to extract 3D-2D correspondences from r_rep_list and use PnP.
        # Note: The exact mapping from r_rep to 3D-2D points is not detailed in the source,
        # but PnP requires 3D points (from NeRF scene) and 2D points (from image).
        # The ray representation likely encodes the direction or the 3D point location.
        # Assuming r_rep allows reconstruction of 3D points and their 2D projections:
        points_3d, points_2d = extract_3d_2d_correspondences(r_rep_list, self.nerf_feat, self.K)
        
        # 5. Solve PnP
        R, t = solve_pnp(points_3d, points_2d, self.K)
        
        return R, t

def barlow_twins_loss(f_p_rec, f_p_gt):
    """
    Compute Barlow Twins loss.
    :param f_p_rec: Reconstructed pixel features
    :param f_p_gt: Ground truth pixel features from ResUNet
    :return: Loss value
    """
    # Standard Barlow Twins implementation:
    # 1. Normalize features
    f_p_rec_norm = normalize(f_p_rec)
    f_p_gt_norm = normalize(f_p_gt)
    
    # 2. Compute cross-correlation matrix
    batch_size = f_p_rec_norm.shape[0]
    c = torch.mm(f_p_rec_norm.T, f_p_gt_norm) / batch_size
    
    # 3. Compute loss
    # Diagonal elements should be 1, off-diagonal elements should be 0
    b = torch.diag(c)
    loss = torch.sum(b) + torch.sum((c - torch.eye(c.shape[0])) ** 2)
    return loss

def sample_features(feature_map, num_samples):
    """
    Sample a set of features from the feature map.
    :param feature_map: Output of ResUNet
    :param num_samples: Number of features to sample
    :return: Sampled features
    """
    # Implementation detail: Sample random pixel locations or use a grid
    # This is a placeholder for the specific sampling strategy not detailed in the source
    # [G] Assuming random sampling of pixel locations
    h, w, d = feature_map.shape
    indices = np.random.randint(0, h, size=num_samples)
    j_indices = np.random.randint(0, w, size=num_samples)
    sampled_features = feature_map[indices, j_indices, :]
    return sampled_features

def extract_3d_2d_correspondences(r_rep_list, nerf_feat, K):
    """
    Extract 3D-2D correspondences from ray representations.
    :param r_rep_list: List of ray representations
    :param nerf_feat: NeRF-Feat model
    :param K: Camera intrinsic matrix
    :return: points_3d, points_2d
    """
    # Implementation detail: The source does not specify how r_rep maps to 3D-2D points.
    # [G] Assuming r_rep contains the 3D point location and the 2D projection can be derived
    # or the ray direction can be used with depth to find the 3D point.
    # This is a placeholder for the specific mapping strategy not detailed in the source.
    points_3d = []
    points_2d = []
    for r_rep in r_rep_list:
        # Placeholder: Extract 3D point and 2D projection from r_rep
        # 3D point might be obtained by intersecting the ray (from r_rep) with the NeRF scene
        # 2D point is the pixel location corresponding to the feature
        point_3d = get_3d_point_from_r_rep(r_rep, nerf_feat)
        point_2d = get_2d_point_from_r_rep(r_rep, K)
        points_3d.append(point_3d)
        points_2d.append(point_2d)
    return np.array(points_3d), np.array(points_2d)

def get_3d_point_from_r_rep(r_rep, nerf_feat):
    # Placeholder implementation
    # [G] This function needs to be implemented based on the specific structure of r_rep
    pass

def get_2d_point_from_r_rep(r_rep, K):
    # Placeholder implementation
    # [G] This function needs to be implemented based on the specific structure of r_rep
    pass

def solve_pnp(points_3d, points_2d, K):
    """
    Solve Perspective-n-Point problem.
    :param points_3d: 3D points
    :param points_2d: 2D points
    :param K: Camera intrinsic matrix
    :return: R, t
    """
    # Use OpenCV's solvePnP or similar
    # [G] Standard PnP solver
    pass
```

**Parameters with typical values:**
*   **High-dimensional feature size $D$:** Not given in the sources.
*   **Number of samples for PnP:** Not given in the sources.
*   **ResUNet:** Used for pixel feature extraction [S1].

**Edge cases and pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate input:** Not given in the sources.

## Operating characteristics
*   **Time complexity:** PI-NeRF is two orders of magnitude faster than existing NeRF-based methods (e.g., pNeRF takes more than 5 seconds, iNeRF is slower) [S1].
*   **Memory complexity:** Not given in the sources.
*   **Measured performance:** Experiments conducted on both synthetic and real-world datasets demonstrate that PI-NeRF is two orders of magnitude faster than existing NeRF-based methods, while the accuracy is competitive without initial pose [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Direct pose estimation without iterative optimization or initial pose; fast inference speed; competitive accuracy [S1].
*   **Limitations:** Relies on the accuracy of the ResUNet and INM; requires a pre-trained NeRF-Feat model [S1].
*   **When to use it:** When fast, accurate camera pose estimation is needed from a single image without prior pose knowledge [S1].
*   **When not:** When iterative optimization is acceptable and higher accuracy is required [S1].

## Background
*   **Origin:** PI-NeRF was proposed by Zhihao Li, Kexue Fu, Haoran Wang, & Manning Wang in the paper "PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation" published in MM '23, October 29-November 3, 2023, Ottawa, ON, Canada [S1].
*   **History and development:** The method builds on the idea of inverting NeRF for pose estimation, which was previously not successfully achieved without iterative optimization or initial pose [S1]. It integrates NeRF with invertible neural networks (INN), which have been used to solve many inverse problems such as image rescaling and robot localization [S1].
*   **Where it is used:** Camera pose estimation, which is the basis of many tasks including robot manipulation and simultaneous localization and mapping [S1].

## Sources
[S1] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
