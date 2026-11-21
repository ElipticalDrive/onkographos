# Feature Plane Compression (CodecGS)
Also called: Compression of 3D Gaussian Splatting with Optimized Feature Planes and Standard Video Codecs, Tri-plane Architecture

## Overview
Feature Plane Compression (CodecGS) is a method for compressing 3D Gaussian Splatting (3DGS) representations by replacing direct storage of unstructured Gaussian primitives with a structured, learnable feature plane representation [S1]. It solves the problem of high storage costs and inefficient entropy coding of irregular 3D point clouds by mapping 3D positions to Gaussian attributes via a tri-plane architecture, which is then compressed using standard video codecs [S1]. Unlike sibling methods that compress geometry or use anchor-based residual coding, CodecGS leverages the spatial redundancy of natural scenes by organizing parameters in a 2D grid-like structure, allowing the use of mature image/video compression standards [S1].

## How it works
**Inputs:** A trained 3D Gaussian Splatting scene (positions, covariances, colors, opacities) or a feed-forward prediction of Gaussian attributes.
**Outputs:** A compressed bitstream representing the feature planes, decodable into Gaussian attributes for rendering.

**Data Structures:**
1.  **Tri-Plane Feature Map:** A set of three 2D feature planes ($F_x, F_y, F_z$) corresponding to the $xy$, $yz$, and $xz$ projections. Each plane stores a high-dimensional feature vector for each grid cell.
2.  **Attribute Prediction:** For a 3D point $\mathbf{x} = (x, y, z)$, the feature vector $\mathbf{f}$ is obtained by interpolating values from the three planes at the projected coordinates of $\mathbf{x}$.
3.  **Encoded Format:** Feature planes are normalized and converted to 16-bit integers, conforming to the YUV 16-bit format standard for video codecs [S1].

**Steps:**
1.  **Tri-Plane Initialization:** Initialize three 2D grids (tri-planes) with learnable parameters. The resolution of these planes determines the granularity of the attribute prediction [S1].
2.  **Two-Phase Training:**
    *   **Phase 1 (Standard 3DGS):** Perform standard 3D Gaussian Splatting training to optimize initial Gaussian positions, covariances, and colors [S1].
    *   **Phase 2 (Feature Plane Optimization):** Replace direct Gaussian attribute storage with the tri-plane prediction. The tri-planes are optimized to predict Gaussian attributes (position, scale, rotation, color, opacity) for any query point $\mathbf{x}$ [S1].
3.  **Progressive Training Strategy:** To address instability caused by sparse 3DGS signals, employ a progressive training strategy. Initially capture coarse geometric information with low-resolution tri-planes, then gradually increase resolution to add finer details [S1].
4.  **DCT Entropy Modeling:** Apply block-wise Discrete Cosine Transform (DCT) to the feature planes during optimization. This transforms the spatial data to leverage spatial correlations, similar to conventional image signals, improving compressibility [S1].
5.  **Smoothing Regularization:** Add a smoothing regularization term during training to promote locally smooth configurations of Gaussians on the grid, further enhancing compressibility [S1].
6.  **Quantization and Encoding:**
    *   Normalize the trained feature planes.
    *   Convert the normalized floating-point values to 16-bit integers [S1].
    *   Encode the resulting 16-bit integer planes using a standard video codec (e.g., H.264/AVC, H.265/HEVC) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def codecgs_train_and_compress(gaussian_scene, tri_plane_resolution):
    """
    Trains a tri-plane feature representation for 3DGS and compresses it.
    """
    # Initialize tri-planes: F_x, F_y, F_z of shape (H, W, C)
    # C is the channel dimension for feature vectors
    F_x = initialize_triplane(tri_plane_resolution)
    F_y = initialize_triplane(tri_plane_resolution)
    F_z = initialize_triplane(tri_plane_resolution)
    
    # Phase 1: Standard 3DGS Training
    # Optimize initial Gaussian primitives (pos, scale, rot, color, opacity)
    # using standard 3DGS loss (rendering error + regularization)
    gaussians = standard_3dgs_training(gaussian_scene)
    
    # Phase 2: Feature Plane Training
    # Replace direct storage with tri-plane prediction
    # Loss includes rendering error and smoothing regularization
    for epoch in range(num_epochs):
        # Sample 3D points x from the scene distribution
        points = sample_3d_points(gaussians)
        
        # Predict attributes from tri-planes
        # Interpolate features from F_x, F_y, F_z at projected coordinates
        features = predict_features_from_triplanes(points, F_x, F_y, F_z)
        
        # Decode attributes from features (pos, scale, rot, color, opacity)
        predicted_gaussians = decode_attributes(features)
        
        # Calculate rendering loss
        loss = calculate_rendering_loss(predicted_gaussians)
        
        # Add smoothing regularization to promote spatial smoothness
        smoothing_loss = calculate_smoothing_loss(F_x, F_y, F_z)
        
        # Total loss
        total_loss = loss + lambda_smooth * smoothing_loss
        
        # Update tri-planes using gradient descent
        update_parameters(F_x, F_y, F_z, total_loss)
        
        # Progressive resolution increase if applicable
        if epoch in progressive_schedule:
            increase_triplane_resolution(F_x, F_y, F_z)

    # Post-training: Prepare for video codec
    # Normalize feature planes to a specific range
    F_x_norm = normalize(F_x)
    F_y_norm = normalize(F_y)
    F_z_norm = normalize(F_z)
    
    # Convert to 16-bit integers (YUV 16-bit format)
    F_x_int = float_to_16bit_int(F_x_norm)
    F_y_int = float_to_16bit_int(F_y_norm)
    F_z_int = float_to_16bit_int(F_z_norm)
    
    # Encode using standard video codec
    # The codec expects YUV format; map F_x, F_y, F_z to Y, U, V components
    # or encode each plane separately depending on codec implementation
    bitstream = video_codec_encode(F_x_int, F_y_int, F_z_int)
    
    return bitstream

