# Feature Prediction Table Construction


## Overview
Feature Prediction Table Construction is a preprocessing step in lossless voxel compression that builds a lookup table mapping local voxel contexts to predicted occupancy values. It reduces redundancy by determining the most frequent target voxel state for every possible configuration of neighboring voxels, enabling efficient residual encoding [S1]. It differs from Sparse Voxel Octree (SVO) approaches by modeling prediction contexts at full grid resolution with varying strides, rather than using local contexts of SVO nodes with different resolutions at each level [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary voxel grid representing the object geometry.
*   **Output:** A set of Feature Prediction Tables (FPTs), one for each prediction level/stride. Each table maps a context index to a predicted occupancy bit.

**Data Structures:**
*   **Feature Prediction Table:** A compact array or hash map. The index is formed by concatenating the binary occupancy values of the context voxels. The value is the predicted occupancy (0 or 1) for the target voxel.
*   **Context:** A fixed arrangement of nearby voxels with known occupancy values surrounding the target voxel.

**Steps:**
1.  **Define Prediction Levels:** Select multiple levels of prediction, each characterized by a specific stride (distance between key voxels) and a corresponding context geometry [S1].
2.  **Iterate Grid:** For each prediction level, iterate through all target voxels in the grid that are not key voxels (intermediate voxels) [S1].
3.  **Extract Context:** For each target voxel, extract the occupancy values of its designated context voxels. These context voxels must have known occupancy values (either they are key voxels or have been processed in a previous step, though the source implies a statistical build phase) [S1].
4.  **Form Index:** Concatenate the binary occupancy values of the context voxels to form an integer index. If the context has $N$ voxels, there are $2^N$ possible indices [S1].
5.  **Statistical Analysis:** Accumulate statistics for each index. For every occurrence of a specific context configuration (index), record the actual occupancy value of the target voxel [S1].
6.  **Determine Prediction:** After scanning the entire grid (or a representative sample), for each index, determine the most frequent occupancy value of the corresponding target voxels across the whole grid [S1].
7.  **Store Table:** Store this most frequent value as the prediction for that index in the Feature Prediction Table [S1].

**Formulas:**
Let $C$ be the set of context voxels for a target voxel $T$. Let $v(c)$ be the occupancy value of context voxel $c \in C$.
The context index $I$ is:
$$ I = \sum_{c \in C} v(c) \cdot 2^{k_c} $$
where $k_c$ is the bit position assigned to context voxel $c$.

The predicted occupancy $\hat{v}(T)$ for index $I$ is:
$$ \hat{v}(T) = \arg\max_{b \in \{0,1\}} \left( \text{count}(I, b) \right) $$
where $\text{count}(I, b)$ is the number of times target voxel occupancy $b$ occurred for context index $I$ in the grid [S1].

**Parameters:**
*   **Context Size:** Not explicitly defined in the sources, but implied to be small enough to allow full enumeration of $2^N$ indices.
*   **Strides:** Varying strides are used for different levels. Exact stride values are "Not given in the sources."

**Edge Cases and Pitfalls:**
*   **Sparse Grids:** If a context configuration never occurs in the grid, the prediction is undefined. The source does not specify a default fallback. [G] Implementation should handle undefined indices by defaulting to 0 or a global majority.
*   **Ties:** If two occupancy values are equally frequent for an index, the source does not specify the tie-breaking rule. [G] Implementation should choose one deterministically (e.g., lower value).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_feature_prediction_table(voxel_grid, context_geometry):
    """
    Construct a Feature Prediction Table for a specific context geometry.
    
    Args:
        voxel_grid: 3D array of binary occupancy values.
        context_geometry: List of relative coordinates for context voxels.
        
    Returns:
        prediction_table: Dictionary mapping context index to predicted occupancy.
    """
    # Initialize counters for all possible context configurations
    # Assuming context has N voxels, there are 2^N possible indices
    num_context_voxels = len(context_geometry)
    num_indices = 1 << num_context_voxels
    counts = [[0, 0] for _ in range(num_indices)]  # counts[index][0] for 0, [1] for 1
    
    # Iterate over all voxels in the grid
    for x, y, z in voxel_grid.indices():
        # Extract context occupancy values
        context_bits = []
        for dx, dy, dz in context_geometry:
            # Check if context voxel is within grid bounds
            cx, cy, cz = x + dx, y + dy, z + dz
            if voxel_grid.is_valid(cx, cy, cz):
                context_bits.append(voxel_grid.get_occupancy(cx, cy, cz))
            else:
                # Handle boundary: assume 0 or skip? Source doesn't specify.
                # [G] Default to 0 for out-of-bounds context voxels
                context_bits.append(0)
        
        # Form index from context bits
        index = 0
        for i, bit in enumerate(context_bits):
            index |= (bit << i)
        
        # Get target voxel occupancy
        target_occupancy = voxel_grid.get_occupancy(x, y, z)
        
        # Update counts
        counts[index][target_occupancy] += 1
    
    # Build prediction table
    prediction_table = {}
    for index in range(num_indices):
        count_0, count_1 = counts[index]
        # Choose the most frequent occupancy
        if count_1 > count_0:
            prediction_table[index] = 1
        else:
            prediction_table[index] = 0
            
    return prediction_table
```

## Subtopics

### [Feature Prediction Table Encoding](9416-feature-prediction-table-encoding.md)
Encodes the constructed tables into the bitstream. It differs by focusing on the serialization of the LUTs rather than their construction. Choose it when you need to store the prediction rules for decoding.

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of context voxels. The statistical scan requires one pass per prediction level.
*   **Memory Complexity:** $O(2^N)$ per prediction level, where $N$ is the number of context voxels. This is small for typical small contexts.
*   **Performance:** Achieved 15.6% and 12.8% storage reduction on synthetic and real tree datasets, respectively, compared to plain PSVO. Average 11% improvement on diverse voxelized objects [S1].
*   **Strengths:** Parallelizable, suitable for GPU execution. Reduces redundancy before SVO encoding [S1].
*   **Limitations:** Requires statistical analysis of the grid. Effectiveness depends on the regularity of the voxel structure.
*   **When to use:** For voxelized models with strong local correlations, such as tree models or structured geometry.
*   **When not to use:** For highly random or noisy voxel data where local context provides little predictive power.

## Background
The method was proposed in 2025 by authors focusing on efficient encoding of voxelized tree models for machine learning applications [S1]. It is motivated by use cases in precision horticulture and forestry, where voxelized tree models are structurally complex [S1]. The method is designed to be efficient and parallelizable, particularly for GPU architectures [S1].

## Sources
[S1] Efficient Encoding and Decoding of Voxelized Models for Machine Learning-Based Applications, https://doi.org/10.1109/access.2025.3526202, fetched 2026-10-27
