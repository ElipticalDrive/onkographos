# General Radiance Field (GRF)


## Overview
General Radiance Field (GRF) is a neural rendering method that learns a 3D scene representation by extracting local features from 2D input images and projecting them into a canonical 3D space [S1, S2]. It operates in canonical space rather than view-space, distinguishing it from methods like [Generalizable Radiance Fields](9480-generalizable-radiance-fields.md) [S1]. The network models 3D geometry and appearance by constructing an internal representation for each point in 3D space and rendering appearance and geometry from an arbitrary position [S2].

## How it works
### Inputs and Outputs
**Inputs:**
1. A set of $M$ input images with their corresponding camera poses and intrinsics: $\{(I_1, \xi_1, K_1), \dots, (I_M, \xi_M, K_M)\}$, where $I_m$ is the image, $\xi_m$ is the pose, and $K_m$ is the intrinsic matrix [S2].
2. A query 3D point $p = \{x_p, y_p, z_p\}$ in canonical space [S2].
3. A query viewpoint $V_p = \{x_{vp}, y_{pv}, z_{pv}\}$ representing the camera position from which the point is observed [S2].

**Outputs:**
1. Predicted RGB color values $\{r_p, g_p, b_p\}$ [S2].
2. Predicted volumetric density $d_p$ [S2].

### Data Structures
The method utilizes a neural network architecture composed of four distinct components [S2]:
1. **Feature Extractor:** A Convolutional Neural Network (CNN) with an encoder-decoder structure that processes 2D images to extract pixel-level features [S2].
2. **Reprojector:** A geometric module that transforms 2D features extracted from input images into the 3D canonical space corresponding to the query point $p$ [S2].
3. **Aggregator:** A module that combines features reprojected from multiple input views into a single general feature vector for point $p$ [S2].
4. **Neural Renderer:** A Multi-Layer Perceptron (MLP) based on the NeRF architecture that maps the aggregated general features and viewpoint information to density and color [S2].

### Steps
1. **Preprocessing Input Images:** For each input image $I_m$, the camera's 3D location (xyz coordinates) is stacked (duplicated) to each pixel of the image [S2]. This allows the feature extractor to be explicitly aware of the pixel's relative position in 3D space [S2].
2. **Feature Extraction:** The preprocessed images are passed through the shared CNN feature extractor [S2]. Skip connections between the encoder and decoder are used to preserve high-frequency local features, while fully connected layers in the middle learn global features [S2]. This produces a set of 2D feature maps, one for each input image [S2].
3. **Reprojection:** For a given query point $p$, the 2D features from each input image $I_m$ are reprojected back to the 3D location of $p$ [S2]. This aligns the features from different viewpoints to the same canonical coordinate [S2].
4. **Aggregation:** The reprojected features from all $M$ views are aggregated [S2]. An attention mechanism is integrated to handle visual occlusions implicitly, weighting features from visible views higher than those from occluded views [S2]. This results in a single general feature vector for point $p$ [S2].
5. **Radiance Field Query:** The aggregated general feature vector, along with the query viewpoint $V_p$, is fed into the NeRF MLP [S2]. The MLP predicts the density $d_p$ and color $\{r_p, g_p, b_p\}$ for point $p$ [S2].
6. **Volume Rendering:** To render a final pixel color for a camera ray, multiple points along the ray are sampled [S2]. The volume rendering equation is used to integrate the predicted densities and colors of these points to obtain the final pixel RGB value [S2].

### Formula
The network $f_W$ with learnable parameters $W$ is defined as [S2]:

$$
(r_p, g_p, b_p, d_p) = f_W \left( (I_1, \xi_1, K_1), \dots, (I_M, \xi_M, K_M), (x_p, y_p, z_p), (x_{vp}, y_{pv}, z_{pv}) \right)
$$

