# Texture-based Transfer Function

Also called: Texture-based TF, tbTF

## Overview
The Texture-based Transfer Function (tbTF) is a feature-based classification method that maps volume data to color and opacity using the scalar data value and 20 computed texture metrics [S1]. It distinguishes features based on local statistical properties of intensity distributions rather than just magnitude or gradient [S1]. Unlike simpler scalar-based TFs, tbTF requires significant preprocessing to compute neighborhood statistics, making it computationally expensive but capable of distinguishing structures with similar intensities but different textures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$.
*   **Output:** A classification result (color and opacity) for each voxel, typically derived from a 2D Lookup Table (LUT) [S1].

**Data Structures:**
The method requires extensive auxiliary data structures computed during preprocessing [S1]:
1.  **Metric Volumes:** 20 separate 3D volumes, one for each texture metric [S1].
2.  **GLCM Volume:** 1 volume storing Gray Level Co-occurrence Matrix statistics [S1].
3.  **Run-Length Volume:** 1 volume storing Run-Length Matrix statistics [S1].
4.  **Label Volume:** 1 volume storing cluster labels (used during rendering for automatic mode) [S1].

**Feature Extraction:**
The 20 texture metrics are categorized into three groups [S1]:
1.  **6 First-order statistics:** Measures of intensity distribution (e.g., mean, variance) without considering spatial relationships.
2.  **7 Second-order statistics:** Derived from Gray Level Co-occurrence Matrices (GLCM), capturing spatial relationships between voxel intensities.
3.  **7 Run-length matrices:** Measures based on the length of consecutive voxels with the same intensity value.

**Specification Methods:**
Due to the high dimensionality (21 inputs: 1 data value + 20 metrics), manual specification is impractical. The method offers three specification modes [S1]:
1.  **Fully Automatic:** Uses clustering algorithms to recognize structures. The user provides only the number of expected structures. The system automatically computes metrics and clusters them to define the TF [S1].
2.  **Semi-Automatic (Manual Weighting):** The user manually weights specific statistics to influence the classification [S1].
3.  **Semi-Automatic (Structure Distinction):** The user specifies two structures to be distinguished, or marks a single structure to be separated from the rest [S1].

**Rendering:**
During rendering, the precomputed metric volumes are sampled [S1].
*   In **Automatic Mode:** The renderer performs lookups in the data volume and the label volume (2 lookups) [S1].
*   In **Semi-Automatic Mode:** The renderer performs lookups in the data volume and all 20 metric volumes (21 lookups) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_based_transfer_function(volume, mode, num_structures=None, weights=None, reference_structures=None):
    """
    Computes and applies the Texture-based Transfer Function.
    
    Args:
        volume: 3D numpy array of input data.
        mode: 'automatic', 'manual_weighting', or 'structure_distinction'.
        num_structures: int, required if mode is 'automatic'.
        weights: list of floats, required if mode is 'manual_weighting'.
        reference_structures: list of voxel coordinates or masks, required if mode is 'structure_distinction'.
        
    Returns:
        color_volume: 3D array of RGB colors.
        opacity_volume: 3D array of opacities.
    """
    
    # Step 1: Preprocessing - Compute Texture Metrics
    # Compute 20 metric volumes. This is expensive.
    # Metrics include 6 first-order, 7 GLCM-based, 7 Run-Length based.
    metric_volumes = compute_texture_metrics(volume)
    
    # Compute auxiliary volumes for GLCM and Run-Length if not included in metric_volumes
    glcm_volume = compute_glcm_statistics(volume)
    runlength_volume = compute_runlength_statistics(volume)
    
    label_volume = None
    
    # Step 2: Specification
    if mode == 'automatic':
        if num_structures is None:
            raise ValueError("num_structures required for automatic mode")
        # Cluster the 20 metric volumes to identify structures
        # This is barely controllable and risks misinterpretation [S1]
        label_volume = cluster_metrics(metric_volumes, num_clusters=num_structures)
        # Map labels to colors/opacities automatically
        tf_map = generate_auto_tf_from_labels(label_volume)
    elif mode == 'manual_weighting':
        if weights is None:
            raise ValueError("weights required for manual weighting mode")
        # Combine metrics using user-defined weights
        # Requires high understanding of metrics [S1]
        combined_score = weighted_sum_metrics(metric_volumes, weights)
        tf_map = generate_tf_from_score(combined_score)
    elif mode == 'structure_distinction':
        if reference_structures is None:
            raise ValueError("reference_structures required for structure distinction mode")
        # Compute distance or similarity to reference structures
        # Separates marked structure from rest [S1]
        tf_map = generate_tf_for_distinction(volume, metric_volumes, reference_structures)
    else:
        raise ValueError("Unknown mode")

    # Step 3: Rendering
    # Apply TF map to volume data
    # In automatic mode: lookup data + label (2 lookups) [S1]
    # In semi-automatic mode: lookup data + 20 metrics (21 lookups) [S1]
    
    color_volume = apply_tf_map(volume, metric_volumes, label_volume, tf_map)
    opacity_volume = compute_opacity(color_volume, tf_map)
    
    return color_volume, opacity_volume

