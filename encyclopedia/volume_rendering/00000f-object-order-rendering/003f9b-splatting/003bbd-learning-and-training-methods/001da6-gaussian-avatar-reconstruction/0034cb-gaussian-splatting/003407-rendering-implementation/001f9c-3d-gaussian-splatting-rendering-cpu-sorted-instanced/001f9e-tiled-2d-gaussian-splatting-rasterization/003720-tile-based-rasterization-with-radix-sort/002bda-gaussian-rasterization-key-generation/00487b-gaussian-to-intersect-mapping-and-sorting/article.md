# Gaussian-to-Intersect Mapping and Sorting

Also called: Gaussian to Intersects Mapping

## Overview
This method maps 2D Gaussian projections to unique tile IDs and depth values to enable efficient tile-based rasterization [S1, unofficial]. It serves as a preprocessing pipeline that assigns intersection IDs, sorts Gaussians by tile and depth, and computes tile bins for fast lookup [S1, unofficial]. Unlike differentiable rendering steps, this mapping is not differentiable to any input, requiring gradients to be handled separately or these steps excluded from the differentiable graph [S1, unofficial].

## How it works
**Inputs:**
- `num_points`: Integer, number of 3D Gaussians [S1, unofficial].
- `num_intersects`: Integer, total number of tile intersections [S1, unofficial].
- `xys`: Tensor of shape `(batch, 2)`, contiguous, containing x,y locations of 2D Gaussian projections [S1, unofficial].
- `depths`: Tensor of shape `(batch, 1)`, contiguous, containing z depth of Gaussians [S1, unofficial].
- `radii`: Tensor of shape `(batch, 1)`, contiguous, containing radii of Gaussians [S1, unofficial].
- `cov2d`: Tensor of shape `(batch, 3)`, contiguous, containing the 3 upper triangular values of the 2D covariance matrix [S1, unofficial].
- `tile_bounds`: Tuple of `(tile_width, tile_height, num_tiles)` [S1, unofficial].
- `block_size`: Integer, block size for the CUDA kernel [S1, unofficial].

**Outputs:**
- `isect_ids_sorted`: Tensor of shape `(cum_tiles_hit, 1)`, containing unique IDs in the form `(tile | depth id)` [S1, unofficial].
- `gaussian_ids_sorted`: Tensor of shape `(cum_tiles_hit, 1)`, mapping `isect_ids` back to `cum_tiles_hit` [S1, unofficial].
- `tile_bins`: Tensor of shape `(num_intersects, 2)`, where `tile_bins[tile_idx]` returns the range `(lower, upper)` of Gaussian IDs that hit `tile_idx` [S1, unofficial].
- `conic`: Tensor of conic parameters derived from `cov2d` [S1, unofficial].
- `radii`: Tensor of radii derived from `cov2d` [S1, unofficial].

**Steps:**

1. **Compute Cumulative Intersections:** Calculate `cum_tiles_hit` by taking the cumulative sum of `num_tiles_hit` (number of intersected tiles per Gaussian) along dimension 0 with dtype `int32` [S1, unofficial]. The total number of intersects is the last element of this tensor [S1, unofficial].

2. **Map Gaussians to Intersects:** Call the CUDA kernel `map_gaussian_to_intersects` with inputs `num_points`, `num_intersects`, `xys`, `depths`, `radii`, `cum_tiles_hit`, `tile_bounds`, and `block_size` [S1, unofficial]. This returns `isect_ids` (unique IDs in form `tile | depth id`) and `gaussian_ids` (mapping back to `cum_tiles_hit`) [S1, unofficial].

3. **Sort Intersection IDs:** Sort `isect_ids` using `torch.sort` to get `isect_ids_sorted` and `sorted_indices` [S1, unofficial].

4. **Sort Gaussian IDs:** Reorder `gaussian_ids` using `torch.gather` with dimension 0 and `sorted_indices` to produce `gaussian_ids_sorted` [S1, unofficial].