Where:
* $(r_p, g_p, b_p, d_p)$ are the predicted RGB value and volumetric density of point $p$ observed from viewpoint $V_p$ [S2].
* $f_W$ is the network with learnable parameters $W$ [S2].
* $(I_m, \xi_m, K_m)$ are the $m$-th image, pose, and intrinsics [S2].
* $(x_p, y_p, z_p)$ are the coordinates of the query point $p$ [S2].
* $(x_{vp}, y_{pv}, z_{pv})$ are the coordinates of the query viewpoint $V_p$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class GRF:
    def __init__(self, num_images, feature_extractor, aggregator, renderer):
        self.num_images = num_images
        self.feature_extractor = feature_extractor
        self.aggregator = aggregator
        self.renderer = renderer

    def extract_2d_features(self, images, camera_positions):
        # images: [M, H, W, 3], camera_positions: [M, 3]
        # Stack camera position to each pixel
        cam_pos_expanded = np.expand_dims(np.expand_dims(camera_positions, axis=1), axis=1)
        cam_pos_stacked = np.broadcast_to(cam_pos_expanded, (images.shape[0], images.shape[1], images.shape[2], 3))
        input_tensor = np.concatenate([images, cam_pos_stacked], axis=-1)
        return self.feature_extractor(input_tensor)

    def reproject_features(self, features_2d, query_points, camera_poses, K_matrices):
        # features_2d: [M, H, W, C]
        # query_points: [N, 3]
        # camera_poses: [M, 4, 4]
        # K_matrices: [M, 3, 3]
        N = query_points.shape[0]
        M = features_2d.shape[0]
        C = features_2d.shape[-1]
        reprojected = np.zeros((M, N, C))

        for m in range(M):
            cam_pose_inv = np.linalg.inv(camera_poses[m])
            p_hom = np.hstack([query_points, np.ones((N, 1))])
            p_cam_hom = (cam_pose_inv @ p_hom.T).T
            p_cam = p_cam_hom[:, :3]

            K = K_matrices[m]
            p_cam_proj = (K @ p_cam.T).T
            u = p_cam_proj[:, 0] / p_cam_proj[:, 2]
            v = p_cam_proj[:, 1] / p_cam_proj[:, 2]
            z = p_cam_proj[:, 2]

            u_norm = (u / (features_2d.shape[2] - 1)) * 2 - 1
            v_norm = (v / (features_2d.shape[1] - 1)) * 2 - 1
            grid = np.stack([u_norm, v_norm], axis=-1).reshape(1, 1, N, 2)
            feat_m = features_2d[m].reshape(1, features_2d.shape[1], features_2d.shape[2], C)
            sampled = np.squeeze(grid_sample(feat_m, grid))
            reprojected[m] = sampled.T

        return reprojected

    def aggregate_features(self, reprojected_features, query_points):
        # reprojected_features: [M, N, C]
        # query_points: [N, 3]
        return self.aggregator(reprojected_features, query_points)

    def query_radiance_field(self, aggregated_features, query_points, query_viewpoints):
        # aggregated_features: [N, C_agg]
        # query_points: [N, 3]
        # query_viewpoints: [N, 3]
        input_data = np.concatenate([aggregated_features, query_points, query_viewpoints], axis=-1)
        output = self.renderer(input_data)
        color = output[:, :3]
        density = output[:, 3:4]
        return color, density

    def render(self, images, camera_positions, camera_poses, K_matrices, rays_origins, rays_directions, num_samples):
        # rays_origins: [H, W, 3], rays_directions: [H, W, 3]
        t_vals = np.linspace(0.0, 1.0, num_samples)
        points = rays_origins[:, :, np.newaxis, :] + rays_directions[:, :, np.newaxis, :] * t_vals[np.newaxis, np.newaxis, :, np.newaxis]
        H, W, N = points.shape[:3]
        points_flat = points.reshape(-1, N, 3)

        features_2d = self.extract_2d_features(images, camera_positions)
        reprojected = self.reproject_features(features_2d, points_flat, camera_poses, K_matrices)
        aggregated = self.aggregate_features(reprojected, points_flat)

        viewpoints_flat = np.broadcast_to(camera_positions[np.newaxis, :, :], (H*W, N, 3)).reshape(-1, 3)
        color, density = self.query_radiance_field(aggregated, points_flat, viewpoints_flat)

        deltas = np.diff(t_vals)
        delta = np.concatenate([deltas, [1e10]])
        delta = np.broadcast_to(delta[np.newaxis, np.newaxis, :], (H, W, N)).reshape(-1, N)

        alpha = 1.0 - np.exp(-density * delta)
        T = np.cumprod(1.0 - alpha + 1e-10, axis=1)
        T = np.concatenate([np.ones_like(T[:, :1]), T[:, :-1]], axis=1)

        weights = alpha * T
        rendered_color = np.sum(weights[:, :, np.newaxis] * color, axis=1)
        return rendered_color.reshape(H, W, 3)

def grid_sample(input_tensor, grid):
    # Standard bilinear sampling implementation omitted for brevity; 
    # assumes standard library or framework implementation [G].
    pass
```

### Parameters
* **Number of Input Images ($M$):** Not specified in the sources [S2].
* **CNN Architecture:** Details are in the appendix of the source paper; not specified here [S2].
* **MLP Architecture:** Based on NeRF; specific layer sizes not given in the provided notes [S2].

### Edge Cases and Pitfalls
* **Lighting Sensitivity:** Using raw RGB values as pixel features is sub-optimal because they are sensitive to lighting conditions and environmental noise [S2].
* **Camera Intrinsics/Rotations:** Stacking additional camera rotation and intrinsics to each pixel does not noticeably improve performance [S2].
* **Occlusions:** The attention mechanism in the aggregator implicitly handles visual occlusions by weighting features from visible views [S2].

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** Not given in the sources.
* **Performance:** Extensive experiments demonstrate the effectiveness of the approach, but specific metrics are not provided in the notes [S2].
* **Strengths:** Operates in canonical space, allowing for generalizable features across views [S1]. The use of CNNs for feature extraction provides robustness to lighting variations [S2].
* **Limitations:** The method relies on accurate camera poses and intrinsics [S2]. The performance of the CNN feature extractor is critical and its optimal design is not detailed in the source [S2].

## Background
General Radiance Field (GRF) was introduced by Alex Trevithick and Bo Yang in October 2020 [S1, S2]. It is similar to pixelNeRF but operates in canonical space rather than view-space [S1]. The method models 3D geometries as a general radiance field, taking 2D images with camera poses and intrinsics as input, constructing an internal representation for each point of the 3D space, and rendering appearance and geometry from an arbitrary position [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
