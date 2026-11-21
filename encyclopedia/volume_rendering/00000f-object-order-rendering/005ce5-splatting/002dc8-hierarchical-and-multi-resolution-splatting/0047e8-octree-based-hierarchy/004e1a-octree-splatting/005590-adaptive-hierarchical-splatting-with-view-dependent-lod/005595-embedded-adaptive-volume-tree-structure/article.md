# Embedded, Adaptive Volume Tree Structure

Also called: Global earth-based hierarchy, Lat/lon quadtree with volume tree leaves

## Overview
This method organizes multiresolution splats of 3D non-uniform data into a hierarchical structure capable of handling different data types and large spatial magnitude differences between overviews and details [S1]. It is specifically designed for earth-based environments, such as atmospheric data, where the domain is a thin layer relative to the planetary ellipsoid [S1]. It sets itself apart from standard octree splatting by using a lat/lon quadtree for the global overview and switching to an embedded volume tree only when local dimensions become comparable, allowing for efficient view-dependent LOD selection as described in [Adaptive Hierarchical Splatting with View-Dependent LOD](15498-adaptive-hierarchical-splatting-with-view-dependent-lod.md).

## How it works
**Inputs:** A set of non-uniform 3D data points (e.g., atmospheric measurements) defined by longitude, latitude, and altitude coordinates, along with scalar field values.
**Outputs:** A hierarchical tree structure consisting of a top-level forest of quadtrees, mid-level volume trees, and leaf nodes containing the data points, along with aggregated metadata for each node.

**Data Structures:**
1.  **Quadtree Node:** Represents a region of the earth's surface. Contains pointers to child quadtree nodes or a volume tree root.
2.  **Volume Tree Node:** Represents a 3D bin or aggregated region.
    *   **Fields:**
        *   `total_raw_elements`: Integer count of raw volumetric data elements in children.
        *   `total_filled_bins`: Integer count of non-empty bins in children.
        *   `avg_scalar`: Float, average scalar field value of children.
        *   `var_scalar`: Float, variance of scalar field values of children.
        *   `children`: Array of pointers to child nodes.
        *   `bins`: Array of data points (only stored in leaf nodes or retained for full resolution rendering) [S1].
    *   **Dimensions:** Node dimensions ($D_x, D_y, D_z$) are not stored explicitly but calculated on-the-fly based on the tree level and bin counts [S1].

**Algorithm Steps:**

1.  **Construct Top-Level Hierarchy:**
    Create a forest of lat/lon quadtrees covering the entire domain (e.g., Earth). This structure is appropriate because the atmosphere is a thin layer relative to the Earth's ellipsoid [S1].

2.  **Determine Switch Point:**
    Traverse the quadtree. When a quadtree node's lat/lon dimensions are of the same order of magnitude as its altitude dimension, switch from the quadtree structure to a volume tree structure for that region [S1].

3.  **Define Bin Grid:**
    For each region requiring a volume tree, divide the space into $N_x \times N_y \times N_z$ bins, where $x, y, z$ correspond to longitude, latitude, and altitude directions, respectively [S1].
    *   **Constraint:** $N_x, N_y, N_z$ must be powers of 2 ($N_x = 2^m, N_y = 2^n, N_z = 2^p$) to permit straightforward binary hierarchy construction [S1].

4.  **Bin Sort Data:**
    Sort the non-uniform data points into the defined bins.
    *   **Complexity:** $O(n)$, where $n$ is the number of data points [S1].
    *   **Note:** Bins are not rectilinear in Cartesian space due to Earth's curvature. This effect is ignored during volume rendering as curvature is small for rendered bins, except at high latitudes where lat/lon quadcells are nearly rectilinear [S1]. Bin widths may be non-uniform to accommodate data density variations (e.g., higher concentration near the ground) [S1].

5.  **Construct Volume Hierarchy:**
    Build the hierarchy levels based on the exponents $m, n, p$. The number of children at a node is determined by the dimensions:
    *   If $m > n > p$:
        *   $p$ levels with 8-fold branching (each parent has 8 children).
        *   $n - p$ levels with 4-fold branching.
        *   $m - n$ levels with 2-fold branching.
    *   If two exponents are equal, there are only 8-fold and 4-fold levels [S1].
    *   The placement of these branching factors depends on the specific data distribution [S1].

