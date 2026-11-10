# Dynamic MLP Maps

Also called: MLP maps, Orthogonal MLP maps

## Overview
Dynamic MLP Maps is a neural representation for volumetric videos that models the radiance field of each frame as a set of shallow Multi-Layer Perceptrons (MLPs) [S1]. The parameters for these MLPs are stored in 2D grids, called MLP maps, and are dynamically predicted by a shared 2D CNN decoder using a per-frame latent code [S1]. This method solves the trade-off between rendering speed and storage cost in dynamic scene representation: it achieves real-time rendering speeds (41.7 fps for 512×512 images) by using small, local MLPs instead of a single global network, while maintaining low storage by predicting parameters on-the-fly rather than storing them explicitly for every frame [S1]. It differs from [K-planes](3317-k-planes.md) by predicting MLP parameter maps instead of static feature maps, and from [Neural Volumes](7824-neural-voxel-fields.md) by using 2D CNNs instead of 3D CNNs [S1].

## How it works
**Inputs:** A multi-view video sequence with synchronized, calibrated cameras.
**Outputs:** A volumetric video representation supporting real-time novel view synthesis.

### Data Structures
1.  **Latent Code ($z$):** A vector $z \in \mathbb{R}^D$ encoding the scene state for a specific video frame.
2.  **MLP Maps:** 2D grids where each pixel stores a parameter vector for a small MLP network.
    *   **Density MLP Map:** Resolution $256 \times 256$. Each pixel stores 32 parameters ($32 \times 1$).
    *   **Color MLP Map:** Resolution $16 \times 16$. Each pixel stores 2624 parameters ($32 \times 32 + (32 + 15) \times 32 + 32 \times 3$).
3.  **Hash Tables:** Three multi-level hash tables ($h_{xy}, h_{xz}, h_{yz}$) for spatial feature embedding.
    *   Resolution: $L \times T \times F$.
    *   Parameters: $L=19$ levels, $T=16$ table size, $F=2$ feature dimension [S1].
4.  **Occupancy Volume:** A low-resolution 3D grid storing binary values indicating occupied voxels (density > threshold) for Empty Space Skipping (ESS) [S1].

### Algorithm Steps

1.  **Encoding (Training Phase):**
    *   Select a subset of camera views for the current video frame.
    *   Pass these views through a 2D CNN encoder to obtain the latent code $z$ [S1].
    *   *Note:* During inference, this step is skipped if Encoder Caching is used.

2.  **Decoding (MLP Map Generation):**
    *   Map $z$ to a 4096-dimensional feature vector using a fully-connected network.
    *   Reshape the vector into a $4 \times 4$ feature map with 256 channels.
    *   Upsample the $4 \times 4$ map to a $D \times D$ feature map using deconvolutional layers [S1].
    *   Apply two separate convolutional networks to the feature map:
        *   One predicts the **Density MLP Map** ($256 \times 256$).
        *   One predicts the **Color MLP Map** ($16 \times 16$) [S1].

3.  **Spatial Embedding (Query Time):**
    *   For a 3D query point $p = (x, y, z)$ and viewing direction $d$:
    *   Project $p$ onto three axis-aligned orthogonal planes (XY, XZ, YZ).
    *   Retrieve features from the three hash tables $h_{xy}, h_{xz}, h_{yz}$ for the projected points.
    *   Aggregate features via summation to get hash feature $\gamma_{ph}(p)$ [S1].
    *   Retrieve tri-plane features $\gamma_{pt}(p)$ from the MLP maps (see Step 4) [S1].
    *   Compute final feature vector: $\gamma_p(p) = \gamma_{ph}(p) + \gamma_{pt}(p)$ [S1].
    *   Encode viewing direction $d$ using positional encoding: $\gamma_d(d)$ [S1].

4.  **MLP Map Querying:**
    *   Project $p$ onto the canonical planes defined by the MLP maps (e.g., YZ plane).
    *   Retrieve the parameter vectors from the Density and Color MLP maps at the projected location via spatial binning [S1].
    *   Construct the small NeRF network:
        *   **Density Head:** 1-layer MLP.
        *   **Color Head:** 3-layer MLP [S1].

