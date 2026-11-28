# K-Means Spatial-Intensity Partitioning


## Overview
K-Means Spatial-Intensity Partitioning partitions a 3D volume into a specified number of components by clustering foreground voxels based on a weighted combination of their spatial coordinates (x, y, z) and intensity values [S1, unofficial]. It sets itself apart from sibling methods like [Greedy Axis-Aligned Spatial Partitioning](11589-greedy-axis-aligned-spatial-partitioning.md) by incorporating intensity into the clustering metric, and from [Automatic Crop Box Generation via Otsu Thresholding](17117-automatic-crop-box-generation-via-otsu-thresholding.md) by producing multiple semantic partitions rather than a single binary mask [G]. This method groups visually or structurally similar regions for downstream processing such as compression or rendering optimization [G].

## How it works
**Inputs:**
- `volume`: A 3D numpy array of voxel data.
- `components`: Integer, number of clusters/partitions to form. Default: 8.
- `spatial_weight`: Float, weight for spatial coordinates (x, y, z). Intensity weight is $1 - \text{spatial\_weight}$. Default: 0.5.
- `seed`: Integer, random seed for reproducibility.
- `batch_size`: Integer, number of voxels processed per prediction batch. Default: 100,000.

**Outputs:**
- `labels`: A 3D array of the same shape as `volume`, containing integer cluster labels. Background voxels (zero intensity) are assigned label 0. Foreground voxels are assigned labels $1$ to $K$.

**Data Structures:**
- `foreground`: 1D array of flat indices where `volume` is non-zero.
- `sample`: 1D array of up to 30,000 indices drawn from `foreground`.
- `features`: 2D array of shape $(N, 4)$, where $N$ is the number of sampled voxels. Columns are $x, y, z, \text{intensity}$.
- `scaler`: A `StandardScaler` instance fitted on the sample features.
- `kmeans_model`: A fitted K-Means model.

**Steps:**
1. **Identify Foreground:** Compute `foreground = np.flatnonzero(volume)` [S1, unofficial]. If `foreground` is empty, return an array of zeros [S1, unofficial].
2. **Initialize Labels:** Create `labels` array of zeros with shape `volume.shape` [S1, unofficial].
3. **Sample Voxels:** Select up to 30,000 indices from `foreground` without replacement: `sample = rng.choice(foreground, min(30000, foreground.size), replace=False)` [S1, unofficial].
4. **Extract Features:** Unravel flat indices to $(z, y, x)$ coordinates. Stack $x, y, z$ and the corresponding intensity value from `volume` to form a feature vector $[x, y, z, \text{intensity}]$. Cast all values to `float32` [S1, unofficial].
5. **Normalize:** Fit a `StandardScaler` on the sample features. Transform the sample features using the fitted scaler [S1, unofficial].
6. **Define Weights:** Construct a weight vector $W = [w_s, w_s, w_s, w_i]$ where $w_s = \text{spatial\_weight}$ and $w_i = 1 - \text{spatial\_weight}$ [S1, unofficial].
7. **Scale Features:** Multiply the transformed sample features element-wise by $W$ [S1, unofficial].
8. **Fit K-Means:** Fit a K-Means model with $n\_clusters = \min(\text{components}, \text{unique\_count})$, where `unique_count` is the number of unique values in `volume`, $n\_init = 3$, $random\_state = \text{seed}$, and $max\_iter = 100$ [S1, unofficial].
9. **Predict Labels:** Iterate through all foreground indices in batches of `batch_size`: extract features, transform using the fitted `scaler`, scale by $W$, predict cluster labels, and assign them to the `labels` array at the corresponding flat indices, adding 1 to the result so background remains 0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans

def kmeans_spatial_intensity_partitioning(volume, components=8, spatial_weight=0.5, seed=42, batch_size=100000):
    # 1. Identify Foreground
    foreground = np.flatnonzero(volume)
    if foreground.size == 0:
        return np.zeros_like(volume, dtype=np.int32)
    
    # 2. Initialize Labels
    labels = np.zeros(volume.shape, dtype=np.int32)
    
    # 3. Sample Voxels
    rng = np.random.default_rng(seed)
    n_samples = min(30000, foreground.size)
    sample_indices = rng.choice(foreground, size=n_samples, replace=False)
    
    # 4. Extract Features
    # Unravel flat indices to (z, y, x)
    coords = np.unravel_index(sample_indices, volume.shape)
    x = coords[2].astype(np.float32)
    y = coords[1].astype(np.float32)
    z = coords[0].astype(np.float32)
    intensity = volume[sample_indices].astype(np.float32)
    
    # Stack features: [x, y, z, intensity]
    features = np.stack([x, y, z, intensity], axis=-1)
    
    # 5. Normalize
    scaler = StandardScaler()
    features_scaled = scaler.fit_transform(features)
    
    # 6. Define Weights
    w_s = spatial_weight
    w_i = 1.0 - spatial_weight
    weights = np.array([w_s, w_s, w_s, w_i], dtype=np.float32)
    
    # 7. Scale Features
    features_weighted = features_scaled * weights
    
    # 8. Fit K-Means
    unique_count = len(np.unique(volume))
    n_clusters = min(components, unique_count)
    
    kmeans = KMeans(n_clusters=n_clusters, n_init=3, random_state=seed, max_iter=100)
    kmeans.fit(features_weighted)
    
    # 9. Predict Labels for all foreground voxels
    n_foreground = foreground.size
    for start in range(0, n_foreground, batch_size):
        end = min(start + batch_size, n_foreground)
        batch_indices = foreground[start:end]
        
        # Extract features for batch
        batch_coords = np.unravel_index(batch_indices, volume.shape)
        bx = batch_coords[2].astype(np.float32)
        by = batch_coords[1].astype(np.float32)
        bz = batch_coords[0].astype(np.float32)
        b_intensity = volume[batch_indices].astype(np.float32)
        
        batch_features = np.stack([bx, by, bz, b_intensity], axis=-1)
        batch_scaled = scaler.transform(batch_features)
        batch_weighted = batch_scaled * weights
        
        # Predict
        batch_labels = kmeans.predict(batch_weighted)
        
        # Assign to output labels (add 1 for background)
        labels.flat[batch_indices] = batch_labels + 1
        
    return labels
