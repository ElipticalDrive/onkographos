# Anisotropic Grid Resolution for Ray Traversal

Also called: Anisotropic Grid Resolution for Radiation Walk, Anisotropic Voxel Grid Traversal, Directional Grid Optimization

## Overview
Anisotropic Grid Resolution is a technique to optimize the cost of grid-based ray traversal by aligning voxel dimensions with the primary direction of ray travel, reducing the number of voxel steps required to traverse empty space [S1, unofficial]. Unlike uniform grids where every voxel is the same size, anisotropic grids allow empty space to be crossed in fewer steps along the travel axis, directly reducing the walk cost which is proportional to distance travelled divided by voxel size [S1, unofficial]. It sets itself apart from siblings like [Binary Grid Ray Traversal Initialization](12500-binary-grid-ray-traversal-initialization.md) by modifying the grid's geometric definition rather than the traversal algorithm's step logic.

## How it works
**Inputs and Outputs:**
- Input: The bounding volume of the scene, the distribution of ray origins and directions (or a representative sample), and the target total number of voxels (or memory budget).
- Output: A grid definition with anisotropic voxel dimensions $(dx, dy, dz)$.

**Data Structures:**
The grid is defined by its dimensions $(N_x, N_y, N_z)$ and physical extents $(L_x, L_y, L_z)$. The voxel sizes are:
$$ dx = \frac{L_x}{N_x}, \quad dy = \frac{L_y}{N_y}, \quad dz = \frac{L_z}{N_z} $$
Unlike a uniform grid where $dx \approx dy \approx dz$, here the dimensions are optimized based on ray statistics.

**Steps:**
1. **Analyze Ray Distribution:** Determine the dominant axis of travel for the ray population. For example, in a chamber with pipe cells, rays may travel primarily along the axial direction [S1, unofficial].
2. **Determine Dominant Axis:** Identify the axis along which rays travel the longest distance or have the highest density. Let this be the "axial" direction (e.g., $x$) and the others be "perpendicular" (e.g., $y, z$) [S1, unofficial].
3. **Size Grid Anisotropically:**
   - Use coarser voxels along the dominant travel axis. This reduces the number of steps required to traverse the length of the scene for rays aligned with that axis.
   - Use finer voxels along perpendicular axes to maintain sufficient spatial resolution for geometry intersection or density sampling in those directions [S1, unofficial].
4. **Calculate Voxel Dimensions:** Set $dx$ to be larger than $dy$ and $dz$ if $x$ is the dominant axis. The total number of voxels $N_{total} = N_x N_y N_z$ should remain within the memory budget.

**Formulas:**
The cost of traversal is proportional to the number of steps. For a ray traveling distance $D$ along axis $i$, the number of steps is $N_{steps} \approx D / v_i$, where $v_i$ is the voxel size along that axis. By increasing $v_{dominant}$, $N_{steps}$ decreases [S1, unofficial].