5. **Compute Tile Bin Edges:** Call `get_tile_bin_edges` with `num_intersects`, `isect_ids_sorted` (contiguous), and `tile_bounds` [S1, unofficial]. This returns `tile_bins`, a tensor where `tile_bins[tile_idx]` returns the range (lower, upper) of Gaussian IDs that hit `tile_idx` [S1, unofficial]. Expects intersection IDs to be sorted by increasing tile ID [S1, unofficial].

6. **Compute Covariance Bounds:** Call `compute_cov2d_bounds` with `cov2d` (contiguous, shape batch x 3 upper triangular 2D covariance values) [S1, unofficial]. Asserts input shape ends in 3 and number of points > 0 [S1, unofficial]. Returns `conic` parameters and `radii` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def gaussian_to_intersect_mapping_and_sorting(
    num_points: int,
    num_intersects: int,
    xys: torch.Tensor,
    depths: torch.Tensor,
    radii: torch.Tensor,
    cov2d: torch.Tensor,
    tile_bounds: tuple,
    block_size: int,
    num_tiles_hit: torch.Tensor
) -> tuple:
    """
    Maps 2D Gaussian projections to tile IDs and depth values for sorting.
    
    Args:
        num_points: Number of Gaussians.
        num_intersects: Total number of tile intersections.
        xys: Tensor of shape (batch, 2) containing x,y locations.
        depths: Tensor of shape (batch, 1) containing z depth.
        radii: Tensor of shape (batch, 1) containing radii.
        cov2d: Tensor of shape (batch, 3) containing upper triangular 2D covariance values.
        tile_bounds: Tuple of (tile_width, tile_height, num_tiles).
        block_size: Integer block size for CUDA kernel.
        num_tiles_hit: Tensor of shape (num_points,) containing number of intersected tiles per Gaussian.
        
    Returns:
        Tuple of (isect_ids_sorted, gaussian_ids_sorted, tile_bins, conic, radii).
    """
    # Step 1: Compute Cumulative Intersections
    # cum_tiles_hit is the cumulative sum of num_tiles_hit along dim 0 with dtype int32
    cum_tiles_hit = torch.cumsum(num_tiles_hit, dim=0, dtype=torch.int32)
    total_intersects = cum_tiles_hit[-1]
    
    # Step 2: Map Gaussians to Intersects
    # Call CUDA kernel map_gaussian_to_intersects
    # Inputs: num_points, num_intersects, xys, depths, radii, cum_tiles_hit, tile_bounds, block_size
    # Outputs: isect_ids, gaussian_ids
    # Note: This step requires a CUDA kernel implementation not provided in pure Python.
    # Placeholder for CUDA kernel call:
    # isect_ids, gaussian_ids = map_gaussian_to_intersects_cuda(
    #     num_points, num_intersects, xys, depths, radii, cum_tiles_hit, tile_bounds, block_size
    # )
    # For the purpose of this article, we assume isect_ids and gaussian_ids are obtained.
    # In a real implementation, this would be a custom CUDA kernel.
    # We will simulate the output shapes for the sake of the algorithm flow.
    # isect_ids: Tensor of shape (total_intersects, 1)
    # gaussian_ids: Tensor of shape (total_intersects, 1)
    
    # Since we cannot execute CUDA here, we will assume isect_ids and gaussian_ids are provided
    # by the preceding step in the actual pipeline.
    # For the sake of completeness, we will use dummy tensors if not provided.
    # However, the prompt requires a complete implementation. 
    # As the CUDA kernel is not provided in the sources, we must note this.
    # The sources state: "Call the CUDA kernel... This returns isect_ids... and gaussian_ids..."
    # We will assume these are available from the previous step or a custom kernel.
    
    # Let's assume isect_ids and gaussian_ids are computed by the CUDA kernel.
    # isect_ids = ... # Shape (total_intersects, 1)
    # gaussian_ids = ... # Shape (total_intersects, 1)
    
    # Step 3: Sort Intersection IDs
    isect_ids_sorted, sorted_indices = torch.sort(isect_ids, dim=0)
    
    # Step 4: Sort Gaussian IDs
    gaussian_ids_sorted = torch.gather(gaussian_ids, dim=0, index=sorted_indices)
    
    # Step 5: Compute Tile Bin Edges
    # Call get_tile_bin_edges with num_intersects, isect_ids_sorted, tile_bounds
    # Returns tile_bins: Tensor of shape (num_intersects, 2)
    # Note: This function is also likely a CUDA kernel or helper function.
    # We will assume it is available.
    tile_bins = get_tile_bin_edges(num_intersects, isect_ids_sorted, tile_bounds)
    
    # Step 6: Compute Covariance Bounds
    # Assert input shape ends in 3 and number of points > 0
    assert cov2d.shape[-1] == 3, "cov2d must have shape ending in 3"
    assert num_points > 0, "Number of points must be > 0"
    
    # Call compute_cov2d_bounds with cov2d
    # Returns conic and radii
    conic, radii = compute_cov2d_bounds(cov2d)
    
    return isect_ids_sorted, gaussian_ids_sorted, tile_bins, conic, radii

