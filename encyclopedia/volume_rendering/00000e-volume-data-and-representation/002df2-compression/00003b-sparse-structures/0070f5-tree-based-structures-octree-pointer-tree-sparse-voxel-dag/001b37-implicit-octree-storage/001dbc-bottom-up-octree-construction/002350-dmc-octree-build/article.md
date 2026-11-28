# DMC Octree Build
Also called: DMC Octree Construction
## Overview
The DMC (Dual Marching Cubes) Octree Build constructs an adaptive-resolution octree from a 3D scalar field by solving a Quadratic Error Function (QEF) to determine optimal node positions and applying various split policies to balance detail and compression [S1, unofficial]. It differs from standard bottom-up approaches by integrating geometric error minimization directly into the node consolidation phase, rather than relying solely on scalar homogeneity thresholds.
## How it works
### Inputs and Outputs
**Inputs:**
- `field`: 3D scalar array representing the volume data.
- `solver`: QEF solver instance used for geometric optimization.
- `epsilon`: Base error threshold for node splitting/merging.
- `globalScale`: Scaling factor for the coordinate system.
- `root_size`: The side length of the root octree node in voxels.
- `split_policies`: Configuration for how nodes are subdivided.
- `rng`: Seeded random number generator (mt19937) for stochastic decisions.

**Outputs:**
- An octree structure where internal nodes represent homogeneous regions and leaf nodes contain optimized geometry or scalar data.

### Data Structures
The implementation relies on a `BuildArgs` structure to pass context to recursive builders:
- `field`: Reference to the input scalar volume.
- `solver`: Reference to the QEF solver.
- `scaled_epsilon`: Pre-computed error threshold.
- `split_policies`: Object defining subdivision criteria.
- `rng`: Random number generator state.

### Steps
1. **Validate Root Size**: Ensure `root_size` is greater than 0 and a power of two. If `root_size & (root_size - 1) != 0`, throw `std::invalid_argument` [S1, unofficial].
2. **Compute Scaled Epsilon**: Calculate the error threshold adjusted for the global scale:
   $$ \text{scaled\_epsilon} = \frac{\epsilon}{\text{globalScale} \times \text{globalScale}} $$
   This ensures the error metric is consistent with the QEF solver's coordinate space [S1, unofficial].
3. **Initialize Build Context**: Create a `BuildArgs` object containing the field, solver, `scaled_epsilon`, split policies, and a seeded `mt19937` RNG [S1, unofficial].
4. **Collapse Root**: Perform an initial collapse operation on the root node to merge trivially homogeneous regions before detailed construction [S1, unofficial].
5. **Build Root Node**: Call the recursive `buildNode` function on the root with:
   - `min_corner`: $(-\frac{\text{root\_size}}{2}, -\frac{\text{root\_size}}{2}, -\frac{\text{root\_size}}{2})$
   - `size`: `root_size`
   - `args`: The initialized `BuildArgs` context [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_dmc_octree(field, solver, epsilon, global_scale, root_size, split_policies):
    # Step 1: Validate root_size
    if root_size <= 0 or (root_size & (root_size - 1)) != 0:
        raise std.invalid_argument("root_size must be > 0 and a power of two")

    # Step 2: Compute scaled_epsilon
    scaled_epsilon = epsilon / (global_scale * global_scale)

    # Step 3: Initialize BuildArgs
    # mt19937 is seeded with a fixed or provided seed for reproducibility
    rng = mt19937(seed=42) 
    build_args = BuildArgs(
        field=field,
        solver=solver,
        scaled_epsilon=scaled_epsilon,
        split_policies=split_policies,
        rng=rng
    )

    # Step 4: Collapse root (implementation detail of the specific library)
    # This merges nodes that are already homogeneous at the root level
    collapse_root(field, root_size)

    # Step 5: Build node recursively
    min_corner = (-root_size / 2.0, -root_size / 2.0, -root_size / 2.0)
    root_node = build_node(min_corner, root_size, build_args)
    
    return root_node

def build_node(min_corner, size, args):
    # Determine if the current voxel block is homogeneous or meets error criteria
    # This involves querying the field and solving QEF if necessary
    # If homogeneous or error < scaled_epsilon:
    #   Return a leaf node with optimized data
    # Else:
    #   Split into 8 children
    #   For each child i in 0..7:
    #       child_size = size / 2
    #       child_min_corner = min_corner + offset(i, child_size)
    #       child_node = build_node(child_min_corner, child_size, args)
    #   Return an internal node with children
    pass
```

### Edge Cases and Pitfalls
- **Non-power-of-two root size**: The algorithm strictly requires `root_size` to be a power of two. Passing other values results in an immediate error [S1, unofficial].
- **Scale Mismatch**: If `globalScale` is not applied correctly to `epsilon`, the QEF solver may produce inaccurate geometry or fail to converge, as the error threshold will be misaligned with the coordinate space [S1, unofficial].
- **Precision**: Floating-point precision in the QEF solver can affect the stability of node splitting, particularly for very small `epsilon` values.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Integrates geometric optimization (QEF) directly into the tree construction, potentially yielding higher quality meshes at lower resolutions compared to scalar-only methods.
- **Limitations**: Requires a QEF solver, adding computational overhead. Strict root size constraints limit flexibility in input dimensions.
- **When to use**: When high-fidelity geometry extraction is required alongside compression, and the input volume dimensions are powers of two.
- **When not to use**: For non-power-of-two volumes without preprocessing, or when scalar homogeneity is sufficient and geometric optimization is too expensive.

## Background
The DMC Octree Build is part of the `isomesh` library by Svenny [S1, unofficial]. It extends the concept of Dual Marching Cubes to an octree structure, allowing for adaptive resolution based on geometric error rather than just scalar variance. The method was documented in the source code of the GitHub repository `Svenny/isomesh` [S1, unofficial].

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
