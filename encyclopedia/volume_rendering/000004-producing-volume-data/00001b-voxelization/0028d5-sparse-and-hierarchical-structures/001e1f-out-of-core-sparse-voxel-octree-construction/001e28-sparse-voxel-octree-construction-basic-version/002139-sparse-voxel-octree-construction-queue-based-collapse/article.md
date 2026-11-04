# Sparse Voxel Octree Construction (Queue-Based Collapse)

Also called: Builder collapse, NodeQueue processing, Queue-based SVO construction, Queue-based hierarchical tree construction, Sparse Voxel Octree (SVO) Builder with Queue-Based Collapse, Zazen Builder

## Overview
This method constructs a Sparse Voxel Octree (SVO) from a stream of indexed leaves by maintaining a fixed-size array of queues, one per tree depth level, and collapsing homogeneous subtrees into single leaf or invalid nodes to compress the structure [S1, unofficial]. It differs from brute-force or sorting-based builders by processing leaves in a single pass without requiring full in-memory tree storage or pre-sorting [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A stream of indexed leaves, where each leaf contains an `index` and a `leaf` data structure [S1, unofficial]. The stream is assumed to be sorted by index [G].
- **Output:** A serialized SVO written to disk via a `BlockWriter`, consisting of Voxel and Leaf nodes [S1, unofficial].

**Data Structures:**
- `Node`: A union-based structure representing either a Voxel or a Leaf [S1, unofficial].
  ```python
  class Node:
      def __init__(self, leaf=None, voxel=None):
          self.leaf = leaf
          self.voxel = voxel
          self.is_leaf = leaf is not None
          self.is_valid = leaf is not None or voxel is not None
  ```
- `Voxel`: A non-leaf node structure [S1, unofficial].
  ```python
  class Voxel:
      def __init__(self):
          self.child = 0  # Offset to children in block
          self.valid = 0  # 8-bit mask for visible children
          self.leaf = 0   # 8-bit mask for leaf children
  ```
- `Leaf`: A leaf node structure [S1, unofficial].
  ```python
  class Leaf:
      def __init__(self, rgba=None):
          self.rgba = rgba if rgba else [0, 0, 0, 0]
      def valid(self):
          return self.rgba[3] != 0x00
  ```
- `NodeQueue`: A fixed-size queue for nodes at a specific depth level [S1, unofficial].
  ```python
  class NodeQueue:
      def __init__(self):
          self.data = [None] * 8
          self.size = 0
      def push(self, node):
          self.data[self.size] = node
          self.size += 1
      def full(self):
          return self.size == 8
      def empty(self):
          return self.size == 0
      def clear(self):
          self.size = 0
      def __getitem__(self, index):
          return self.data[index]
  ```
- `Builder`: The main builder class [S1, unofficial].
  ```python
  class Builder:
      def __init__(self, queue_count, writer):
          self.queues = [NodeQueue() for _ in range(queue_count)]
          self.queue_count = queue_count
          self.stream_index = 0
          self.writer = writer
  ```

**Parameters:**
- `queue_count`: The number of queues used for the tree construction, typically 8 for an octree [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Initialize `stream_index` to 0 and all `queues` to empty [S1, unofficial].
2. **Processing a Leaf (`add_leaf`):**
   - Calculate `leaf_count = ileaf.index - stream_index` [S1, unofficial].
   - Loop while `leaf_count > 0`:
     - Calculate `power`: if `leaf_count >= 8`, `power = floor(log(leaf_count) / log(8))`; else `power = 0` [S1, unofficial].
     - Find `non_empty_i`: index of the first non-empty queue in `queues[0..queue_count-1]` [S1, unofficial]. If all empty, `non_empty_i = queue_count` [S1, unofficial].
     - Calculate `insert_i = min(power, non_empty_i)` [S1, unofficial].
     - If `insert_i >= queue_count`, throw error "Homogenous model" and write the leaf directly [S1, unofficial].
     - Create `to_push` Node: if `ileaf.leaf.valid()` is true, `to_push = Node(ileaf.leaf)`; else `to_push = Node()` (invalid) [S1, unofficial].
     - Call `push_at(to_push, insert_i)` [S1, unofficial].
     - Subtract `pow(8, insert_i)` from `leaf_count` [S1, unofficial].
   - Set `stream_index = ileaf.index` [S1, unofficial].
3. **Pushing to Queue (`push_at`):** Push `node` to `queues[index]` and call `collapse(index)` [S1, unofficial].
4. **Collapse Logic (`collapse`):**
   - If `queues[index]` is not full, return immediately [S1, unofficial].
   - Initialize `parent_is_leaf = true` [S1, unofficial].
   - Initialize `parent` Voxel: `parent.valid = 0`, `parent.leaf = 0`, `parent.child = writer.pos()` [S1, unofficial].
   - Loop `j` from 0 to 7 over `queues[index]`:
     - If `queues[index][j].is_leaf` is false, set `parent_is_leaf = false` [S1, unofficial].
     - Update masks: If `queues[index][j].is_valid` is false, update `parent.valid` mask accordingly. If `queues[index][j].is_leaf` is true, update `parent.leaf` mask accordingly [S1, unofficial].
   - Determine `to_push`: If `parent_is_leaf`, `to_push = Node(queues[index][0].leaf)`. Else if `parent.valid == 0`, `to_push = Node()` (invalid). Else, write children to `writer` (leaf if `node.is_leaf`, voxel if `node.is_valid`), then `to_push = Node(parent)` [S1, unofficial].
   - After processing all 8 children, push `to_push` to the parent queue and clear `queues[index]` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Homogenous Model:** If `insert_i >= queue_count`, the algorithm throws an error and writes the leaf directly to bypass the tree structure for that segment [S1, unofficial].
- **Truncated Logic:** The exact logic for updating `parent.valid` and `parent.leaf` masks, as well as queue advancement after collapse, is not fully specified in the source [S1, unofficial].
- **Precision:** `log` and `floor` operations for `power` calculation should use floating-point arithmetic with care for large `leaf_count` values [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def add_leaf(self, ileaf_index, ileaf_leaf):
    leaf_count = ileaf_index - self.stream_index
    while leaf_count > 0:
        if leaf_count >= 8:
            power = int(math.log(leaf_count) / math.log(8))
        else:
            power = 0

        non_empty_i = self.queue_count
        for i in range(self.queue_count):
            if not self.queues[i].empty():
                non_empty_i = i
                break

        insert_i = min(power, non_empty_i)
        if insert_i >= self.queue_count:
            raise Exception("Homogenous model")
            # Not given in the sources.

        if ileaf_leaf.valid():
            to_push = Node(leaf=ileaf_leaf)
        else:
            to_push = Node()

        self.push_at(to_push, insert_i)
        leaf_count -= 8 ** insert_i

    self.stream_index = ileaf_index

def push_at(self, node, index):
    self.queues[index].push(node)
    self.collapse(index)

def collapse(self, index):
    if not self.queues[index].full():
        return

    parent_is_leaf = True
    parent = Voxel()
    parent.valid = 0
    parent.leaf = 0
    parent.child = self.writer.pos()

    for j in range(8):
        child = self.queues[index][j]
        if not child.is_leaf:
            parent_is_leaf = False
        # Not given in the sources. Standard logic would update parent.valid and parent.leaf masks here [G].

    if parent_is_leaf:
        to_push = Node(leaf=self.queues[index][0].leaf)
    elif parent.valid == 0:
        to_push = Node()
    else:
        # Write children to writer [S1, unofficial]
        # For each child, if it is a leaf, write the Leaf; if it is a valid voxel, write the Voxel.
        # Not given in the sources. Standard logic would push to_push to queues[index-1] (if index > 0) and clear queues[index].
        to_push = Node(voxel=parent)
        self.queues[index].clear()
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Low memory footprint, suitable for out-of-core construction, simple implementation [S1, unofficial].
- **Limitations:** Truncated logic in source makes full implementation difficult without inference [S1, unofficial].

## Background
This method is part of the Zazen project, an implementation of a volume renderer using Sparse Voxel Octrees [S1, unofficial]. It is part of a series of SVO construction methods, evolving from the Basic Version to include queue-based collapse for better compression [S1, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