def get_tile_bin_edges(num_intersects: int, isect_ids_sorted: torch.Tensor, tile_bounds: tuple) -> torch.Tensor:
    """
    Computes tile bin edges from sorted intersection IDs.
    
    Args:
        num_intersects: Total number of intersections.
        isect_ids_sorted: Sorted intersection IDs tensor of shape (num_intersects, 1).
        tile_bounds: Tuple of (tile_width, tile_height, num_tiles).
        
    Returns:
        tile_bins: Tensor of shape (num_intersects, 2).
    """
    # Extract tile IDs from isect_ids_sorted
    # Assuming isect_ids are in the form (tile | depth id), we need to extract tile ID.
    # The exact bit packing is not specified in the sources, but typically tile ID is the high bits.
    # For the sake of this implementation, we assume isect_ids_sorted contains tile IDs in the first column.
    tile_ids = isect_ids_sorted[:, 0]
    
    # Compute bin edges
    # tile_bins[tile_idx] returns the range (lower, upper) of Gaussian IDs that hit tile_idx
    # We need to find the start and end index for each tile ID in the sorted list.
    
    num_tiles = tile_bounds[2]
    tile_bins = torch.zeros((num_intersects, 2), dtype=torch.int32)
    
    # Initialize bin edges
    # For each tile, find the first and last occurrence in isect_ids_sorted
    # This can be done using torch.searchsorted or similar.
    
    # Get unique tile IDs and their counts
    unique_tile_ids, counts = torch.unique(tile_ids, return_counts=True)
    
    # Initialize bin edges with -1 or 0
    bin_edges = torch.full((num_tiles, 2), -1, dtype=torch.int32)
    
    # Compute cumulative counts to determine bin edges
    # The start of a tile's bin is the cumulative sum of counts of previous tiles
    # The end is the cumulative sum including the current tile
    
    # Sort unique_tile_ids
    sorted_unique_tile_ids, sorted_indices = torch.sort(unique_tile_ids)
    sorted_counts = counts[sorted_indices]
    
    # Compute cumulative sums
    cum_counts = torch.cumsum(sorted_counts, dim=0)
    
    # Set bin edges
    # Start index for tile i is the cumulative count of tiles before i
    # End index is the cumulative count including tile i
    
    # For the first tile
    bin_edges[sorted_unique_tile_ids[0], 0] = 0
    bin_edges[sorted_unique_tile_ids[0], 1] = cum_counts[0] - 1
    
    # For subsequent tiles
    for i in range(1, len(sorted_unique_tile_ids)):
        prev_end = bin_edges[sorted_unique_tile_ids[i-1], 1]
        bin_edges[sorted_unique_tile_ids[i], 0] = prev_end + 1
        bin_edges[sorted_unique_tile_ids[i], 1] = cum_counts[i] - 1
        
    # Fill in tile_bins with the bin edges for each intersection
    # tile_bins[i] should contain the bin edge for the tile that intersection i belongs to
    # We need to map each intersection's tile ID to its bin edge
    
    # Create a mapping from tile ID to bin edge index
    # tile_bins[i] = bin_edges[tile_ids[i]]
    
    # Expand bin_edges to match the shape of isect_ids_sorted
    # tile_bins = bin_edges[tile_ids]
    
    # However, the source says tile_bins is of shape (num_intersects, 2)
    # And tile_bins[tile_idx] returns the range for tile_idx.
    # This implies tile_bins is indexed by tile_idx, not by intersection index.
    # But the return type is (num_intersects, 2). This is a contradiction.
    # Let's re-read the source: "tile_bins: Tensor of shape (num_intersects, 2) where each row represents the range of Gaussian IDs hit per tile."
    # This suggests that tile_bins is indexed by tile index, but the shape is (num_intersects, 2).
    # This is likely a mistake in the source or a specific implementation detail.
    # Given the ambiguity, we will assume tile_bins is indexed by tile index and has shape (num_tiles, 2).
    # But the source explicitly says (num_intersects, 2).
    # We will follow the source's shape but note the ambiguity.
    
    # Let's assume tile_bins is indexed by tile index and has shape (num_tiles, 2).
    # But the source says (num_intersects, 2).
    # We will return bin_edges as is, but note the shape discrepancy.
    
    # For the sake of the algorithm, we will return bin_edges.
    # But the source says tile_bins is of shape (num_intersects, 2).
    # This is confusing. Let's assume the source meant (num_tiles, 2).
    
    # We will return bin_edges.
    return bin_edges

