# BasicDAG

Also called: BaseDAG

## Overview
BasicDAG is a sparse voxel directed acyclic graph (DAG) structure that stores node indices in a flat, contiguous array to enable fast traversal and leaf extraction [S1, unofficial]. It distinguishes itself from sibling structures by relying on a static array layout with direct indexing for node access, rather than pointer-based navigation or complex hash tables [S1, unofficial]. It serves as a foundational implementation for DAG-based volume representation, focusing on memory efficiency through node sharing and simple array-based lookup [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of voxel data to be compressed into a DAG structure.
- Output: A `StaticArray<uint32>` containing the serialized node indices of the DAG.

**Data Structures:**
The core data structure is a single flat array:
- `data`: `StaticArray<uint32>`. Stores node indices sequentially in memory. Each element is a 32-bit unsigned integer representing the index of a node in the DAG [S1, unofficial].

**Algorithm Steps:**
1. **Construction:** The DAG is built by recursively subdividing the 3D space. Isomorphic nodes (nodes with identical children and data) are merged to form the DAG structure, ensuring each unique node is stored only once [S1, unofficial].
2. **Storage:** The resulting DAG nodes are stored in the `data` array. The index in the array corresponds to the node's identifier [S1, unofficial].

**Access Methods:**
The following operations define how to navigate the structure stored in `data`:

1. **Get Node:** Retrieve the node index at a specific level and index.
   - Formula: `node_index = data[index]`
   - Definition: `index` is the position in the `data` array [S1, unofficial].

2. **Get Child Index:** Compute the index of a specific child of a parent node.
   - Formula: `child_index = data[index + Utils::child_offset(childMask, child)]`
   - Definition:
     - `index`: The index of the parent node in the `data` array.
     - `childMask`: A bitmask indicating which children exist for the parent node.
     - `child`: The identifier of the specific child (e.g., 0-7 for an octree child).
     - `Utils::child_offset`: A helper function that calculates the offset based on the `childMask` and `child` identifier [S1, unofficial].

3. **Get Leaf:** Extract leaf node data.
   - Formula: `leaf = { data[index], data[index + 1] }`
   - Definition: `index` is the starting position of the leaf node in the `data` array. A leaf is represented by two consecutive `uint32` values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BasicDAG:
    def __init__(self, data_array):
        """
        Initialize BasicDAG with a flat array of uint32 node indices.
        :param data_array: List or array of uint32 values representing the DAG nodes.
        """
        self.data = data_array

    def get_node(self, index):
        """
        Returns the node index at the given position.
        :param index: The index in the data array.
        :return: The node index stored at data[index].
        """
        return self.data[index]

    def get_child_index(self, index, child_mask, child):
        """
        Computes the child node index by adding a precomputed offset to the parent index.
        :param index: The index of the parent node in the data array.
        :param child_mask: Bitmask indicating existing children.
        :param child: Identifier of the specific child (e.g., 0-7).
        :return: The index of the child node in the data array.
        """
        # Utils::child_offset is a helper function defined elsewhere in the implementation
        # that calculates the offset based on child_mask and child.
        offset = self._utils_child_offset(child_mask, child)
        return self.data[index + offset]

    def get_leaf(self, index):
        """
        Extracts a leaf node by reading two consecutive uint32 values.
        :param index: The starting index of the leaf node in the data array.
        :return: A tuple (data[index], data[index + 1]) representing the leaf.
        """
        return (self.data[index], self.data[index + 1])

    def _utils_child_offset(self, child_mask, child):
        """
        Placeholder for the helper function Utils::child_offset.
        In a full implementation, this would calculate the offset based on the
        bitmask of existing children and the target child index.
        :param child_mask: Bitmask of existing children.
        :param child: Target child index.
        :return: Integer offset.
        """
        # Implementation detail not provided in source.
        # Typically involves counting set bits in child_mask up to 'child'.
        pass
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The use of `uint32` limits the maximum number of nodes to $2^{32}-1$. If the DAG exceeds this size, overflow or incorrect indexing may occur [S1, unofficial].
- **Boundaries:** Accessing `data[index + 1]` in `get_leaf` requires ensuring that `index + 1` is within the bounds of the `data` array. If the leaf is at the end of the array, this may cause an out-of-bounds error [S1, unofficial].
- **Degenerate Input:** The structure assumes a valid DAG where isomorphic nodes have been merged. If the input data does not result in a DAG (e.g., due to construction errors), the traversal logic may fail or produce incorrect results [S1, unofficial].

## Subtopics

### [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md)
Stores uncompressed color data for DAG leaves in a flat array, indexed by leaf count.

### [BasicDAGColorsBase](6742-basicdagcolorsbase.md)
Base class for DAG color data, decoupling leaf count storage from the main geometry array using an enclosed leaves array.

## Operating characteristics
- **Time Complexity:** Node access is $O(1)$ due to direct array indexing [S1, unofficial]. Child index computation depends on the `Utils::child_offset` function, which is likely $O(1)$ or $O(\log N)$ depending on implementation [S1, unofficial].
- **Memory Complexity:** Memory usage is proportional to the number of unique nodes in the DAG, stored as `uint32` values. The flat array structure minimizes overhead compared to pointer-based structures [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of unique nodes. As the DAG becomes more sparse and shares more nodes, memory efficiency improves [S1, unofficial].
- **Strengths:** Simple implementation, fast node access, low memory overhead for node indices [S1, unofficial].
- **Limitations:** Limited by `uint32` index size, requires careful handling of leaf data extraction [S1, unofficial].
- **When to use:** Suitable for applications requiring fast traversal and simple memory layout, where the DAG size fits within `uint32` limits [S1, unofficial].
- **When not to use:** Not suitable for very large DAGs exceeding $2^{32}$ nodes or when complex color data storage is required without extensions [S1, unofficial].

## Background
BasicDAG is part of the HashDAG project by Phyronnaz [S1, unofficial]. It is an unofficial implementation, likely from a GitHub repository, and serves as a basic building block for sparse voxel DAG representations [S1, unofficial].

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
