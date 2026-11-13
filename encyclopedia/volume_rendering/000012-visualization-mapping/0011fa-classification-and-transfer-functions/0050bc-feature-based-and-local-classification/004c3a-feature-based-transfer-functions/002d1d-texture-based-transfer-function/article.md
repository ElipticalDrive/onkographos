# Texture-based Transfer Function

Also called: Texture-based TF, tbTF

## Overview
The Texture-based Transfer Function (tbTF) maps scalar data values and 20 computed texture metrics to color and opacity, distinguishing features based on their local statistical texture rather than just intensity or gradient [S1]. It sets itself apart from siblings like the Distance-based TF [5451-distance-based-transfer-function.md] and Size-Based Transfer Functions [5845-size-based-transfer-functions.md] by utilizing a high-dimensional feature space (20 metrics) to separate structures with similar intensities but different textural properties [S1]. Unlike the 1D data-based TF [13183-feature-based-transfer-functions.md], it requires significant preprocessing to compute these metrics, making it unsuitable for real-time interactive rendering without optimization [S1].

## How it works
The method extends the standard 1D transfer function to a multi-dimensional domain $T(v, f_1, \dots, f_{20}) \rightarrow (c, \alpha)$, where $v$ is the scalar data value and $f_i$ are the 20 texture metrics [S1].

### Inputs and Outputs
*   **Input:** A scalar volume $V$.
*   **Output:** A volume rendering with color $c$ and opacity $\alpha$ determined by the texture-based mapping.

### Data Structures
The preprocessing phase generates auxiliary volumes required for the transfer function evaluation [S1]:
1.  **Metric Volumes:** 20 separate 3D volumes, one for each statistical metric.
2.  **GLCM Volume:** 1 volume holding Gray Level Co-occurrence Matrix data.
3.  **Run-Length Volume:** 1 volume holding run-length matrix data.
4.  **Label Volume (Optional):** Used in the fully automatic method to store cluster labels [S1].

### Parameters
*   **Metrics:** 20 texture metrics consisting of:
    *   6 First-order statistics.
    *   7 Second-order statistics.
    *   7 Run-length matrices [S1].
*   **Resolution:** The texture analysis does not strictly require full volume resolution; lower resolution can be used to trade precision for speed [S1].

### Specification Methods
The tbTF offers four specification modes [S1]:
1.  **Fully Automatic:** Uses clustering to recognize structures. The user inputs only the number of structures. This method is barely controllable, has poor ability to emphasize specific features, and carries a high risk of misinterpretation [S1].
2.  **Semi-Automatic (Manual Weighting):** The user manually weights individual statistics. This requires high understanding of the metrics and is time-consuming [S1].
3.  **Semi-Automatic (Two Structures):** The user specifies two structures to be distinguished [S1].
4.  **Semi-Automatic (Single Structure):** The user marks a single structure to be separated from the rest [S1].

### Algorithm Steps
1.  **Preprocessing:** Compute the 20 texture metrics for the input volume $V$. This involves calculating first-order, second-order, and run-length statistics. Generate the GLCM and run-length volumes [S1].
2.  **Clustering (Automatic Mode Only):** If using the fully automatic method, apply clustering to the metric volumes to assign labels to voxels based on the specified number of structures [S1].
3.  **Rendering Setup:**
    *   If using the automatic method, load the Label Volume and the original Data Volume.
    *   If using semi-automatic methods, load the 20 Metric Volumes and the Data Volume.