def predict_features_from_triplanes(points, F_x, F_y, F_z):
    """
    Interpolate features from tri-planes for given 3D points.
    """
    # Project 3D point x=(x,y,z) to 2D coordinates on each plane
    # xy-plane: (x, y), yz-plane: (y, z), xz-plane: (x, z)
    coords_xy = (points.x, points.y)
    coords_yz = (points.y, points.z)
    coords_xz = (points.x, points.z)
    
    # Sample features from each plane using bilinear interpolation
    f_xy = bilinear_sample(F_x, coords_xy)
    f_yz = bilinear_sample(F_y, coords_yz)
    f_xz = bilinear_sample(F_z, coords_xz)
    
    # Combine features (e.g., concatenation or weighted sum)
    # Note: Exact combination method not specified in sources, 
    # standard tri-plane usage often concatenates or averages.
    # [G] Assuming concatenation of feature vectors from each plane
    features = concatenate(f_xy, f_yz, f_xz)
    
    return features
```

**Parameters:**
*   **Tri-plane Resolution:** Not given in the sources. Determines the spatial granularity.
*   **Feature Channel Dimension ($C$):** Not given in the sources.
*   **Smoothing Regularization Weight ($\lambda_{smooth}$):** Not given in the sources.
*   **Quantization:** 16-bit integers [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Signals:** Feature plane training can be unstable due to sparse 3DGS signals. Mitigated by progressive training [S1].
*   **Boundary Conditions:** Interpolation at the boundaries of the tri-planes requires careful handling (e.g., clamping or padding) [G].
*   **Precision:** Conversion to 16-bit integers may introduce quantization artifacts. The choice of normalization range is critical [S1].

## Operating characteristics
*   **Compression Ratio:** Not given in the sources.
*   **Rendering Quality:** Not given in the sources.
*   **Complexity:** Encoding complexity depends on the standard video codec used. Decoding involves video decoding followed by tri-plane interpolation.
*   **Strengths:** Leverages mature video compression standards; exploits spatial redundancy in natural scenes; avoids complex entropy coding of irregular 3D structures [S1].
*   **Limitations:** Requires training of feature planes; interpolation overhead during rendering; potential loss of high-frequency details due to quantization and video codec compression [S1].
*   **When to Use:** When high compression ratios are needed and standard video codecs are available; suitable for feed-forward or pre-computed scenes [S1].
*   **When Not to Use:** When real-time training of feature planes is prohibitive; when exact preservation of unstructured Gaussian data is required without interpolation artifacts [S1].

## Background
CodecGS was introduced as a method to compress 3D Gaussian Splatting by employing optimized feature planes and standard video codecs [S1]. It builds on the observation that 3DGS parameters can be organized in a 2D grid to capitalize on perceptual redundancies found in natural scenes [S1]. The method interprets the training process of positioning and optimizing Gaussians as a sampling procedure, drawing inspiration from 3DGS-MCMC approaches [S1]. It is implemented in the `gsplat` library, an open-source library for CUDA-accelerated differentiable rasterization of 3D Gaussians [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
