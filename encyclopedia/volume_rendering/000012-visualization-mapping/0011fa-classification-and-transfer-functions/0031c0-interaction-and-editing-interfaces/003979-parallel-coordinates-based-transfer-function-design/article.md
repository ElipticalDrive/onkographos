# Parallel Coordinates based Transfer Function Design

Also called: PCbTF Design, Parallel Coordinates based Transfer Function (PCbTF)

## Overview
Parallel Coordinates based Transfer Function Design (PCbTF) is a method for designing multi-dimensional transfer functions by visualizing high-dimensional data parameters and spatial attributes as polylines in a parallel coordinates plot [S1]. It solves the complexity of high-dimensional classification by allowing users to identify correlations and select parameters through polyline patterns, or by automatically reducing dimensionality using Local Linear Embedding (LLE) for k-means classification [S1]. Unlike methods that operate solely in parameter space, PCbTF integrates spatial information as explicit parameters and leverages the structural patterns of polylines to facilitate dimension reduction and transfer function primitive design [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (e.g., CT or MRI) and a set of seed points for background removal [S1].
*   **Output:** A multi-dimensional transfer function defined by widgets (primitives) in the parallel coordinates space, mapping voxel attributes to opacity and color [S1].

**Data Structures:**
*   **Voxel Attributes:** For each non-background voxel, a vector of 16 statistical attributes is computed [S1].
*   **Parallel Coordinates Plot (PCP):** A visualization where each axis represents one attribute and each voxel is represented as a polyline connecting points on these axes [S1].
*   **Sorting Order:** A permutation of the 16 axes, optimized via Simulated Annealing to minimize visual clutter [S1].
*   **TF Primitives:** Defined as geometric constraints (lines, rectangles, polygons, planes) or filter curves (Gaussian, Laplacian) spanning one or more axes [S1].

**Numbered Steps:**

1.  **Parameter Calculation:** Calculate 16 statistical attributes for each voxel in the input dataset [S1]. The attributes are: angular second moment, contrast, correlation, variance, inverse difference moment, individual entropy, sum average, sum variance, sum entropy, skewness, kurtosis, correlation information measurements, intensity, gradient, and second order derivative [S1].
2.  **Background Removal:** Apply a region growing method starting from selected seed points to identify and remove background voxels (e.g., air in CT/MRI datasets) [S1]. This reduces computational load and removes irrelevant polylines [S1].
3.  **Noise Removal:** Identify and erase outlier polylines from the PCP to remove randomly distributed noise [S1].
4.  **Axis Sorting (Simulated Annealing):** Sort the axes in the PCP to minimize internal energy (clutter), which reveals correlations between neighboring parameters [S1].
    *   **Energy Function:** The internal energy $f(h)$ between neighboring coordinates is calculated as:
        $$f(h)=\frac{1}{\sum_{i=1}^{n}(p_i - \frac{1}{N})^2}$$
        where $N = N_1 \times N_2$ is the pre-defined number of bins for the 2D joint histogram of adjacent coordinates, $p_i$ is the probability of the $i$-th bin, and $p_i = \frac{l_i}{\sum_{j=1}^{N} l_j}$, with $l_i$ being the number of polylines in the $i$-th bin [S1].
    *   **Optimization:** Use Simulated Annealing to find the permutation of axes that minimizes $f(h)$ [S1]. The algorithm initializes temperature $T \to T_0$ and sequence $X \to X_0$ [S1]. In each iteration, it swaps two axes $k$ and $m$ ($k < m$) to create $X_j$ [S1]. If the energy difference $df = f(X_j) - f(X_i) < 0$, the new sequence is accepted [S1]. If $df > 0$, it is accepted with probability $\exp(-df/T_i)$ [S1]. The complexity is $O(n \cdot p \cdot 2^n)$, where $n$ relates to the number of parameters and $p$ is a factor in energy calculation [S1].
5.  **Parameter Selection:** Identify correlated patterns in the sorted PCP (e.g., inverse linear, direct linear, elliptical, hyperbolic, quadratic, or parabola correlations) and remove redundant parameters either manually or via dimension reduction [S1].
6.  **Transfer Function Design (Interactive Path):**
    *   **Primitives:** Design TF primitives in the PC space [S1]. These include: 2D lines (defined by slope $b$ and rate $a$) [S1], 2D rectangles and polygons (defined by control points) [S1], 3D planes (defined by intersection points of vertical dashed lines) [S1], and 3D plane scaling [S1].
    *   **Filters:** Apply Gaussian or Laplacian filter widgets, drawn as curves in neighboring coordinates, to adjust opacity smoothly [S1].
    *   **nD Extension:** Extend 2D/3D patterns to nD by setting constraints between neighboring coordinates: $x_2 = f_1(x_1), x_3 = f_2(x_2), \dots, x_n = f_{n-1}(x_{n-1})$ [S1]. Adjusting the first parameter $x_1$ automatically controls the selection of subsequent parameters based on these pre-defined functions [S1].
7.  **Transfer Function Design (Automatic Path):**
    *   **Dimension Reduction:** Project all attribute parameters to a 2D space using Local Linear Embedding (LLE) [S1].
    *   **Classification:** Apply k-means clustering in the 2D reduced space to identify classes [S1].
    *   **Mapping:** Assign colors and opacities to the resulting classes [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def pc_tf_design(volume_data, seeds):
    """Main pipeline for Parallel Coordinates based Transfer Function Design [S1]."""
    # Step 1: Calculate 16 statistical attributes for each voxel [S1]
    attributes = compute_voxel_attributes(volume_data)
    
    # Step 2: Remove background voxels using region growing from seeds [S1]
    voxel_mask = region_growing(volume_data, seeds)
    active_voxels = [v for v in volume_data if voxel_mask[v]]
    active_attributes = [attributes[v] for v in active_voxels]
    
    # Step 3: Remove noise by erasing outlier polylines [S1]
    active_attributes = remove_outlier_polylines(active_attributes)
    
    # Step 4: Sort axes using Simulated Annealing to minimize internal energy [S1]
    num_params = len(active_attributes[0])
    initial_order = list(range(num_params))
    sorted_order = simulated_annealing_sort(active_attributes, initial_order)
    
    # Step 5: Select parameters by identifying correlated patterns [S1]
    selected_indices = sorted_order  # Manual or automatic selection logic
    
    # Step 6/7: Design TF (Interactive or Automatic LLE path) [S1]
    # Interactive path: design_widgets_interactive(active_attributes, selected_indices)
    # Automatic path: reduced_data = local_linear_embedding(active_attributes, target_dims=2); classes = k_means(reduced_data); tf_widgets = map_classes_to_colors_opacities(classes)
    # Note: The sources do not provide implementation details for the interactive widget design or automatic LLE/k-means mapping steps.
    return sorted_order, selected_indices

def compute_voxel_attributes(volume):
    """Computes 16 statistical attributes per voxel [S1]."""
    attrs = []
    for voxel in volume:
        # Standard statistical calculations for local neighborhoods [G]
        attrs.append([
            angular_second_moment(voxel), contrast(voxel), correlation(voxel),
            variance(voxel), inverse_difference_moment(voxel), individual_entropy(voxel),
            sum_average(voxel), sum_variance(voxel), sum_entropy(voxel),
            skewness(voxel), kurtosis(voxel), correlation_information_measurements(voxel),
            intensity(voxel), gradient_magnitude(voxel), second_order_derivative(voxel)
        ])
    return attrs

def region_growing(volume, seeds):
    """Standard flood-fill region growing to identify background voxels [G]."""
    mask = [False] * len(volume)
    queue = list(seeds)
    while queue:
        idx = queue.pop(0)
        if not mask[idx]:
            mask[idx] = True
            for neighbor in get_neighbors(idx, volume):
                if not mask[neighbor] and is_similar_intensity(volume[idx], volume[neighbor]):
                    queue.append(neighbor)
    return mask

def remove_outlier_polylines(attributes):
    """Filters out polylines that deviate significantly from the main distribution [G]."""
    median = compute_median_polyline(attributes)
    THRESHOLD = 2.0  # [G]
    return [attr for attr in attributes if distance_to_median(attr, median) < THRESHOLD]

def simulated_annealing_sort(attributes, initial_order):
    """Sorts axes to minimize internal energy f(h) [S1]."""
    T0 = 100.0  # [G]
    Tmin = 0.01  # [G]
    dT = 0.95  # [G]
    N1, N2 = 10, 10  # Pre-defined bin numbers [S1]
    
    T = T0
    current_order = initial_order
    best_order = initial_order
    
    while T > Tmin:
        k, m = random_swap_indices(len(current_order))
        new_order = swap(current_order, k, m)
        
        energy_current = calculate_energy(attributes, current_order, N1, N2)
        energy_new = calculate_energy(attributes, new_order, N1, N2)
        df = energy_new - energy_current
        
        if df < 0:
            current_order = new_order
            best_order = new_order
        elif math.exp(-df / T) > random.random():
            current_order = new_order
            
        T *= dT
        
    return best_order

def calculate_energy(attributes, order, N1, N2):
    """Calculates internal energy f(h) based on joint histogram distribution [S1]."""
    total_energy = 0.0
    for i in range(len(order) - 1):
        axis1 = order[i]
        axis2 = order[i+1]
        histogram = build_joint_histogram(attributes, axis1, axis2, N1, N2)
        total_polylines = sum(histogram)
        if total_polylines == 0:
            continue
        probs = [l / total_polylines for l in histogram]
        N = N1 * N2
        sum_sq_diff = sum((p - 1.0 / N) ** 2 for p in probs)
        if sum_sq_diff > 0:
            total_energy += 1.0 / sum_sq_diff
    return total_energy

def build_joint_histogram(attributes, axis1, axis2, N1, N2):
    """Builds 2D joint histogram for two axes [G]."""
    hist = [0] * (N1 * N2)
    for attr in attributes:
        b1 = int((attr[axis1] - min(attr[axis1])) / (max(attr[axis1]) - min(attr[axis1]) + 1e-9) * N1)
        b2 = int((attr[axis2] - min(attr[axis2])) / (max(attr[axis2]) - min(attr[axis2]) + 1e-9) * N2)
        b1 = max(0, min(N1 - 1, b1))
        b2 = max(0, min(N2 - 1, b2))
        hist[b1 * N2 + b2] += 1
    return hist

def swap(order, k, m):
    """Swaps elements at indices k and m in a list."""
    new_order = order.copy()
    new_order[k], new_order[m] = new_order[m], new_order[k]
    return new_order

def random_swap_indices(n):
    """Returns two distinct random indices k and m where k < m."""
    k, m = sorted(random.sample(range(n), 2))
    return k, m

def get_neighbors(idx, volume):
    """Returns adjacent voxel indices in a 3D grid [G]."""
    # Standard 6-connected neighborhood logic [G]
    return [idx - 1, idx + 1, idx - len(volume[0]), idx + len(volume[0])]

def is_similar_intensity(v1, v2):
    """Checks if two voxels have similar intensity values [G]."""
    return abs(v1[12] - v2[12]) < 10.0  # Using intensity index [G]

def compute_median_polyline(attributes):
    """Computes the median value for each attribute across all polylines [G]."""
    n_attrs = len(attributes[0])
    median = []
    for j in range(n_attrs):
        col = sorted([attr[j] for attr in attributes])
        median.append(col[len(col) // 2])
    return median

def distance_to_median(attr, median):
    """Computes Euclidean distance between a polyline and the median polyline [G]."""
    return math.sqrt(sum((a - m) ** 2 for a, m in zip(attr, median)))
```

**Parameters:**
*   **Initial Temperature ($T_0$):** Not given in the sources. [G]
*   **Minimum Temperature ($T_{min}$):** Not given in the sources. [G]
*   **Temperature Decay ($T_d$):** Not given in the sources. [G]
*   **Bin Numbers ($N_1, N_2$):** Pre-defined values for the 2D joint histogram of adjacent coordinates. Specific values are not given in the sources. [S1]
*   **Number of Attributes:** 16 (15 statistical + intensity) [S1].

**Edge Cases and Pitfalls:**
*   **Background Noise:** If background voxels are not removed, they significantly affect visualization and computational speed [S1].
*   **Local Optimization:** Random swapping in sorting can suffer from repetition; nearest neighbor and greedy algorithms may get stuck in local optima [S1].
*   **Dimensionality:** High-dimensional TF design is complex and impossible to draw directly in orthogonal coordinates [S1].

## Operating characteristics
*   **Time Complexity:** The sorting step using Simulated Annealing has a complexity of $O(n \cdot p \cdot 2^n)$, where $n$ is related to the number of parameters and $p$ is a factor in the energy calculation [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   Combines spatial space (voxel position) and parameter space [S1].
    *   Allows analysis of polyline structural patterns for dimension reduction [S1].
    *   Provides a simple interface for designing high-dimensional transfer functions [S1].
    *   Can emphasize details difficult to visualize by other approaches [S1].
*   **Limitations:**
    *   Requires extra work such as sorting coordinates [S1].
    *   Computational burden can be high without dimension reduction [S1].
*   **When to Use:** Suitable for multi-dimensional transfer function design where users need to analyze correlations between high-dimensional parameters and spatial data, particularly in medical imaging (e.g., bladder cancer detection, MRI prostate zone detection) [S1].

## Background
*   **Origin:** The method was proposed in a paper titled "Multi-dimensional Reduction and Transfer Function Design using Parallel Coordinates" [S1].
*   **History:** It builds on the use of parallel coordinates for visualizing high-dimensional geometry and multivariate data [S1]. It extends previous work by adding spatial information as new parameters and using LLE for automatic dimension reduction [S1].
*   **Usage:** Applied to bladder cancer detection (highlighting high intensity/gradient areas) and MRI prostate zone detection (enhancing opacity/color in specific intensity/gradient regions) [S1].

## Sources

[S1] Multi-dimensional Reduction and Transfer Function Design using Parallel Coordinates, https://doi.org/10.2312/vg/vg10/069-076, fetched 2026-10-26
[S2] Multi-dimensional Reduction and Transfer Function Design using Parallel ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC4536824/, fetched 2026-10-27
