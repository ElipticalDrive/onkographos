# Texture-based Transfer Function

Also called: Texture-based TF, tbTF

## Overview
The Texture-based Transfer Function (tbTF) is a data feature-based classification method that maps voxel properties to visual attributes using raw data values and 20 local texture metrics [S1]. It solves the problem of distinguishing materials or structures that have similar intensity values but different microstructures, which scalar or gradient-based transfer functions cannot separate [S1]. Unlike its siblings, tbTF relies on a high-dimensional feature space (21 dimensions) and typically employs clustering or semi-automatic weighting rather than direct manual mapping, making it distinct from the simpler [Distance-based Transfer Function](5451-distance-based-transfer-function.md) or [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $V$ and a specification mode (fully automatic or semi-automatic).
*   **Output:** A classification result (labels or confidence scores) mapped to opacity $\alpha$ and color $\mathbf{c}$.

**Data Structures:**
The method requires significant preprocessing storage. The layout includes:
1.  **Input Volume:** $V$ (scalar).
2.  **Metric Volumes:** 20 separate volumes, one for each texture metric.
3.  **Matrix Volumes:** 2 additional volumes storing Gray Level Co-occurrence Matrices (GLCM) and Run-Length Matrices (RLM) data [S1].
4.  **Output Volume:** A label volume (or confidence volume) used during rendering.

**Feature Extraction:**
The feature vector $\mathbf{f}_i$ for each voxel $i$ consists of 21 components:
1.  The raw data value $V_i$.
2.  Six 1st-order statistics.
3.  Seven 2nd-order statistics.
4.  Seven metrics derived from Run-Length Matrices (RLM) [S1].

*Note: The specific mathematical formulas for the 20 texture metrics are not provided in the source [CR08] cited by [S1]. The source notes that these metrics characterize local texture patterns [S1].*

**Classification Steps:**
The specification of the transfer function follows one of two paths:

1.  **Fully Automatic Mode:**
    1.  Compute the 20 texture metrics for the volume.
    2.  Perform clustering on the 21-dimensional feature vectors.
    3.  Input: The user specifies only the number of structures (clusters) to identify.
    4.  Output: Cluster labels are assigned to voxels.
    5.  *Constraint:* This mode is barely controllable and carries a high risk of misinterpretation [S1].

2.  **Semi-Automatic Mode:**
    The user defines the classification logic using one of three sub-methods:
    *   **Manual Weighting:** The user manually weights individual statistics. This requires high expertise in the metrics [S1].
    *   **Two-Structure Distinction:** The user specifies two structures to be distinguished; the classifier separates them based on metric differences.
    *   **Single Structure Separation:** The user marks a single structure to be separated from the background.

**Rendering:**
During rendering, the precomputed labels or metrics are used to look up opacity and color.
*   If using labels (output of automatic/semi-automatic classification): 2 texture lookups per sample (data + label) [S1].
*   If using raw metrics directly (rare, as mapping is difficult): 21 texture lookups per sample (data + 20 metrics) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_based_transfer_function(volume, mode, num_clusters=None, user_weights=None, seed_structure=None):
    """
    Computes texture-based transfer function.
    
    Args:
        volume: 3D array of scalar values V.
        mode: 'automatic' or 'semi-automatic'.
        num_clusters: int, required if mode is 'automatic'.
        user_weights: dict/list, required if mode is 'semi-automatic' (manual weighting).
        seed_structure: mask/label, required if mode is 'semi-automatic' (single structure separation).
        
    Returns:
        label_volume: 3D array of classification labels.
    """
    # Step 1: Preprocessing - Compute Texture Metrics
    # Note: Specific formulas for the 20 metrics are not detailed in [S1].
    # We assume a function compute_texture_metrics exists that returns 20 volumes.
    metric_volumes = compute_texture_metrics(volume)
    
    # Step 2: Construct Feature Vectors
    # Stack the raw volume and the 20 metric volumes into a feature space
    # Shape: (H, W, D, 21)
    feature_space = stack([volume] + metric_volumes)
    
    if mode == 'automatic':
        if num_clusters is None:
            raise ValueError("num_clusters must be specified for automatic mode")
        
        # Step 3: Clustering
        # Perform clustering (e.g., K-Means) on the flattened feature space
        # The source [S1] states this is "very expensive"
        labels = cluster(feature_space, k=num_clusters)
        
    elif mode == 'semi-automatic':
        if user_weights is not None:
            # Manual weighting of statistics
            # Weighted combination of metrics to form a discriminative score
            score = weighted_sum_metrics(feature_space, user_weights)
            labels = threshold_or_cluster(score)
        elif seed_structure is not None:
            # Separate a single marked structure
            # Likely involves distance or similarity to the seed in feature space
            labels = separate_structure(feature_space, seed_structure)
        else:
            # Two-structure distinction
            # Requires user to specify two target regions
            labels = distinguish_two_structures(feature_space)
            
    return labels

def compute_texture_metrics(volume):
    """
    Computes 20 texture metrics:
    - 6 1st order statistics
    - 7 2nd order statistics
    - 7 Run-Length Matrix metrics
    Returns a list of 20 3D volumes.
    """
    # Implementation details for specific texture metrics (e.g., GLCM, RLM) 
    # are not provided in the source [S1].
    # This function would calculate local histograms, co-occurrence matrices, 
    # and run-length encodings for each voxel's neighborhood.
    raise NotImplementedError("Specific metric formulas not in source")
```

## Operating characteristics
*   **Time Complexity:** Very high preprocessing time due to the computation of 20 texture metrics and potential clustering [S1]. Rendering time is comparable to other TFs if labels are used, but significantly higher if raw metrics are looked up (21 lookups vs 2) [S1].
*   **Memory Complexity:** Very high. Requires storing up to 20 metric volumes plus 2 matrix volumes during preprocessing [S1]. During rendering, only the label volume is needed (1 additional volume), which does not necessarily need to be full size [S1].
*   **Real-time Capability:** No. The preprocessing is too expensive for real-time interaction [S1].
*   **Strengths:** Can distinguish features with similar intensities but different textures (e.g., left vs. right features in Figure 6 of [S1], or cerebellum vs. brain in Figure 8d) [S1].
*   **Limitations:** Highly sensitive to noise; requires very clean datasets [S1]. The fully automatic mode is barely controllable and prone to misinterpretation [S1]. Semi-automatic manual weighting is difficult and time-consuming [S1].
*   **Trade-offs:** Texture analysis does not require full volume resolution; lower resolution can be used to trade precision for speed [S1].

## Background
The tbTF was introduced in the context of comparing transfer function types for volume rendering, specifically citing [CR08] [S1]. It is part of a family of data feature-based classification methods that extend beyond scalar or gradient mapping. The method is particularly relevant in medical visualization where tissue texture differences are diagnostically important despite similar Hounsfield units or signal intensities [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