```

## Subtopics

### [SLIC Superpixel Segmentation with Agglomerative Clustering](11782-slic-superpixel-segmentation-with-agglomerative-clustering.md)
A segmentation method for multivariate volumes that generates SLIC superpixels from multivariate voxels and merges them through progressive agglomerative clustering to accurately segment phases/composites. It sets itself apart from its parent method by first creating local spatial coherence via superpixels and then refining global phase consistency via agglomerative merging, solving the problem of disjointed regions of the same phase being separated by noise. Choose it when accurate phase segmentation in multivariate data is required. [SLIC Superpixel Segmentation with Agglomerative Clustering](11782-slic-superpixel-segmentation-with-agglomerative-clustering.md)

### [Sub-Vector Quantization (SVQ) for Gaussian Features](13486-sub-vector-quantization-svq-for-gaussian-features.md)
A compression technique applied to per-Gaussian features (scale, rotation, appearance) that uses fixed K-Means cluster indices and dynamically optimized codebooks to reduce storage footprint. It sets itself apart by decoupling cluster assignment from codebook optimization, allowing for feature-specific sub-vector lengths and dynamic range learning via 16-bit floating-point codebooks. Choose it when a trade-off between storage efficiency and attribute representation accuracy is needed. [Sub-Vector Quantization (SVQ) for Gaussian Features](13486-sub-vector-quantization-svq-for-gaussian-features.md)

### [Depth Map Clustering](14006-depth-map-clustering.md)
A method that reduces the number of depth layers in a depth map to accelerate hologram calculation with minimal visual quality decrease. It partitions the depth range into a limited set of intervals using K-Means clustering on the depth map histogram. Choose it when accelerating hologram rendering is the priority. [Depth Map Clustering](14006-depth-map-clustering.md)

### [Illumination-Aware Data Transformation](16293-illumination-aware-data-transformation.md)
A precomputation step that transforms multiple volumetric modalities into a single common data structure using K-Means cluster analysis with illumination-aware weighting. It sets itself apart by incorporating occlusion and boundary-based weights into the distance metric to prioritize visually significant voxels. Choose it when rendering multimodal volumes with advanced illumination algorithms to reduce GPU memory footprint. [Illumination-Aware Data Transformation](16293-illumination-aware-data-transformation.md)

### [Voxel Clustering (Global)](16560-voxel-clustering-global.md)
A preprocessing step that groups downsampled voxels into a small number of clusters based on their albedo values to share weight factors for scattering parameters. It addresses the challenge of managing a large number of unknowns in anisotropic media rendering by reducing spatial variability into discrete, globally consistent groups. Choose it when minimizing global rendering error in anisotropic media is required. [Voxel Clustering (Global)](16560-voxel-clustering-global.md)

### [Spherical Harmonic Clustering](17070-spherical-harmonic-clustering.md)
A method that reduces the memory footprint of Gaussian Splatting assets by clustering spherical harmonic (SH) coefficients using K-Means, trading precision for storage efficiency. Choose it when compressing Gaussian Splatting assets is the goal. [Spherical Harmonic Clustering](17070-spherical-harmonic-clustering.md)

## Operating characteristics
- **Time Complexity:** Dominated by the K-Means fitting on the sample (up to 30,000 points) and the prediction pass over all foreground voxels [S1, unofficial]. Prediction scales as $O(N_{fg} \cdot K \cdot D)$, where $N_{fg}$ is the foreground voxel count, $K$ is the number of clusters, and $D=4$ is the feature dimension [S1, unofficial].
- **Memory Complexity:** $O(N_{fg})$ for storing the output labels and $O(N_{sample})$ for intermediate feature arrays, where $N_{sample} \le 30,000$ [S1, unofficial].
- **Strengths:** Balances spatial coherence and intensity similarity. The fixed sampling cap makes it feasible for very large volumes without exhausting RAM [S1, unofficial].
- **Limitations:** Relies on a random sample, which may not represent the entire volume's distribution if the volume is highly heterogeneous [S1, unofficial]. The choice of `spatial_weight` significantly impacts cluster shape and semantic meaning [S1, unofficial].
- **When to use:** When partitioning a volume into semantically meaningful groups that respect both geometry and content intensity [S1, unofficial].
- **When not to use:** When strict spatial locality is required without intensity consideration, or when only a binary foreground/background split is needed [G].

## Background
This method is implemented in the `VolumeSegmenter` repository [S1, unofficial]. It serves as a flexible preprocessing step for volume partitioning pipelines, adapting to different volume characteristics via the `spatial_weight` parameter [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
