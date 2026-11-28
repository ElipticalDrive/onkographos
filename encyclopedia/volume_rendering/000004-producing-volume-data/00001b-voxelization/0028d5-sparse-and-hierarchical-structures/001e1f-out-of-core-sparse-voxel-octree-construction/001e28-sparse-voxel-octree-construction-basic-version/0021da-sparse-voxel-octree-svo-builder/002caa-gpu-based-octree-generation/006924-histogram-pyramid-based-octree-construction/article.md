# Histogram Pyramid-based Octree Construction

Also called: Octree Construction Using Histogram Pyramids

## Overview
This method initializes pointer-based octrees by parallelizing the construction of internal and leaf node lists using histogram pyramids [S1, unofficial]. It differs from standard SVO builders by explicitly separating internal and leaf structures during the GPU-based reduction phase, allowing for efficient pointer resolution via co-traversal [S1, unofficial].

## How it works
**Inputs:** A set of voxel coordinates (or binary classified input data) representing occupied regions.
**Outputs:** An octree structure consisting of an internal node list and a leaf node list, where internal nodes store pointers to children and leaf nodes store values.

**Data Structures:**
1.  **Internal Node List:** Stores internal nodes. Each node contains an array of pointers (size 8 for octrees) and potentially metadata. Pointers are initialized to `-1` (not-set flag).
2.  **Leaf Node List:** Stores leaf nodes. Each node contains a value and potentially metadata.
3.  **Histogram Pyramids:** Two temporary GPU-resident structures generated via parallel reductions:
    *   **Internal Histogram Pyramid:** Tracks the count of internal nodes at each level of the Morton order hierarchy.
    *   **Leaf Histogram Pyramid:** Tracks the count of leaf nodes at each level of the Morton order hierarchy.

**Algorithm Steps:**

1.  **Generate Histogram Pyramids:**
    Compute the internal and leaf histogram pyramids from the input voxel data using parallel reductions [S1, unofficial]. These pyramids temporarily store the cumulative counts of nodes at each level of the spatial hierarchy [S1, unofficial].

2.  **Parallel Node Construction:**
    Execute the construction phase in parallel for all $n_{internal} + n_{leaf}$ final octree nodes [S1, unofficial]. Distinguish between two traversal types:

    **A. Internal Node Traversal:**
    *   Initialize all pointers in the internal node list to `-1` [S1, unofficial].
    *   Traverse the internal histogram pyramid from root to children to resolve parent-child relationships [S1, unofficial].
    *   **Termination Condition:** Unlike voxel list generation which stops at the leaf level, this traversal can terminate at internal levels of the histogram pyramid [S1, unofficial].
    *   **Index Matching:** If an index is exactly matched during traversal, an internal node is found [S1, unofficial].
    *   **Parent Resolution:** To find the parent of an internal node at index $I$, look at the index one level above $I$ in the histogram pyramid [S1, unofficial].
        *   *Example:* To find the parent of index 6, increase by one to 7. Check histogram pyramid entries until finding the node corresponding to 7. The parent node is at index $8 - 1 = 7$ (assuming specific pyramid structure where 8 is the next boundary) [S1, unofficial].
    *   **Pointer Writing:** Children write their index into their parent’s pointer array [S1, unofficial].
    *   **Limitation:** This step initializes all internal nodes except the highest level (root), which points to leaves and is handled in leaf traversal [S1, unofficial].

    **B. Leaf Node Traversal:**
    *   Write values into leaf nodes [S1, unofficial].
    *   Set pointers from the lowest level internal nodes to leaf nodes [S1, unofficial].
    *   **Reverse Order:** Write pointers in reverse order [S1, unofficial].
    *   **Co-traversal:** Traverse both the internal and leaf histogram pyramids simultaneously (co-traversal) to find the leaf node's parent index [S1, unofficial].
    *   **Parent Retrieval:** Retrieve the parent node index from the internal node histogram pyramid while traversing the leaf histogram pyramid [S1, unofficial].
    *   **Pointer Writing:** Write the leaf node index into the parent internal node's pointer array [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Generate Histogram Pyramids
internal_hist_pyramid = compute_histogram_pyramid(internal_nodes_data)
leaf_hist_pyramid = compute_histogram_pyramid(leaf_nodes_data)

# Step 2: Parallel Construction
# Initialize pointers to -1
for node in internal_node_list:
    node.pointers = [-1, -1, -1, -1, -1, -1, -1, -1]

# Parallel Kernel for Internal Nodes
def internal_node_traversal(node_index):
    # Traverse internal histogram pyramid
    current_idx = node_index
    # Find parent by checking equality in histogram pyramid
    # If exact match found, parent is at index one level above
    parent_idx = find_parent_in_histogram(current_idx, internal_hist_pyramid)
    
    # Write child index to parent's pointer
    # Determine child slot based on Morton order or spatial position
    child_slot = get_child_slot(node_index)
    internal_node_list[parent_idx].pointers[child_slot] = node_index

# Parallel Kernel for Leaf Nodes
def leaf_node_traversal(leaf_index, leaf_value):
    # Co-traverse internal and leaf histogram pyramids
    internal_parent_idx, leaf_parent_idx = co_traverse_histograms(
        leaf_index, 
        internal_hist_pyramid, 
        leaf_hist_pyramid
    )
    
    # Write value to leaf node
    leaf_node_list[leaf_index].value = leaf_value
    
    # Write pointer from internal parent to leaf
    child_slot = get_child_slot(leaf_index)
    internal_node_list[internal_parent_idx].pointers[child_slot] = leaf_index

# Launch kernels
parallel_execute(internal_node_traversal, internal_node_list)
parallel_execute(leaf_node_traversal, leaf_node_list)
```

**Parameters:**
*   **Morton Order:** Required for spatial sorting. Adaptation from quad- to octree requires updating Morton code computations [S1, unofficial].
*   **Not-Set Flag:** `-1` [S1, unofficial].

**Edge Cases:**
*   **Root Node:** The highest level internal nodes point to leaves and are only written during leaf traversal [S1, unofficial].
*   **Empty Regions:** Handled by the binary classification and histogram reduction; empty regions do not generate nodes [S1, unofficial].

## Operating characteristics
*   **Complexity:** Parallel construction allows for $O(N)$ or $O(N \log N)$ depending on reduction implementation, where $N$ is the number of voxels/nodes.
*   **Memory:** Requires temporary storage for two histogram pyramids on the GPU [S1, unofficial].
*   **Strengths:** Fully parallelizable construction of both internal and leaf lists; efficient pointer resolution via histogram traversal [S1, unofficial].
*   **Limitations:** Requires significant GPU memory for temporary histogram structures; complex traversal logic for parent resolution [S1, unofficial].

## Background
This method is based on the work of Ziegler [S1, unofficial], who explained the use of histogram pyramids for initializing octrees. It generalizes previous quadtree construction methods to octrees by adapting Morton order computations [S1, unofficial]. The approach is part of the JiTTree construction pipeline, following the selection of optimal brick types [S1, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