4.  **Raycasting Loop:** For each sample point $\mathbf{x}$ along the ray:
    *   Sample the scalar value $v = V(\mathbf{x})$.
    *   Sample the relevant feature values $f_i$ from the precomputed volumes.
        *   In automatic mode: Sample the label $L$ and data $v$.
        *   In semi-automatic mode: Sample $v$ and the 20 metrics $f_1, \dots, f_{20}$ [S1].
    *   Query the transfer function $T(v, f_1, \dots, f_{20})$ or $T(v, L)$ to obtain color $c$ and opacity $\alpha$ [S1].
    *   Composite using front-to-back integration [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def compute_texture_metrics(volume):
    """
    Computes 20 texture metrics for the input volume.
    Returns a list of 20 volumes and 2 auxiliary volumes (GLCM, Run-Length).
    """
    metrics = []
    # Compute 6 first-order statistics
    metrics.append(compute_first_order_stat_1(volume))
    metrics.append(compute_first_order_stat_2(volume))
    metrics.append(compute_first_order_stat_3(volume))
    metrics.append(compute_first_order_stat_4(volume))
    metrics.append(compute_first_order_stat_5(volume))
    metrics.append(compute_first_order_stat_6(volume))
    
    # Compute 7 second-order statistics
    metrics.append(compute_second_order_stat_1(volume))
    metrics.append(compute_second_order_stat_2(volume))
    metrics.append(compute_second_order_stat_3(volume))
    metrics.append(compute_second_order_stat_4(volume))
    metrics.append(compute_second_order_stat_5(volume))
    metrics.append(compute_second_order_stat_6(volume))
    metrics.append(compute_second_order_stat_7(volume))
    
    # Compute 7 run-length matrices
    metrics.append(compute_run_length_1(volume))
    metrics.append(compute_run_length_2(volume))
    metrics.append(compute_run_length_3(volume))
    metrics.append(compute_run_length_4(volume))
    metrics.append(compute_run_length_5(volume))
    metrics.append(compute_run_length_6(volume))
    metrics.append(compute_run_length_7(volume))
    
    glcm_volume = compute_glcm_volume(volume)
    run_length_volume = compute_run_length_volume(volume)
    
    return metrics, glcm_volume, run_length_volume

def cluster_metrics(metrics, num_structures):
    """
    Applies clustering to the metric volumes to generate labels.
    """
    # Flatten metrics into feature vectors
    feature_vectors = []
    for x in range(volume_width):
        for y in range(volume_height):
            for z in range(volume_depth):
                vec = [m[x, y, z] for m in metrics]
                feature_vectors.append(vec)
    
    # Perform clustering (e.g., K-Means)
    labels = kmeans_cluster(feature_vectors, num_structures)
    
    # Reshape labels back to 3D volume
    label_volume = reshape_to_3d(labels, volume_shape)
    return label_volume

# Rendering Phase
def render_tbtf(volume, metrics, label_volume=None, mode='semi-automatic'):
    """
    Renders the volume using the Texture-based Transfer Function.
    """
    output_image = initialize_output_image()
    
    for each ray:
        color_accum = (0, 0, 0, 0)
        alpha_accum = 0
        
        for sample_point along ray:
            v = sample_volume(volume, sample_point)
            
            if mode == 'automatic':
                # Use label and data value
                l = sample_volume(label_volume, sample_point)
                c, alpha = query_tf_data_label(v, l)
            else:
                # Use data value and 20 metrics
                f_values = [sample_volume(m, sample_point) for m in metrics]
                c, alpha = query_tf_metrics(v, f_values)
            
            # Front-to-back compositing
            weight = alpha * (1 - alpha_accum)
            color_accum += weight * c
            alpha_accum += weight
            
            if alpha_accum > 1.0:
                break
                
        output_image[ray_id] = color_accum
        
    return output_image
```

### Edge Cases and Pitfalls
*   **Noise:** Texture analysis is highly sensitive to noise. The dataset must be very clean; otherwise, the texture metrics become unreliable [S1].
*   **Automatic Mode Risk:** The fully automatic clustering method is barely controllable and has a high risk of producing wrong results or critical misinterpretations due to poor feature emphasis [S1].
*   **Complexity:** Computing the metrics and performing clustering is very expensive, leading to high preprocessing times and memory consumption [S1].

## Operating characteristics
*   **Time Complexity:** Preprocessing is very expensive due to the computation of 20 metrics and potential clustering [S1]. Rendering time is comparable to other TFs if using labels, but higher if using raw metrics [S1].
*   **Memory Complexity:** Requires up to 20 additional 3D volumes for metrics, plus GLCM and run-length volumes during preprocessing [S1]. During rendering, if using the automatic method, only 1 additional volume (label) is needed, which does not necessarily need to be full size [S1].
*   **Real-time Capability:** No. It requires 2 3D texture lookups per sample if using labels (data + label) or 21 lookups if using metrics (data + metrics) [S1].
*   **Strengths:** Can distinguish features based on texture, allowing separation of structures with similar intensities but different textural characteristics (e.g., left vs. right features in Figure 6 of [S1]) [S1].
*   **Limitations:** High computational cost, poor controllability in automatic mode, and sensitivity to noise [S1].
*   **When to use:** When features cannot be distinguished by intensity or gradient alone and have distinct textural properties, and interactive speed is not required [S1].
*   **When not to use:** For real-time applications or noisy data [S1].

## Background
The Texture-based Transfer Function was introduced by [CR08] and is compared against other TF types (1D, gradient-based, curvature-based, size-based, distance-based) in a survey by [S1]. It is particularly useful in medical visualization for distinguishing anatomical structures based on tissue texture [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