6.  **Aggregate Properties:**
    Accumulate bins to form parent nodes. Derive parent properties from weighted averages of child properties [S1].
    *   `total_raw_elements` = Sum of `total_raw_elements` of children.
    *   `total_filled_bins` = Sum of `total_filled_bins` of children.
    *   `avg_scalar` and `var_scalar` are computed as weighted averages of children's scalar statistics [S1].

7.  **Store Leaf Data:**
    Store the specific data element positions in the leaf bins. This allows for full-resolution rendering if needed, while the hierarchy enables memory savings by retrieving coordinates only for viewable bins at the appropriate LOD [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_embedded_adaptive_volume_tree(data_points, domain_bounds):
    """
    data_points: List of objects with (lon, lat, alt, scalar_value)
    domain_bounds: Object with min/max lon, lat, alt
    """
    # Step 1: Construct Top-Level Lat/Lon Quadtree
    # The quadtree covers the entire earth domain.
    # We assume a standard quadtree construction algorithm for the surface.
    quadtree_root = build_lat_lon_quadtree(domain_bounds)
    
    # Step 2 & 3: Traverse and Switch to Volume Trees
    # We process each leaf of the quadtree.
    # If the node's spatial extent in lat/lon is comparable to altitude,
    # we build a volume tree inside it.
    
    for quad_node in quadtree_root.leaves():
        node_width_lon = quad_node.max_lon - quad_node.min_lon
        node_width_lat = quad_node.max_lat - quad_node.min_lat
        node_height_alt = quad_node.max_alt - quad_node.min_alt
        
        # Heuristic: Switch when lat/lon dims are same order of magnitude as altitude
        # [S1] "when the lat/lon dimensions are of the same order of magnitude as the altitude dimension"
        if is_comparable_magnitude(node_width_lon, node_height_alt) or \
           is_comparable_magnitude(node_width_lat, node_height_alt):
            
            # Define bin dimensions as powers of 2
            # [S1] Nx, Ny, Nz must be powers of 2
            Nx = 2 ** get_power_of_2_for_dimension(node_width_lon)
            Ny = 2 ** get_power_of_2_for_dimension(node_width_lat)
            Nz = 2 ** get_power_of_2_for_dimension(node_height_alt)
            
            # Step 4: Bin Sort
            # Create bins for this region
            bins = create_bins(Nx, Ny, Nz, quad_node.bounds)
            
            # Sort points into bins. O(n) complexity.
            for point in data_points:
                if point_in_bounds(point, quad_node.bounds):
                    bin_index = get_bin_index(point, bins)
                    bins[bin_index].points.append(point)
            
            # Step 5 & 6: Construct Hierarchy and Aggregate
            # Exponents for dimensions
            m = log2(Nx)
            n = log2(Ny)
            p = log2(Nz)
            
            # Build the volume tree root for this quad node
            volume_tree_root = build_volume_hierarchy(bins, m, n, p)
            
            # Attach volume tree to quad node
            quad_node.volume_tree = volume_tree_root
        else:
            # Continue quadtree subdivision if not switching yet
            # [S1] Implicitly handled by quadtree construction logic
            pass

    return quadtree_root

def build_volume_hierarchy(bins, m, n, p):
    """
    Builds the binary volume hierarchy from the sorted bins.
    """
    # Determine branching factors based on exponents
    # [S1] If m > n > p: p 8-fold, n-p 4-fold, m-n 2-fold levels
    # If two exponents equal: only 8-fold and 4-fold levels
    
    levels = []
    
    # Calculate number of levels for each branching type
    # Note: The exact order of 2, 4, 8-fold levels depends on data distribution [S1]
    # We assume a standard accumulation from leaves to root.
    
    current_level_bins = bins
    
    # We need to aggregate until we reach a single root node
    # The number of steps depends on the max exponent
    max_exp = max(m, n, p)
    
    for level_idx in range(max_exp):
        parent_bins = []
        
        # Determine branching factor for this level
        # This is a simplification; the paper states placement depends on distribution.
        # A common approach for embedded trees is to aggregate along the smallest dimension first
        # or follow the exponent order.
        
        # [S1] "The placement of 2, 4, and 8-fold levels within the hierarchy will depend on the distribution"
        # Since specific placement logic isn't fully detailed, we assume a standard accumulation
        # where we group children based on the available dimensions.
        
        # For implementation, we iterate through current bins and group them.
        # The grouping factor is determined by the remaining exponents.
        
        # Simplified aggregation logic for demonstration:
        # Group by x, then y, then z dimensions based on m, n, p.
        
        # ... Implementation of specific aggregation strategy based on m, n, p ...
        
        # Calculate parent properties
        for parent_bin in parent_bins:
            parent_bin.total_raw_elements = sum(child.total_raw_elements for child in parent_bin.children)
            parent_bin.total_filled_bins = sum(child.total_filled_bins for child in parent_bin.children)
            
            # Weighted average for scalar value
            total_weight = sum(child.total_raw_elements for child in parent_bin.children)
            if total_weight > 0:
                parent_bin.avg_scalar = sum(child.avg_scalar * child.total_raw_elements for child in parent_bin.children) / total_weight
                # Variance calculation requires more complex formula (total sum of squares)
                # [S1] "average scalar field value and its variance"
                parent_bin.var_scalar = calculate_variance(parent_bin.children)
            
            parent_bin.children = parent_bin.children # Reference to children
            
        current_level_bins = parent_bins
        
        if len(current_level_bins) == 1:
            return current_level_bins[0]
            
    return current_level_bins[0]

def is_comparable_magnitude(a, b):
    # Helper to check if two values are of the same order of magnitude
    # [S1] "same order of magnitude"
    return abs(log10(a) - log10(b)) <= 1.0

def get_power_of_2_for_dimension(width):
    # Return integer m such that 2^m approximates the required resolution
    # [S1] Nx = 2^m
    return int(log2(width)) # Simplified

def calculate_variance(children):
    # Placeholder for variance calculation
    # [S1] Variance is stored
    return 0.0 # Not given in the sources.
```

**Parameters:**
*   $N_x, N_y, N_z$: Bin dimensions in longitude, latitude, and altitude directions. Must be powers of 2 [S1].

**Edge Cases and Pitfalls:**
*   **Curvature:** Bins are not rectilinear in Cartesian space. The method ignores this effect during volume rendering because Earth's curvature is small for rendered bins, and lat/lon quadcells are nearly rectilinear except at high latitudes [S1].
*   **Dynamic Data:** The bin structure and volume hierarchy are static in space. The method can be applied to moving volumetric elements only if the range of local spatial densities and the volume of the data do not change much over time [S1].

## Operating characteristics
*   **Time Complexity:** The bin sort of non-uniform data points is $O(n)$, where $n$ is the number of points [S1].
*   **Memory Complexity:** The hierarchy provides significant savings in memory space and retrieval cost since only data element coordinates for viewable bins at the appropriate LOD are retrieved [S1].
*   **Strengths:** Efficiently supports different data types and spatial magnitudes. The bin sort is fast, and the hierarchy enables quick aggregation for detail management and view frustum culling [S1].
*   **Limitations:** The structure is static in space, limiting applicability to highly dynamic data distributions. The approximation of ignoring Earth's curvature may introduce errors at high latitudes or for very high-resolution rendering of small bins [S1].

## Background
This method was presented by Jang, Ribarsky, Shaw, and Faust in 2002 as part of their work on View-Dependent Multiresolution Splatting [S1]. It particularizes to earth-based data, specifically atmospheric data and whole earth environments, where the atmosphere is a thin layer relative to the Earth's ellipsoid [S1]. The approach builds on the need for automatic generation of volume tree structures in environments with different data organizations and spatial magnitudes [S1].

## Sources
[S1] View-Dependent Multiresolution Splatting of Non-Uniform Data, https://doi.org/10.2312/vissym/vissym02/125-132, fetched 2026-10-30