def compute_texture_metrics(volume):
    """
    Computes 20 texture metrics for each voxel.
    Returns a list of 20 3D volumes.
    """
    # First-order statistics (6 metrics)
    # e.g., mean, variance, skewness, kurtosis, energy, entropy
    first_order = compute_first_order_stats(volume)
    
    # Second-order statistics from GLCM (7 metrics)
    # e.g., contrast, correlation, energy, homogeneity, ASM, entropy, variance
    glcm_stats = compute_glcm_stats(volume)
    
    # Run-length statistics (7 metrics)
    # e.g., Short Run Emphasis, Long Run Emphasis, Run Length, Run Percentage, etc.
    runlength_stats = compute_runlength_stats(volume)
    
    return first_order + glcm_stats + runlength_stats

def cluster_metrics(metric_volumes, num_clusters):
    """
    Clusters the flattened metric volumes to assign labels.
    """
    # Flatten volumes to 2D array (N_voxels x 20_metrics)
    data_matrix = np.vstack([m.flatten() for m in metric_volumes]).T
    # Apply clustering algorithm (e.g., K-Means)
    labels = kmeans(data_matrix, n_clusters=num_clusters)
    # Reshape labels back to 3D volume
    label_volume = labels.reshape(metric_volumes[0].shape)
    return label_volume
```

**Parameters:**
*   **Number of Structures:** Integer, used in automatic mode [S1].
*   **Weights:** Array of 20 floats, used in manual weighting mode [S1].
*   **Reference Structures:** Voxel coordinates or masks, used in structure distinction mode [S1].

**Edge Cases and Pitfalls:**
*   **Noise:** Texture analysis is highly sensitive to noise. The dataset must be very clean, or preprocessing denoising is required [S1].
*   **Automatic Mode Risks:** The fully automatic method is barely controllable, has poor ability to emphasize determined features, and carries a high risk of wrong results/misinterpretation [S1].
*   **Resolution Trade-off:** Texture analysis does not necessarily require full volume resolution. Lower resolution can be used to trade time for precision [S1].

## Operating characteristics
*   **Preprocessing Time:** Very expensive. Computing 20 metrics and clustering is time-consuming [S1].
*   **Memory Consumption:** High. Requires up to 20 metric volumes, 1 GLCM volume, and 1 run-length volume during preprocessing [S1]. During rendering, only 1 additional volume (label volume) is needed if using automatic mode, or 20 metric volumes if using semi-automatic mode [S1].
*   **Rendering Performance:**
    *   **Automatic Mode:** Real-time capable? No. Requires 2 3D texture lookups per sample (data + label) [S1].
    *   **Semi-Automatic Mode:** Real-time capable? No. Requires 21 3D texture lookups per sample (data + 20 metrics) [S1].
*   **Strengths:** Can distinguish features with similar intensities but different textures [S1].
*   **Limitations:** High computational cost, sensitivity to noise, poor controllability in automatic mode [S1].
*   **When to Use:** When texture is a distinguishing feature and real-time performance is not critical.
*   **When Not to Use:** When data is noisy, when real-time rendering is required, or when manual control over feature emphasis is needed (automatic mode is unsuitable).

## Background
The Texture-based Transfer Function was introduced by [CR08] and is compared against other TF types (1D, gradient-based, curvature-based, size-based, distance-based) in a survey by [S1]. It is particularly relevant in medical visualization where tissue texture can indicate pathology [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
