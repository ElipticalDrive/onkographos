# Spatial Partitioning for Distributed NeRF

Also called: Aspect Ratio Optimization Partitioning, Recursive CDF Partitioning

## Overview
This method partitions the 3D scene volume into non-overlapping tiles for distributed NeRF training by recursively splitting space based on a Probability Density Function (PDF) derived from a point cloud or ray samples, optimizing for cubic aspect ratios [S1]. Unlike Block-NeRF, which partitions based on training cameras, this method partitions space directly, allowing for finer bounding boxes in regions with more content or frequent capture (level-of-detail property) [S1].

## How it works
**Inputs:**
1.  Training data: Either a sparse point cloud from Structure from Motion (SfM) or a set of discretized training rays [S1].
2.  Target number of partitions: $2^n$ (where $n$ is the number of recursive iterations) [S1].

**Outputs:**
A set of $2^n$ non-overlapping axis-aligned bounding boxes (tiles) that partition the scene volume [S1].

**Data Structures:**
*   **Point Cloud / Samples:** A list of 3D points $\mathbf{p}_i$ or ray sample positions.
*   **PDF:** A histogram or kernel density estimate representing the spatial distribution of points along each axis.
*   **CDF:** The cumulative distribution function derived from the PDF for each axis.

**Algorithm Steps:**

1.  **Generate Point Cloud:**
    *   If SfM data is available, use the sparse point cloud [S1].
    *   If SfM data is unavailable (e.g., Building capture), discretize training rays into samples to form the point cloud [S1].

2.  **Construct PDF:**
    *   Compute the Probability Density Function (PDF) for the distribution of the point cloud along the x, y, and z axes [S1].