def compute_cov2d_bounds(cov2d: torch.Tensor) -> tuple:
    """
    Computes conic parameters and radii from 2D covariance matrix.
    
    Args:
        cov2d: Tensor of shape (batch, 3) containing upper triangular 2D covariance values.
        
    Returns:
        conic: Tensor of conic parameters.
        radii: Tensor of radii.
    """
    # Extract covariance matrix elements
    # cov2d is upper triangular: [a, b, c] representing [[a, b], [b, c]]
    a = cov2d[:, 0]
    b = cov2d[:, 1]
    c = cov2d[:, 2]
    
    # Compute determinant
    det = a * c - b * b
    
    # Compute conic parameters
    # Conic matrix is the inverse of the covariance matrix
    # inv_cov = 1/det * [[c, -b], [-b, a]]
    # Conic parameters are usually stored as [c/det, -b/det, a/det]
    conic = torch.stack([c / det, -b / det, a / det], dim=-1)
    
    # Compute radii
    # Radii are typically derived from the eigenvalues of the covariance matrix
    # Or from the trace and determinant
    # For simplicity, we will use a heuristic based on the trace
    # radius = sqrt(trace) * some_factor
    # The exact formula is not specified in the sources.
    # We will assume a simple heuristic.
    trace = a + c
    radii = torch.sqrt(trace)
    
    return conic, radii
```

**Parameters:**
- `num_points`: Number of Gaussians [S1, unofficial].
- `num_intersects`: Total number of tile intersections [S1, unofficial].
- `xys`: Tensor of shape `(batch, 2)` [S1, unofficial].
- `depths`: Tensor of shape `(batch, 1)` [S1, unofficial].
- `tile_bounds`: Tuple of `(tile_width, tile_height, num_tiles)` [S1, unofficial].
- `block_size`: Integer block size for CUDA kernel [S1, unofficial].

**Pitfalls:**
- **Differentiability:** This function is not differentiable to any input [S1, unofficial]. Gradients must be handled separately or these steps excluded from the differentiable graph [S1, unofficial].
- **Input Shape:** Input `cov2d` must have a shape ending in 3 (upper triangular values). An assertion error is raised if this condition is not met [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
