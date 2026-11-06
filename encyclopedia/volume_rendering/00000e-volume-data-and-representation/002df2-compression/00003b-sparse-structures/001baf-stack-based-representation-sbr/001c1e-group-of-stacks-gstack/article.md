# Group of Stacks (gstack)
Also called: Gstack

## Overview
A gstack is a compact data structure that represents a rectangular region of vertical stacks in a voxel grid, where all stacks share an identical sequence of attribute values and the same number of intervals [S1, unofficial]. It solves the problem of redundant storage in sparse volume representations by compressing the common attribute information while storing the varying geometry as a set of non-intersecting heightfields [S1, unofficial]. Unlike standard spatial decompositions that subdivide solely based on geometric complexity, gstacks specifically exploit attribute uniformity to reduce memory footprint, serving as the fundamental building block for the QuadStack hierarchy [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A rectangular region of stacks $S$ defined by coordinates $[x_{min}, y_{min}, x_{max}, y_{max}]$ within the volume grid [S1, unofficial]. Each stack $S_{x,y}$ in this region must have the same number of intervals $n$ and an identical sequence of attribute values $a_1, a_2, \dots, a_n$ [S1, unofficial].
- **Output**: A gstack structure $G$ consisting of a sequence of $n$ intervals [S1, unofficial].

### Data Structures
A gstack $G$ is encoded as a sequence of $n$ intervals $i_1, i_2, \dots, i_n$ [S1, unofficial]. Each interval $i_k$ is a tuple:
$$ i_k = \langle a_k, H_k \rangle $$
Where:
- $a_k$: The attribute value (scalar or vector) common to all stacks in the region for the $k$-th interval [S1, unofficial].
- $H_k$: A heightfield (2D array) with dimensions $(x_{max} - x_{min} + 1) \times (y_{max} - y_{min} + 1)$ [S1, unofficial].

### Layout and Mapping
The heightfield $H_k$ stores the geometric extent (height) of the $k$-th interval for each stack in the region [S1, unofficial]. The height $h_{x,y}$ of a specific stack $S_{x,y}$ is mapped to the heightfield coordinate $(x - x_{min}, y - y_{min})$ [S1, unofficial]. Specifically:
$$ H_k[x - x_{min}, y - y_{min}] = h_{x,y,k} $$
where $h_{x,y,k}$ is the height of the $k$-th interval in stack $S_{x,y}$ [S1, unofficial].

### Constraints and Properties
1. **Attribute Uniformity**: All stacks covered by the gstack share the exact same attribute sequence $a_1, \dots, a_n$ [S1, unofficial].
2. **Non-Intersecting Heightfields**: Because the intervals are ordered consistently for all stacks, the heightfields never intersect [S1, unofficial]. For any coordinate $(i, j)$ in the heightfield domain and any interval index $k$:
   $$ h_{i,j,k} < h_{i,j,k+1} $$
   This ensures that the $k$-th interval is always below the $(k+1)$-th interval for every stack in the region [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Construction and access logic inferred from layout rules [S1, unofficial].
class GStack:
    def __init__(self, x_min, y_min, x_max, y_max, attribute_sequence):
        self.x_min = x_min
        self.y_min = y_min
        self.x_max = x_max
        self.y_max = y_max
        self.n = len(attribute_sequence)
        self.attributes = attribute_sequence
        width = x_max - x_min + 1
        height = y_max - y_min + 1
        # Initialize heightfields for each interval
        self.heightfields = [[[0] * height for _ in range(width)] for _ in range(self.n)]

    def set_stack_height(self, x, y, k, height_val):
        local_x = x - self.x_min
        local_y = y - self.y_min
        if not (0 <= local_x < len(self.heightfields[0][0]) and 0 <= local_y < len(self.heightfields[0])):
            raise ValueError("Coordinates out of GStack bounds")
        self.heightfields[k][local_x][local_y] = height_val

    def get_stack_height(self, x, y, k):
        local_x = x - self.x_min
        local_y = y - self.y_min
        return self.heightfields[k][local_x][local_y]
```

### Edge Cases and Pitfalls
- **Boundary Conditions**: The heightfield dimensions are strictly determined by the bounding box [S1, unofficial]. Accessing coordinates outside $[x_{min}, x_{max}] \times [y_{min}, y_{max}]$ is invalid [G].
- **Attribute Mismatch**: A gstack cannot represent a region where stacks have different attribute sequences [S1, unofficial]. If attributes differ, the region must be subdivided or represented by multiple gstacks [G].
- **Height Ordering**: The implementation must ensure that $h_{i,j,k} < h_{i,j,k+1}$ holds for all $i, j$ [S1, unofficial]. Violating this implies a logical error in the volume data or the stacking process [G].

## Subtopics

### Wildcard Interval (*-interval)
A wildcard interval is a special type of interval used in internal nodes of the QuadStack hierarchy to represent gaps between terminal intervals [S1, unofficial]. It corresponds to intervals stored elsewhere in descendant or ancestor nodes [S1, unofficial]. Defined as $i_w = \langle *, H_w \rangle$, it has an associated heightfield $H_w$ [S1, unofficial]. If $i_w$ represents intervals $i_k, \dots, i_{k+j}$ in a descendant node $n_d$, $H_w$ combines the heightfields of the top intervals of all sets of intervals in descendant nodes grouped by $i_w$, such that $H_{k+j}$ corresponds to a quadrant of $H_w$ [S1, unofficial]. If $i_w$ refers to intervals in an ancestor node $n_a$, $H_w$ corresponds to a quadrant of the ancestor's heightfield $H_{k+j}$, as the ancestor covers a larger part of the xy-plane [S1, unofficial]. This allows flexible gstacks that combine terminal intervals with areas lacking information at the current level [S1, unofficial].

## Operating characteristics
- **Compression**: Compresses attribute information significantly when large rectangular regions share identical attribute sequences [S1, unofficial]. Geometry is stored as heightfields, which can be further compressed using standard heightfield encoding methods [S1, unofficial].
- **Memory**: Space complexity is proportional to the number of intervals $n$ times the area of the region $(x_{max}-x_{min}+1)(y_{max}-y_{min}+1)$ [S1, unofficial]. It is more space-efficient than storing individual stacks when attribute redundancy is high [S1, unofficial].
- **Performance**: Building gstacks requires identifying regions of identical attribute sequences [S1, unofficial]. Querying a stack's height is $O(1)$ given the coordinates [G].
- **Limitations**: Only compresses attributes; geometric variations are still stored explicitly in heightfields [S1, unofficial]. Not suitable for regions with high attribute variability [G].

## Background
The gstack was introduced as part of the QuadStack representation for efficient encoding and direct rendering of layered datasets [S1, unofficial]. It builds on the Stack-Based Representation (SBR) by adding a layer of attribute-based compression [S1, unofficial]. The method was developed to address the inefficiencies of standard spatial decompositions that do not exploit attribute uniformity [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