**Parameters:**
- Target voxel count: Determined by memory constraints.
- Anisotropy ratio: The ratio of coarse to fine voxel sizes. In the cited example, a 128³ grid over a long box resulted in 13.6 mm along $x$ and much finer values across, compared to a default near-cubic grid of 8.2 mm [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Near-Cubic Default:** The "near-cubic" rule is a shipped default that may be wrong for long thin vessels (like reactors) [S1, unofficial].
- **Scene Geometry:** Assuming anisotropy is always better is risky; cube-ish scenes may prefer the current uniform rule [S1, unofficial].
- **Ray Distribution:** If rays are isotropic (random directions), anisotropic resolution may not provide significant benefits and could even degrade performance for non-dominant directions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_anisotropic_grid(scene_bounds, ray_sample, target_voxel_count):
    """
    Compute anisotropic grid dimensions based on ray distribution.
    
    Args:
        scene_bounds: Tuple (Lx, Ly, Lz) representing the physical extents of the scene.
        ray_sample: List of ray directions or a statistical summary of ray travel distances per axis.
        target_voxel_count: Total number of voxels in the grid.
        
    Returns:
        Tuple (Nx, Ny, Nz) for the grid dimensions.
    """
    Lx, Ly, Lz = scene_bounds
    
    # Step 1: Analyze ray distribution to find dominant axis
    # Calculate average travel distance or ray density per axis
    # For simplicity, assume we have pre-computed weights wx, wy, wz
    # representing the importance or length of travel along each axis.
    # In the example, pipe cells have strong axial (x) travel.
    
    # Example heuristic: weight by inverse of expected step count
    # If rays travel mostly along X, we want fewer steps along X, so larger dx.
    # This implies Nx should be smaller relative to Ly/Lz if we keep total voxels constant.
    
    # Let's assume we have a metric 'travel_weight' for each axis
    # travel_weight[i] = sum of ray lengths projected on axis i
    travel_weights = compute_travel_weights(ray_sample) # [G] Helper to aggregate ray stats
    
    # Step 2: Determine dominant axis
    # The axis with the highest travel weight should have the coarsest voxels (fewest divisions)
    # However, to minimize total steps, we want voxel size proportional to travel distance?
    # Actually, the goal is to reduce steps. Steps = Distance / VoxelSize.
    # To reduce steps for long distances, we need LARGE voxel size.
    # So, VoxelSize[i] should be proportional to TravelDistance[i]?
    # But we are constrained by total voxel count.
    
    # Let's use the principle: VoxelSize[i] ~ TravelDistance[i]^(1/3) ? 
    # Or simpler: Allocate more voxels to axes where geometry is complex or rays are dense?
    # The source says: "coarser voxels along the dominant travel axis".
    # This means if X is dominant, Nx is SMALLER.
    
    # Heuristic: Inverse proportionality between voxel count and travel weight?
    # If travel weight is high, we want fewer voxels (coarser grid) to skip empty space faster.
    
    # Normalize weights
    total_weight = sum(travel_weights)
    weights = [w / total_weight for w in travel_weights]
    
    # Allocate voxel counts inversely proportional to travel weight?
    # If X has high weight, we want small Nx.
    # Let's try: Nx ~ 1 / weight_x
    # But we need to satisfy Nx * Ny * Nz = target_voxel_count
    
    # Let's use a power law allocation
    # Nx = target_voxel_count^(1/3) * (1/weight_x)^alpha
    # This is a guess. The source doesn't give a specific formula, just the concept.
    
    # Alternative: The source mentions a 128^3 grid vs 212x11x114.
    # The 212x11x114 grid is anisotropic.
    # It seems the grid dimensions are chosen to match the aspect ratio of the scene or ray paths.
    
    # Let's assume a simple approach:
    # 1. Identify dominant axis (max travel weight).
    # 2. Set voxel size along dominant axis to be larger.
    # 3. Keep voxel sizes along other axes smaller to maintain resolution.
    
    # Since no exact formula is given, we use a heuristic based on aspect ratio of travel.
    
    # Let's assume the scene bounds are fixed.
    # We want to minimize sum(D_i / v_i) subject to product(v_i) = constant?
    # No, the cost is sum of steps. Steps_i = D_i / v_i.
    # Total steps = D_x/v_x + D_y/v_y + D_z/v_z.
    # We want to minimize this sum subject to (L_x/v_x) * (L_y/v_y) * (L_z/v_z) = N_total.
    # Let v_x = L_x/N_x, etc.
    # Minimize D_x * N_x / L_x + D_y * N_y / L_y + D_z * N_z / L_z
    # Subject to N_x * N_y * N_z = N_total.
    
    # Using Lagrange multipliers, the optimal N_i is proportional to sqrt(D_i / L_i).
    # But D_i is the ray length, L_i is the scene extent.
    # If rays span the whole scene, D_i ~ L_i. Then N_i ~ sqrt(1) = constant?
    # This suggests uniform grid is optimal if rays are isotropic and span the scene.
    
    # However, if rays are directional, e.g., D_x >> D_y, D_z.
    # Then N_x should be larger? No, wait.
    # Cost = D_x/v_x. If D_x is large, we want v_x to be large to reduce cost.
    # Large v_x means small N_x.
    # So N_x should be SMALL if D_x is large.
    
    # Let's re-evaluate the Lagrange result.
    # Minimize sum(C_i * N_i) where C_i = D_i / L_i.
    # Subject to product(N_i) = K.
    # N_i = K^(1/3) * (1/C_i)^(1/3) ?
    # No, for sum(C_i N_i) with product(N_i)=K, the minimum is when C_i N_i are equal?
    # No, that's for sum(N_i) with product fixed?
    # Let's check: N_x = N_y = N_z -> Cost = sum(C_i) N.
    # If we vary N_x, N_y, N_z.
    # The optimal solution for minimizing sum(a_i x_i) subject to prod(x_i)=C is x_i proportional to 1/a_i?
    # No, that's for sum(x_i) with prod fixed?
    # Actually, for sum(a_i x_i) with prod(x_i)=C, the minimum is when a_i x_i are all equal?
    # Let a_x x_x = a_y x_y = a_z x_z = k.
    # Then x_x = k/a_x, x_y = k/a_y, x_z = k/a_z.
    # prod(x_i) = k^3 / (a_x a_y a_z) = C.
    # k = (C a_x a_y a_z)^(1/3).
    # So x_i = (C a_x a_y a_z)^(1/3) / a_i = C^(1/3) (a_y a_z / a_x)^(1/3).
    # So N_i is proportional to (a_j a_k / a_i)^(1/3).
    # Since a_i = D_i / L_i, N_i is proportional to ( (D_j/L_j)(D_k/L_k) / (D_i/L_i) )^(1/3).
    
    # If D_i ~ L_i (rays span the scene), then a_i ~ 1.
    # Then N_i ~ (1*1/1)^(1/3) = 1. So N_x = N_y = N_z. Uniform grid.
    
    # But if rays are directional, e.g., D_x is large, D_y, D_z small.
    # Then a_x is large, a_y, a_z small.
    # N_x ~ (small * small / large)^(1/3) = small.
    # N_y ~ (large * small / small)^(1/3) = large.
    # N_z ~ (large * small / small)^(1/3) = large.
    # So N_x is small, N_y and N_z are large.
    # This means coarse voxels along X, fine voxels along Y and Z.
    # This matches the source: "coarser voxels along the dominant travel axis".
    
    # So the formula is:
    # N_i = round( N_total^(1/3) * ( (a_j * a_k) / a_i )^(1/3) )
    # where a_i = D_i / L_i.
    
    # We need to estimate D_i. If we don't have ray data, we might use L_i.
    # But the source implies we analyze ray distribution.
    
    # Let's implement this heuristic.
    
    a = [travel_weights[i] / scene_bounds[i] for i in range(3)]
    
    # Calculate N_i
    N = [0, 0, 0]
    for i in range(3):
        j = (i + 1) % 3
        k = (i + 2) % 3
        # N_i proportional to (a_j * a_k / a_i)^(1/3)
        factor = (a[j] * a[k] / a[i]) ** (1/3)
        N[i] = factor
        
    # Normalize to target_voxel_count
    total_factor = sum([n**3 for n in N]) # Wait, N_i are not normalized yet.
    # We need product(N_i) = target_voxel_count.
    # Let N_i = k * factor_i.
    # prod(N_i) = k^3 * prod(factor_i) = target_voxel_count.
    # k = (target_voxel_count / prod(factor_i))^(1/3).
    
    prod_factors = N[0] * N[1] * N[2]
    k = (target_voxel_count / prod_factors) ** (1/3)
    
    N = [round(k * n) for n in N]
    
    # Ensure minimum 1 voxel per axis
    N = [max(1, n) for n in N]
    
    return tuple(N)
```

## Operating characteristics
- **Time Complexity:** The grid setup is $O(1)$ after ray analysis. The traversal cost is reduced by a factor proportional to the improvement in voxel size along the dominant axis. In the cited example, a coarser axial voxel provided a 3.26x speedup for pipe-cell rays [S1, unofficial].
- **Memory Complexity:** Depends on the total number of voxels. Anisotropic grids use the same memory budget as uniform grids but distribute it differently.
- **Measured Performance:** In a test with 300,000 rays per corner, switching from a default area-sized grid (212, 11, 114) to a 128³ grid for pipe cells resulted in a performance drop? No, the text states: 'a coarser axial voxel is worth 3.26x to them'. Specifically, the default grid (near cubic at 8.2 mm) vs 128³ grid (13.6 mm along x) showed that pipe-cell rays (axial) benefited from the coarser axial voxel size [S1, unofficial].
- **Strengths:** Significant speedup for directional ray distributions (e.g., pipe flows, reactors). Simple to implement by adjusting grid dimensions.
- **Limitations:** May degrade performance for isotropic or cube-ish scenes. Requires analysis of ray distribution.
- **When to Use:** When rays have a strong dominant direction and the scene is elongated along that direction.
- **When Not to Use:** When rays are isotropic or the scene is cube-like.

## Background
This technique was discussed in the context of optimizing ray traversal for radiation transport simulations in the `aquaflux` project [S1, unofficial]. It addresses the observation that the grid walk's cost is dominated by the distance travelled, not the number of triangles tested, and that skipping empty space is key [S1, unofficial]. The issue highlights that sizing the grid anisotropically to the ray distribution is a simpler and potentially bigger win than optimizing the walk direction itself [S1, unofficial].

## Sources
[S1] Radiation: the grid walk's cost is the distance travelled, not the triangles tested — start from the near end and skip empty space, https://github.com/DeGrootResearchGroup/aquaflux/issues/503, fetched 2026-10-25 (unofficial)
