# Partial Region Growing for Transfer Function Generation

Also called: Derived Transfer Function, Partial Growing, Partial Region Growing for Transfer Function Derivation, Partial Region Growing with Derived Transfer Functions, Semi-automatic feature extraction, Transfer function derivation from region growing

## Overview
Partial Region Growing for Transfer Function Generation is a semi-automatic technique that derives a 2D transfer function from the statistical properties of a partially segmented region to visualize similar, disconnected features throughout the entire volume [S1][S2, unofficial]. Unlike full region growing which segments all connected components, this method performs growth on only a subset of the data or a small subset of regions, significantly reducing computation time while providing a robust classifier for the whole dataset [S2, unofficial]. It is particularly useful when features of the same type are disconnected in the volume, preventing a single region growing pass from capturing all instances [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset $V$ with scalar values and computed gradient magnitudes; a seed point or initial region for partial growth; a growth limit (percentage or voxel count).
*   **Output:** A 2D transfer function (classifier) defined on the scatterplot of data values vs. gradient magnitudes, ready for volume rendering.

**Data Structures**
*   **Volume Data:** A 3D array of scalar values. Gradient magnitudes are pre-computed or computed on-the-fly.
*   **Region Growing State:** A queue or stack of voxels to process, and a visited mask to prevent re-processing.
*   **Statistical Accumulators:** Variables to store the sum, sum of squares, count, min, and max of data values and gradient magnitudes for the extracted region.
*   **Classifier Representation:**
    *   *Rectangular:* Defined by center $(\mu_v, \mu_g)$ and size $(\sigma_v, \sigma_g)$.
    *   *Polygonal/Histogram-based:* A set of merged regions in the 2D histogram space, constructed via k-d tree partitioning.

**Steps**

1.  **Initialize Partial Region Growing:**
    Select a seed voxel or small initial region within the volume. Initialize the region growing algorithm (e.g., using a connectivity criterion based on data value similarity and gradient magnitude thresholds).

2.  **Execute Partial Growth:**
    Perform region growing on the volume, but stop when a predefined extent is reached. The extent can be defined as:
    *   A fixed number of voxels.
    *   A percentage of the expected full region size (typically 20-40% of the full region growing result is sufficient) [S2, unofficial].
    *   A time limit.
    Let $R$ be the set of voxels extracted during this partial growth.

3.  **Compute Statistical Properties:**
    Calculate the following statistics for the extracted region $R$:
    *   $\mu_v$: Average data value of voxels in $R$.
    *   $\mu_g$: Average gradient magnitude of voxels in $R$.
    *   $\sigma_v$: Standard deviation of data values in $R$.
    *   $\sigma_g$: Standard deviation of gradient magnitudes in $R$.

    $$ \mu_v = \frac{1}{|R|} \sum_{v \in R} \text{value}(v) $$
    $$ \mu_g = \frac{1}{|R|} \sum_{v \in R} \text{gradient}(v) $$
    $$ \sigma_v = \sqrt{\frac{1}{|R|} \sum_{v \in R} (\text{value}(v) - \mu_v)^2} $$
    $$ \sigma_g = \sqrt{\frac{1}{|R|} \sum_{v \in R} (\text{gradient}(v) - \mu_g)^2} $$

4.  **Generate Transfer Function (Classifier):**
    Two methods are described for constructing the classifier from the statistics:

    *   **Method A: Rectangular Classifier (Simple)**
        Define a rectangular region on the 2D scatterplot (Data Value vs. Gradient Magnitude).
        *   Center coordinates: $(\mu_v, \mu_g)$.
        *   Size (width/height): Defined by $\sigma_v$ and $\sigma_g$.
        This rectangle serves as the classification function. Voxels whose (value, gradient) pair falls within this rectangle are considered part of the feature [S1][S2, unofficial].

    *   **Method B: Histogram-Based Classifier (Robust)**
        1.  Partition the 2D histogram region defined by the partial growing results using a k-d tree.
        2.  For each leaf node of the k-d tree, sort the data points by their value.
        3.  Construct a local classifier for each leaf node using data points with values over a pre-selected threshold. Discard other voxels as noise.
        4.  Recursively merge local classifiers by traversing the k-d tree. Two classifiers are merged if the new combined classifier contains most of the data points and its area is similar to or less than the sum of the areas of the original two classifiers.
        5.  Assign an opacity map and color map to the resulting classifier. The opacity map can be constructed from the feature's histogram data or predefined functions (linear, Gaussian) [S2, unofficial].

5.  **Apply Transfer Function:**
    Apply the derived 2D transfer function to the entire volume. This reveals other features that match the statistical profile of the partially extracted region, even if they are disconnected from the seed [S1][S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_region_growing_transfer_function(volume, seed_voxel, growth_limit_percent):
    """
    Derives a 2D transfer function from partial region growing.
    
    Args:
        volume: 3D array of scalar values.
        seed_voxel: (x, y, z) coordinates of the starting voxel.
        growth_limit_percent: Percentage of full region growth to perform (e.g., 0.2 for 20%).
        
    Returns:
        transfer_function: A function or data structure representing the 2D classifier.
    """
    # Step 1: Estimate full region size (optional, for limit calculation)
    # In practice, growth_limit_percent might be applied to a pre-estimated size
    # or used as a hard stop condition based on time/voxel count.
    # For this implementation, we assume a max_voxel_count is derived from 
    # a quick heuristic or previous run, or we simply run until a time limit.
    # Here we simulate the limit by stopping after a certain number of voxels.
    
    # Step 2: Perform Partial Region Growing
    extracted_region = []
    queue = [seed_voxel]
    visited = set()
    visited.add(seed_voxel)
    
    # Heuristic: Assume we want to grow until we hit a limit.
    # The limit is often defined as a percentage of the *expected* full region.
    # If the expected full region size is unknown, we might use a fixed voxel count
    # or time limit. The sources state 20-40% is often sufficient.
    # Let's assume we have a function to estimate full size or a fixed limit.
    # For generality, we'll just run a standard region growing loop 
    # and stop when a condition is met (e.g., time or voxel count).
    
    # Note: The exact stopping condition for "partial" is implementation-specific.
    # Common approach: Run for a fixed time or until N voxels are processed.
    
    max_voxels = estimate_full_region_size(volume, seed_voxel) * growth_limit_percent
    
    while queue and len(extracted_region) < max_voxels:
        current = queue.pop(0)
        extracted_region.append(current)
        
        neighbors = get_neighbors(current, volume.shape)
        for neighbor in neighbors:
            if neighbor not in visited:
                # Standard region growing connectivity check
                if is_connected(current, neighbor, volume):
                    visited.add(neighbor)
                    queue.append(neighbor)
    
    # Step 3: Compute Statistics
    if not extracted_region:
        raise ValueError("No voxels extracted")
        
    values = [volume[v] for v in extracted_region]
    gradients = [compute_gradient_magnitude(volume, v) for v in extracted_region]
    
    mu_v = sum(values) / len(values)
    mu_g = sum(gradients) / len(gradients)
    
    sigma_v = (sum((v - mu_v)**2 for v in values) / len(values))**0.5
    sigma_g = (sum((g - mu_g)**2 for g in gradients) / len(gradients))**0.5
    
    # Step 4: Generate Rectangular Classifier
    # Center: (mu_v, mu_g)
    # Size: (sigma_v, sigma_g)
    # This defines a rectangle in the 2D scatterplot.
    
    def rectangular_classifier(data_value, gradient_value):
        # Check if point is within the rectangle
        # Assuming the rectangle is centered at mean with half-widths equal to std dev
        # Or full width/height equal to std dev? Sources say "size is defined by standard deviations".
        # Typically, this implies a range of [mu - sigma, mu + sigma] or similar.
        # Let's assume a range of +/- 1 sigma for simplicity, or as defined by the user.
        # The source says "two sigma values define the size of the region".
        # This is ambiguous. It could mean the width is sigma, or 2*sigma.
        # Given it's a "classifier", a common interpretation is a box around the mean.
        # We will define it as a box from mu-sigma to mu+sigma.
        
        val_range = [mu_v - sigma_v, mu_v + sigma_v]
        grad_range = [mu_g - sigma_g, mu_g + sigma_g]
        
        return val_range[0] <= data_value <= val_range[1] and \
               grad_range[0] <= gradient_value <= grad_range[1]

    return rectangular_classifier

def estimate_full_region_size(volume, seed):
    """
    Placeholder for estimating the full region size.
    In practice, this might be done by a quick full grow or heuristic.
    [G] This is a helper function not detailed in sources, but necessary for the limit.
    """
    # Simple heuristic: assume a fixed large number or run a quick grow
    # For the purpose of this article, we note that the limit is a percentage
    # of the full region.
    return 10000 # Placeholder
```

**Parameters**
*   **Growth Extent:** 20-40% of the full region growing result is often sufficient to derive a good transfer function [S2, unofficial].
*   **Thresholds:** Pre-selected thresholds for noise discarding in the histogram-based method [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Noise:** If the partial region includes noisy voxels, the statistical averages and standard deviations may be skewed, leading to an inaccurate transfer function. The histogram-based method (Method B) is more robust to noise as it discards low-value points [S2, unofficial].
*   **Disconnected Features:** If the seed region is not representative of the other features (e.g., different material properties), the derived transfer function will fail to capture them [S1].
*   **Precision:** Standard deviation calculations can be sensitive to outliers. Using robust statistics (median, MAD) might be beneficial but is not specified in the sources [G].

## Subtopics

### RGVis Region Growing
The parent method that provides the interactive framework for region growing in volume visualization. Partial Region Growing is an optimization within this framework. [RGVis Region Growing](14368-rgvis-region-growing.md)

### Histogram-Based Classifier Generation
An alternative to the rectangular classifier that uses k-d tree partitioning and merging for more robust feature extraction. This is a sub-method of Partial Region Growing for Transfer Function Generation.

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** Partial region growing is significantly faster than full region growing. On a 1.8GHz Pentium 4, partial growing (20-40%) took under 0.2 seconds, while full growing took 2.8-8.1 seconds for various datasets [S2, unofficial]. The transfer function generation step is negligible in comparison.
*   **Memory:** Requires memory for the volume data, the visited mask, and the queue for region growing. The classifier storage is minimal (4 values for rectangular, or a set of regions for histogram-based).

**Measured Performance**
*   **Confocal Microscopy ganglion (55x512x512):** Full=5.7s, Partial=0.022s [S2, unofficial].
*   **MRA head vasculature (125x250x250):** Full=7.2s, Partial=0.028s [S2, unofficial].
*   **MRI head (128^3):** Full=2.8s, Partial=0.34s [S2, unofficial].
*   **MRI Chest (256^3):** Full=8.1s, Partial=0.12s [S2, unofficial].

**Strengths and Limitations**
*   **Strengths:**
    *   Drastically reduces computation time for feature extraction [S2, unofficial].
    *   Automatically suggests a transfer function, reducing manual trial-and-error [S1].
    *   Effective for visualizing disconnected features of the same type [S1].
    *   The histogram-based classifier is robust to noise [S2, unofficial].
*   **Limitations:**
    *   Requires a representative seed region. If the seed is not typical, the transfer function will be inaccurate [S1].
    *   The rectangular classifier may be too simplistic for complex feature distributions; the histogram-based method is more complex to implement [S2, unofficial].
    *   Does not provide a geometric surface representation directly (though this can be generated from the segmented volume) [S1].

**When to Use**
*   Use when features are disconnected and full region growing is too slow or captures unwanted noise.
*   Use when a quick exploration of similar features is needed.
*   Use when the user wants to avoid manual tuning of the transfer function.

**When Not to Use**
*   Use when features are highly heterogeneous and a single transfer function cannot represent them.
*   Use when a precise geometric surface is required immediately (though this can be derived later).

## Background
This method was developed as part of the RGVis system for visualizing industrial CT volume data for nondestructive testing (NDT) applications [S1]. It was introduced to lower the cost of segmentation and facilitate volumetric feature extraction by coupling region growing techniques with a 2D histogram interface [S1]. The technique allows users to quickly identify and display relevant features in NDT data, which often contain concealed structures and noise [S1]. The method was further detailed in the RGVis thesis, which explored various region growing based visualization techniques [S2, unofficial].

## Sources
[S1] Visualizing industrial CT volume data for nondestructive testing applications, https://doi.org/10.1109/visual.2003.1250418, fetched 2026-10-27
[S2] RGVis: Region Growing Based Visualization Techniques for Volume Visualization, https://escholarship.org/uc/item/3td71868, fetched 2026-11-03 (unofficial)
