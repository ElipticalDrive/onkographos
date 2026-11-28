# Point Feature Generation


## Overview
Point Feature Generation generates features for ray points by indexing volume features and building color volumes from images [S1, unofficial]. It associates geometric and appearance attributes with points along a camera ray by splitting input features into geometric channels and color channels derived from multi-view images [S1, unofficial].

## How it works
**Inputs:**
- `view_ids`: List of view identifiers.
- `data_dir`: Directory containing rectified images and depth maps.
- `scan`: Scan identifier.
- `light_idx`: Lighting condition index.
- `downSample`: Downsampling factor for image resolution.
- `proj_mats`, `intrinsics`, `w2cs`, `c2ws`: Pre-computed projection matrices, camera intrinsics, and world-to-camera / camera-to-world transforms.
- `remap`: Mapping from view IDs to index matrices.
- `use_color_volume`: Boolean flag indicating whether to build a color volume.
- `img_feat`: Optional 2D image feature tensor.
- `base_feat_dim`: Base feature dimension, typically 8 [S1, unofficial].
- `num_ray_points`: Number of points along each ray [G].

**Outputs:**
- `ray_feats`: Feature vector assigned to each ray point.
- `color_volume`: (Optional) 3D volume of color data indexed from images.
- `input_feat`: Initial feature tensor containing geometric and color data.

**Data Structures:**
- `input_feat`: Tensor with channels split into geometric features (first 8 channels) and color features (remaining channels) [S1, unofficial].
- `ray_feats`: Tensor of shape $(N, 8)$, where $N$ is the number of ray points [G].
- `color_volume`: 3D grid storing color values indexed by spatial coordinates [G].
- `feat_dim`: Base feature dimension, incremented by `img_feat` spatial dimensions if provided [S1, unofficial].

**Steps:**
1. **Initialize Image Data:** For each view ID in `view_ids`: construct the image filename using 1-based indexing (`vid + 1`), load the image, resize it to `img_wh = round(original_size * downSample)` using bilinear interpolation, apply standard normalization, and extract the projection matrix and near/far planes using the `remap` index [S1, unofficial]. Store the affine matrix and its inverse [S1, unofficial].
2. **Determine Feature Dimensions:** Set `feat_dim` to `base_feat_dim`. If `img_feat` is provided, add its spatial dimensions to `feat_dim` [S1, unofficial].
3. **Initialize Input Features:** If not using a color volume, initialize `input_feat` and index volume features [S1, unofficial].
4. **Assign Features to Ray Points:** Assign the first 8 channels of `input_feat` to `ray_feats` and the remaining channels to the color volume built from images [S1, unofficial].
5. **Indexing:** If using a color volume, directly index volume features [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_point_features(view_ids, data_dir, scan, light_idx, downSample,
                            proj_mats, intrinsics, w2cs, c2ws, remap,
                            use_color_volume, img_feat=None, base_feat_dim=8, num_ray_points=0):
    affine_mat = []
    affine_mat_inv = []
    imgs = []

    # Step 1: Initialize Image Data
    for vid in view_ids:
        img_filename = os.path.join(data_dir, f'Rectified/{scan}_train/rect_{vid + 1:03d}_{light_idx}_r5000.png')
        img = Image.open(img_filename)
        img_wh = np.round(np.array(img.size) * downSample).astype('int')
        img = img.resize(img_wh, Image.BILINEAR)
        img = normalize_tensor(img)  # [G] Standard normalization applied to image tensors
        imgs.append(img)

        index_mat = remap[vid]
        proj_mat_ls, near_far = proj_mats[index_mat]
        affine_mat.append(proj_mat_ls)
        affine_mat_inv.append(np.linalg.inv(proj_mat_ls))

    # Step 2: Determine Feature Dimensions
    feat_dim = base_feat_dim
    if img_feat is not None:
        feat_dim += img_feat.shape[-1]  # [G] Add spatial/channel dimension of img_feat

    # Step 3: Initialize Input Features
    if not use_color_volume:
        input_feat = allocate_tensor((num_ray_points, feat_dim))  # [G] Allocates feature tensor
        index_volume_features(input_feat)                          # [G] Populates from volume storage
        # Step 4: Assign Features to Ray Points
        ray_feats = input_feat[:, :8]
        color_volume_channels = input_feat[:, 8:]
        # Step 5: Build color volume
        color_volume = build_color_volume_from_images(color_volume_channels, imgs, affine_mat, affine_mat_inv)
    else:
        # If using color volume, directly index volume features
        ray_feats = index_volume_features_directly()  # [G] Retrieves features directly from volume
        color_volume = None

    return ray_feats, color_volume
```

**Parameters:**
- `downSample`: Typical value depends on resolution requirements; not specified in sources.
- `light_idx`: Varies by scene; not specified in sources.

**Edge Cases and Pitfalls:**
- View ID indexing: Image filenames use 1-based indexing for the view ID (`vid + 1`), while depth maps use 0-based indexing (`vid`) [S1, unofficial].
- Missing depth maps: Depth files may not exist for all views; the method should handle this gracefully [S1, unofficial].
- Precision: Image resizing uses bilinear interpolation, which may introduce smoothing artifacts [S1, unofficial].

## Subtopics

### [Voxel Feature Composition](7021-voxel-feature-composition.md)
Generates per-voxel feature vectors by sampling points within each Gaussian's volume and weighting them by Gaussian attenuation. It aggregates voxel-level geometric and appearance data into a single feature vector per voxel for storage or further processing. Unlike its sibling [Mean Feature Aggregation by Voxel](11242-mean-feature-aggregation-by-voxel.md), which relies on direct spatial averaging of raw points, this method computes features by sampling Gaussian volumes and weighting by attenuation.

### [Attention Aggregation for 3D Points](12093-attention-aggregation-for-3d-points.md)
Aggregates unordered, variable-sized sets of retrieved pixel features from multiple input views into a single unique feature vector for a query 3D point. It solves the problem of identifying which retrieved features are true descriptions of the query point in the presence of visual occlusions and arbitrary input counts. It sets itself apart from simple statistical aggregation (like mean pooling) by learning to weight views based on visual consistency and relative distance, rather than treating all views equally.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Explicitly builds a color volume from images, enabling direct feature indexing. Supports multi-view input [S1, unofficial].
- **Limitations:** Requires pre-computed camera parameters and aligned images [S1, unofficial].
- **When to use:** When ray-based rendering requires per-point features derived from 2D image data [G].
- **When not to use:** When out-of-core processing is required or when geometric features are sufficient without color data [G].

## Background
Point Feature Generation was developed in the Xharlie/pointnerf repository [S1, unofficial]. It was created to enhance the feature representation of ray points by leveraging multi-view image data [S1, unofficial].

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
