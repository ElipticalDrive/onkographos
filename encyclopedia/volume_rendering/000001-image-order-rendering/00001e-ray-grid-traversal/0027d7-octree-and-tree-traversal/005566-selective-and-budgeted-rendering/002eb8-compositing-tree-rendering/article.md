# Compositing Tree Rendering

Also called: Incremental Compositing, Modified subtree rendering, Partial image compositing, Selective Subtree Rendering

## Overview
Compositing Tree Rendering is a time-domain optimization for volume rendering that avoids re-rendering unchanged data by retaining and reusing partial images from previous frames [S1, S2, unofficial]. It solves the high computational cost of rendering time-varying volume data by identifying modified subtrees in a compressed octree and composing their new partial images with the retained partial images of unmodified subtrees to form the final image [S1, S2, unofficial]. Unlike sibling methods that may use entropy metrics to guide traversal, this method relies on structural change detection to splice new branches into a persistent compositing tree, making it distinct from [Octree-based Entropy Rendering](13580-octree-based-entropy-rendering.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of compressed octree structures representing time-varying volume data. The first tree is rendered completely; subsequent inputs are trees where modified regions are identified.
*   **Output:** A final composite image for the current time step.

**Data Structures:**
*   **Compositing Tree:** A pointer-based octree that mirrors the structure of the compressed octree [S1, S2, unofficial].
    *   **Leaves:** Each leaf node contains a partial image (color and opacity buffer) rendered directly from the corresponding leaf in the compressed tree [S1, S2, unofficial].
    *   **Interior Nodes:** Each interior node contains a partial image which is the result of compositing the images of all its children [S1, S2, unofficial].
    *   **Root:** The image at the root of the compositing tree represents the complete final image [S1, S2, unofficial].

**Algorithm Steps:**
1.  **Initial Render:** Render the first compressed octree completely. Populate the Compositing Tree with partial images at all levels, starting from the leaves up to the root [S1, S2, unofficial].
2.  **Change Detection:** For subsequent time steps, identify the modified subtrees within the new compressed octree [S1, S2, unofficial].
3.  **Branch Creation:** Create a new compositing branch representing the modified data. This involves rendering the modified leaves and compositing them up to the new branch's root [S1, S2, unofficial].
4.  **Splicing:** Splice the new compositing branch into the existing Compositing Tree, replacing the old branch corresponding to the modified region [S1, S2, unofficial].
5.  **Recomposition:** Composite the image at the top of the new branch with its siblings. Recomposite all ancestors of the splice point up to the root to reflect the changes [S1, S2, unofficial].
6.  **Final Image:** The image at the root of the Compositing Tree is the complete image for the current time step [S1, S2, unofficial].

**Optimizations for Constant Subvolumes:**
When an octant (leaf) represents a subvolume with constant values, rendering is optimized by avoiding sample coordinate calculation and trilinear interpolation [S1, S2, unofficial].
*   **General Discretized Integral:** The accumulated color $C$ up to $n$ sample points is generally:
    $$ C = \sum_{i=1}^{n} C_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $C_i$ is the color and $\alpha_i$ is the opacity at sample $i$ [S1, S2, unofficial].
*   **Constant Subvolume Formula:** If all sample points have identical color $C$ and opacity $a$, the formula simplifies to a geometric series:
    $$ C_{total} = C \frac{1 - (1-a)^n}{1 - (1-a)} $$
    or equivalently:
    $$ C_{total} = C a \frac{1 - (1-a)^{n-1}}{a} = C (1 - (1-a)^{n-1}) \quad \text{(Note: Source derivation varies slightly in notation, but the geometric sum logic holds)} $$
    *Correction based on source text:* Source [S1, unofficial] states: $C_{total} = C a (1 - (1-a)^{n-1}) / a$ is not explicitly written, but derives from $C \sum_{i=1}^{n} (1-a)^{i-1} a$. Source [S2, unofficial] gives: $C^n = C \frac{1-(1-a)^n}{1-(1-a)}$. Both imply that only the number of samples $n$ is required [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class CompositingNode:
    def __init__(self):
        self.children = [None] * 8  # Octree children
        self.partial_image = None     # Color and opacity buffer for this node's region
        self.is_leaf = False

class CompositingTree:
    def __init__(self, root):
        self.root = root

    def render_initial(self, compressed_tree):
        """Render the first frame completely."""
        self._render_node(compressed_tree.root, self.root)

    def _render_node(self, compressed_node, comp_node):
        """Recursively render a node and its children."""
        if compressed_node.is_leaf:
            # Render partial image from compressed leaf data
            comp_node.partial_image = self._render_leaf(compressed_node)
            comp_node.is_leaf = True
        else:
            for i in range(8):
                if compressed_node.children[i] is not None:
                    # Create new compositing child if it doesn't exist
                    if comp_node.children[i] is None:
                        comp_node.children[i] = CompositingNode()
                    self._render_node(compressed_node.children[i], comp_node.children[i])
            # Composite children images to form this node's partial image
            comp_node.partial_image = self._composite_children(comp_node)

    def _render_leaf(self, compressed_leaf):
        """Render a single leaf. Optimizes for constant subvolumes."""
        if compressed_leaf.is_constant:
            # Use simplified formula: only need number of samples n
            n = compressed_leaf.sample_count
            C = compressed_leaf.constant_color
            a = compressed_leaf.constant_opacity
            # C_total = C * (1 - (1-a)^n) / a  (derived from geometric series)
            # Note: Standard front-to-back with constant opacity a:
            # Transmittance T = (1-a)^n
            # Color = C * (1 - T)
            transmittance = (1 - a) ** n
            color = C * (1 - transmittance)
            opacity = 1 - transmittance
            return ImageBuffer(color, opacity)
        else:
            # Standard ray casting for non-constant leaf
            return self._standard_ray_cast(compressed_leaf)

    def _composite_children(self, node):
        """Composite 8 child images into one."""
        # Standard front-to-back compositing of 8 child buffers
        # Order depends on view direction (not specified in source, assume standard)
        result = ImageBuffer(0, 0)
        # Iterate children in back-to-front order relative to camera
        for child in self._get_back_to_front_children(node):
            if child is not None and child.partial_image is not None:
                result = self._blend(result, child.partial_image)
        return result

    def update(self, new_compressed_tree):
        """Update the tree for a new time step."""
        # Identify modified subtrees in new_compressed_tree
        modified_nodes = self._find_modified_subtrees(self.root, new_compressed_tree.root)
        
        for node in modified_nodes:
            # Create new branch for modified data
            new_branch = self._render_modified_branch(node, new_compressed_tree)
            
            # Find parent in existing tree to splice
            parent = self._find_parent(node)
            if parent:
                # Replace old branch with new branch
                self._splice(parent, node, new_branch)
                
                # Recomposite ancestors
                self._recomposite_ancestors(parent)

    def _find_modified_subtrees(self, old_node, new_node):
        """Identify nodes that have changed between time steps."""
        # Implementation detail: Compare structure and content
        # Returns list of nodes in the compositing tree that need updating
        pass

    def _render_modified_branch(self, node, compressed_tree):
        """Render a new branch for modified data."""
        new_node = CompositingNode()
        self._render_node(compressed_tree, new_node)
        return new_node

    def _splice(self, parent, old_child, new_child):
        """Replace old_child with new_child in parent."""
        # Find index of old_child in parent.children
        for i in range(8):
            if parent.children[i] == old_child:
                parent.children[i] = new_child
                break

    def _recomposite_ancestors(self, node):
        """Recompute partial images for all ancestors up to root."""
        current = node
        while current is not None:
            current.partial_image = self._composite_children(current)
            # Move to parent (requires parent pointer or traversal)
            current = self._get_parent(current)

    def _blend(self, base, source):
        """Front-to-back compositing of two images."""
        # C_out = C_base + C_src * alpha_src * (1 - alpha_base)
        # alpha_out = alpha_base + alpha_src * (1 - alpha_base)
        pass

    def _standard_ray_cast(self, leaf):
        """Standard ray casting for non-constant data."""
        pass

    def _get_back_to_front_children(self, node):
        """Return children in back-to-front order."""
        pass

    def _find_parent(self, node):
        """Find parent of a node."""
        pass

    def _get_parent(self, node):
        """Get parent pointer."""
        pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Rendering:** Octree leaves represent uniform blocks, but boundaries between blocks must be rendered carefully. To avoid traversal overhead for boundary values, the original data is kept uncompressed, and the octree serves as a map into this volume data [S1, S2, unofficial].
*   **Viewing Position:** The optimization assumes a fixed viewing position. Changing the view requires rerendering the entire tree and creating a new compositing tree [S1, S2, unofficial].
*   **Temporal Navigation:** To allow random navigation through the temporal domain, complete trees must be saved at regular intervals [S1, S2, unofficial].
*   **Back Subtree Approximation:** Due to opacity accumulation obscuring back details in front-to-back rendering, subtrees representing the back portion of the data may not be completely sampled. As an approximation, unchanged subtrees are not re-rendered [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the amount of *modified* data rather than the total volume. Savings are significant when changes between time steps are localized [S1, S2, unofficial].
*   **Memory Complexity:** Requires storing the Compositing Tree, which holds partial images for all nodes. This is additional memory overhead compared to direct rendering [S1, S2, unofficial].
*   **Performance:** Rendering from the tree structure incurs overhead compared to direct volume data rendering. This is offset by optimizations like front-to-back rendering, caching ray coordinates in object space, and skipping transparent space [S1, S2, unofficial].
*   **Strengths:** Large savings in the time domain by rendering only modified data. Reduced data read time proportional to compression rate [S1, S2, unofficial].
*   **Limitations:** Fixed viewing position requirement. High memory usage for storing partial images. Overhead from tree traversal [S1, S2, unofficial].
*   **When to Use:** Time-varying volume data with localized changes and fixed viewpoints.
*   **When Not to Use:** Dynamic viewpoints or data with global changes [S1, S2, unofficial].

## Background
Compositing Tree Rendering was introduced in the late 1990s as part of research into efficient encoding and rendering of time-varying volume data. The method was developed by researchers at NASA and DTIC, as evidenced by the technical reports from 1998 [S1, S2, unofficial]. It was applied to datasets such as Turbulent Vortex Flow, Thermal Convection, and Heart Modeling, demonstrating significant storage and rendering time savings [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