5.  **Prediction:**
    *   Compute density $\sigma$ and color $\mathbf{c}$:
        $$ (\sigma, \mathbf{c}) = M(\gamma_p(p), \gamma_d(d)) $$
        where $M$ is the constructed small MLP network [S1].

6.  **Aggregation (Orthogonal Variant):**
    *   If using three orthogonal MLP maps (one for each plane), predict $(\sigma_i, \mathbf{c}_i)$ for $i=1,2,3$.
    *   Aggregate results via summation [S1].

7.  **Rendering:**
    *   Perform volume rendering by sampling points along rays.
    *   Use Empty Space Skipping (ESS): Skip sampling in voxels marked as empty in the Occupancy Volume [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def positional_encoding(v, num_frequencies=15):
    """Standard positional encoding for viewing direction."""
    scales = [2**i for i in range(num_frequencies)]
    scaled_v = v * scales
    return math.sin(scaled_v)

def get_hash_features(point, hash_tables):
    """
    Retrieve and aggregate features from multi-level hash tables.
    hash_tables: list of 3 hash tables [h_xy, h_xz, h_yz]
    point: tuple (x, y, z)
    """
    features = []
    # Project point to planes and query hash tables
    # Note: Exact hash table implementation details (level selection) 
    # are abstracted here as per source [S1] reference to [40].
    for table in hash_tables:
        # Project point to the plane corresponding to the table
        # e.g., for h_xy, project to (x, y)
        projected_point = project_to_plane(point, table.plane)
        # Query hash table for feature vector
        feat = table.query(projected_point)
        features.append(feat)
    
    # Aggregate via summation
    aggregated = sum(features)
    return aggregated

def get_tri_plane_features(point, mlp_maps):
    """
    Retrieve features from MLP maps.
    mlp_maps: dict containing 'density_map' and 'color_map'
    """
    # Project point to canonical planes (e.g., YZ, XZ, XY)
    # Retrieve parameters from the 2D grids at projected coordinates
    # This returns the feature vector gamma_pt(p)
    # Implementation depends on specific spatial binning logic
    return retrieve_mlp_features(point, mlp_maps)

def build_small_mlp(density_params, color_params):
    """
    Construct the small NeRF network from retrieved parameters.
    density_params: vector of 32 values
    color_params: vector of 2624 values
    """
    # Density Head: 1 layer
    # Color Head: 3 layers
    # Parameters are reshaped and loaded into network weights
    density_head = LinearLayer(density_params)
    color_head = MultiLayerNetwork(color_params)
    return SmallNeRF(density_head, color_head)

def query_scene(point, direction, hash_tables, mlp_maps):
    """
    Query density and color for a point.
    """
    # 1. Spatial Embedding
    gamma_ph = get_hash_features(point, hash_tables)
    gamma_pt = get_tri_plane_features(point, mlp_maps)
    gamma_p = gamma_ph + gamma_pt
    
    # 2. Direction Encoding
    gamma_d = positional_encoding(direction)
    
    # 3. Retrieve MLP Parameters from Maps
    # Project point to planes defined by MLP maps
    density_params = retrieve_from_map(point, mlp_maps['density_map'])
    color_params = retrieve_from_map(point, mlp_maps['color_map'])
    
    # 4. Build and Evaluate MLP
    mlp = build_small_mlp(density_params, color_params)
    sigma, color = mlp(gamma_p, gamma_d)
    
    return sigma, color

def render_frame(latent_code, hash_tables, occupancy_volume, camera_params):
    """
    Render a single frame.
    """
    # 1. Decode MLP Maps from Latent Code
    # (Only needed if not using cached maps, but typically maps are generated per frame)
    feature_map = decode_latent_to_feature_map(latent_code)
    density_map = predict_density_map(feature_map)
    color_map = predict_color_map(feature_map)
    mlp_maps = {'density_map': density_map, 'color_map': color_map}
    
    # 2. Ray Marching with ESS
    for each pixel in image:
        ray = generate_ray(camera_params, pixel)
        samples = sample_ray(ray, occupancy_volume)
        color = volume_render(samples, hash_tables, mlp_maps)
        set_pixel(pixel, color)
```

### Parameters and Typical Values
*   **Rendering Resolution:** 512 × 512 [S1].
*   **Density MLP Map Resolution:** 256 × 256 [S1].
*   **Color MLP Map Resolution:** 16 × 16 [S1].
*   **Hash Table Levels ($L$):** 19 [S1].
*   **Hash Table Size ($T$):** 16 [S1].
*   **Hash Table Feature Dimension ($F$):** 2 [S1].
*   **Density MLP Parameters:** 32 per map [S1].
*   **Color MLP Parameters:** 2624 per map [S1].

### Edge Cases and Pitfalls
*   **Resolution Trade-off:** Decreasing MLP map resolution undermines rendering performance and increases storage cost (due to added downsampling layers). Increasing resolution does not necessarily improve quality because it increases parameters to optimize and forces smaller batch sizes due to memory limits [S1].
*   **Orthogonality:** Using a single dynamic MLP map on one plane (e.g., XY) or three maps all on the same plane significantly degrades rendering quality. Orthogonal maps (XY, XZ, YZ) are required for high-frequency content [S1].
*   **Training Difficulty:** High-resolution density maps are easier to train than color maps because geometry has lower frequency than appearance, and the parameter counts are balanced ($256 \times 256$ density map $\approx 28 \times 28$ color map) [S1].
*   **Video Length:** The method is limited to short videos (100–300 frames). Storage cost increases with video length due to hash tables and latent vectors [S1].
*   **Camera Views:** Requires dense camera views for training [S1].

## Subtopics

### [Dynamic MLP Maps with Hash Encoding](7378-dynamic-mlp-maps-with-hash-encoding.md)
A variant that replaces the direct feature lookup in the base method with multi-level hash encoding for spatial features. It predicts MLP parameters via 2D CNNs from axis-aligned orthogonal maps, combined with hash tables to improve spatial resolution and detail capture while maintaining real-time rendering efficiency. It builds on [Dynamic MLP Maps](7360-dynamic-mlp-maps.md) by enhancing the spatial feature representation.

### Encoder Caching
An acceleration strategy where the 2D CNN encoder is discarded after training [S1]. Latent vectors for all frames are pre-computed and stored (300 KB for 300 frames), eliminating the encoder inference time during rendering [S1].

## Operating characteristics
*   **Speed:** 41.7 fps on an RTX 3090 GPU for 512×512 images [S1]. Rendering time for a 512×512 image is 24 ms with the complete model, compared to 144 ms without Empty Space Skipping (ESS) [S1]. This is over 100 times faster than DyNeRF [S1].
*   **Inference Breakdown:** MLP maps decoder ~4 ms, querying multi-level hash tables ~7 ms, color and density MLP maps total 12 ms [S1].
*   **Memory/Storage:** For a 300-frame video: 300 KB latent vectors, 103 MB MLP maps decoders, 131 MB multi-level hash tables, 8 MB occupancy volumes [S1]. ESS adds 5 MB to storage but improves speed 6 times [S1].
*   **Complexity:** Rendering speed is largely unaffected by MLP map resolution because the number of MLP evaluations is independent of the map resolution [S1].
*   **Strengths:** Real-time rendering, low storage cost compared to explicit storage methods, high quality on NHR and ZJU-MoCap datasets [S1].
*   **Limitations:** Limited to short videos (100-300 frames), requires dense camera views, storage scales with video length [S1].

## Background
Dynamic MLP Maps was introduced to address the slow rendering speed and high storage cost of extending neural radiance fields to dynamic scenes [S1]. It builds on the idea of using shallow MLPs for fast evaluation, as seen in static scene methods, but adapts it for video by dynamically predicting parameters via a shared 2D CNN [S1]. The method was evaluated on the NHR and ZJU-MoCap datasets, achieving state-of-the-art PSNR, SSIM, and LPIPS scores among compared methods at the time of publication [S1].

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