3.  **Recursive Splitting (for $k = 0$ to $n-1$):**
    For each existing partition (initially the whole volume):
    a.  **Compute CDF:** Calculate the Cumulative Distribution Function (CDF) along each axis (x, y, z) for the points within the current partition [S1].
    b.  **Identify Candidate Planes:** Find the plane position along each axis where the CDF equals 0.5. This position represents the median, signifying an optimal separation that evenly divides the spatial content into two partitions [S1].
    c.  **Select Best Axis:** For each candidate plane (one per axis), calculate the aspect ratio ($AR$) of the resulting two sub-partitions. The aspect ratio $AR_i$ for a partition is defined by its dimensions. Select the axis (x, y, or z) that yields partitions with aspect ratios closest to 1 (i.e., most cubic) [S1].
    d.  **Split:** Divide the current partition along the selected plane.
    e.  **Repeat:** Apply steps a-d recursively to the new sub-partitions until $2^n$ partitions are achieved [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatial_partition(points, n_iterations):
    """
    Recursively partition points into 2^n tiles based on CDF median and aspect ratio.
    
    Args:
        points: List of 3D points (numpy array or list of tuples).
        n_iterations: Number of recursive split iterations.
        
    Returns:
        List of bounding boxes (min_x, min_y, min_z, max_x, max_y, max_z).
    """
    # Initial partition: bounding box of all points
    current_partitions = [compute_bounding_box(points)]
    
    for iteration in range(n_iterations):
        next_partitions = []
        for partition in current_partitions:
            # Extract points within current partition
            partition_points = filter_points_in_box(points, partition)
            
            if len(partition_points) == 0:
                # Handle empty partition by duplicating or skipping
                # [G] Fallback: split geometrically if no points
                next_partitions.append(split_geometrically(partition))
                continue
            
            # Compute CDF along each axis
            cdf_x = compute_cdf(partition_points, axis='x')
            cdf_y = compute_cdf(partition_points, axis='y')
            cdf_z = compute_cdf(partition_points, axis='z')
            
            # Find median planes (CDF = 0.5)
            plane_x = find_median_plane(cdf_x, partition)
            plane_y = find_median_plane(cdf_y, partition)
            plane_z = find_median_plane(cdf_z, partition)
            
            # Calculate aspect ratios for each candidate split
            ar_x = compute_aspect_ratio_after_split(partition, plane_x, axis='x')
            ar_y = compute_aspect_ratio_after_split(partition, plane_y, axis='y')
            ar_z = compute_aspect_ratio_after_split(partition, plane_z, axis='z')
            
            # Select axis with AR closest to 1
            best_axis = argmin([ar_x, ar_y, ar_z])
            best_plane = [plane_x, plane_y, plane_z][best_axis]
            
            # Split partition
            p1, p2 = split_partition(partition, best_plane, best_axis)
            next_partitions.extend([p1, p2])
            
        current_partitions = next_partitions
        
    return current_partitions

def compute_cdf(points, axis):
    """Compute normalized CDF for points along a specific axis."""
    coords = [p[axis] for p in points]
    coords.sort()
    n = len(coords)
    # [G] Simple histogram-based CDF or linear interpolation
    # Returns array of (coordinate, cumulative_probability)
    return [(coords[i], (i + 1) / n) for i in range(n)]

def find_median_plane(cdf, partition):
    """Find coordinate where CDF is closest to 0.5."""
    # [G] Binary search or linear scan on CDF array
    for coord, prob in cdf:
        if prob >= 0.5:
            return coord
    return partition.max_coord

def compute_aspect_ratio_after_split(partition, plane, axis):
    """Compute max aspect ratio of the two resulting sub-partitions."""
    p1, p2 = split_partition(partition, plane, axis)
    ar1 = max(p1.dimensions) / min(p1.dimensions)
    ar2 = max(p2.dimensions) / min(p2.dimensions)
    return max(ar1, ar2)

def split_partition(partition, plane, axis):
    """Split bounding box along plane."""
    # [G] Standard AABB split logic
    pass

def split_geometrically(partition):
    """Fallback split for empty partitions."""
    # [G] Split along longest dimension
    pass
```

**Parameters:**
*   $n$: Number of recursive iterations. Determines the number of tiles $2^n$ [S1].
*   Target Aspect Ratio: 1 (cubic) [S1].

**Edge Cases and Pitfalls:**
*   **Missing Point Cloud:** For captures lacking SfM data (e.g., Building capture), the method falls back to discretizing training rays into samples to form the PDF [S1].
*   **Imbalanced Workload:** Despite spatial partitioning, imbalanced workload distribution across GPUs remains a bottleneck, potentially addressed by better spatial partitioning algorithms [S1].

## Subtopics

### NeRF-XL Spatial Partitioning
A strategy to partition 3D space into non-overlapping bounding boxes for multi-GPU assignment, aiming to balance compute by ensuring each region contains a similar amount of spatial content [S1]. It generates a point cloud (via SfM or ray discretization), constructs a PDF, and recursively partitions space along the axis that yields partitions closest to cubic (minimizing aspect ratio $AR_i$) for $n$ iterations to achieve $2^n$ partitions [S1]. Imbalanced workload distribution across GPUs remains a bottleneck [S1].

## Operating characteristics
*   **Time Complexity:** The partitioning process involves sorting points and computing CDFs, which is $O(N \log N)$ per iteration, where $N$ is the number of points. With $n$ iterations, the total partitioning time is manageable. However, multi-GPU communication remains the primary bottleneck of the system, with communication costs reduced by over 2× through partition-based volume rendering (tile-based vs. sample-based synchronization) [S1].
*   **Memory Complexity:** Requires storing the point cloud and the partition structure. The NeRF models are distributed, so memory per GPU is reduced [S1].
*   **Strengths:** Ensures an even distribution of points across tiles, demonstrating a level-of-detail property where regions with more content or frequent capture have finer bounding boxes [S1]. Versatile for various captures (drone, street, object-centric) [S1].
*   **Limitations:** Imbalanced workload distribution across GPUs can still occur, limiting scalability [S1].

## Background
Originated in NeRF-XL, a principled algorithm to efficiently harness multi-GPU setups for large-scale NeRF performance [S1]. It revisited existing approaches of decomposing large-scale scenes into independently trained NeRFs, identifying issues that impeded effective utilization of additional computational resources [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
