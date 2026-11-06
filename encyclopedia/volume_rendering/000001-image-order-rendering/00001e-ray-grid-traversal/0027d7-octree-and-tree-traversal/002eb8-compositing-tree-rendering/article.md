# Compositing Tree Rendering

Also called: Incremental Compositing, Modified subtree rendering, Partial image compositing, Selective Subtree Rendering

## Overview
Compositing Tree Rendering is a time-varying volume rendering strategy that minimizes computational cost by rendering only modified data subtrees and compositing them with retained partial images of unmodified data [S1, S2, unofficial]. It differs from standard incremental rendering by maintaining a persistent hierarchical structure (the compositing tree) that mirrors the data's compressed octree, allowing for efficient spatial updates without re-rendering the entire volume for every time step [S1, S2, unofficial].

## How it works
**Inputs:** A sequence of compressed octree representations of time-varying volume data; a fixed viewing position and camera parameters.
**Outputs:** A final composite image for each time step.

**Data Structures:**
1.  **Compositing Tree:** A pointer-based octree with the exact same structure as the input compressed octree [S1, S2, unofficial].
    *   **Leaves:** Each leaf node stores a `partial_image` (a 2D buffer of color and opacity values) rendered from the corresponding leaf in the compressed octree [S1, S2, unofficial].
    *   **Interior Nodes:** Each interior node stores a `partial_image` which is the result of compositing the images of all its children [S1, S2, unofficial].
2.  **Volume Data:** The original uncompressed volume data is retained to handle boundary values between uniform blocks [S1, S2, unofficial].

**Algorithm Steps:**

1.  **Initialization (Time Step 0):**
    *   Render the first compressed octree completely using standard ray casting [S1, S2, unofficial].
    *   Construct the initial Compositing Tree. For each leaf, render the corresponding subvolume and store the result in the leaf's `partial_image` [S1, S2, unofficial].
    *   For each interior node, compute its `partial_image` by compositing the `partial_image`s of its eight children using front-to-back compositing [S1, S2, unofficial].
    *   The root's `partial_image` is the final image for time step 0 [S1, S2, unofficial].

2.  **Update (Time Step $t > 0$):**
    *   Identify modified subtrees in the compressed octree by comparing the current tree structure/data with the previous time step [S1, S2, unofficial].
    *   For each modified subtree:
        *   Render the modified data using ray casting [S1, S2, unofficial].
        *   Create a new branch in the Compositing Tree to represent this modified data [S1, S2, unofficial].
        *   Splice this new branch into the Compositing Tree, replacing the old branch corresponding to the modified region [S1, S2, unofficial].
    *   **Recomposition:**
        *   For the top of each new branch, composite the new leaf images with their siblings [S1, S2, unofficial].
        *   Propagate changes upward: Recomposite all ancestor nodes from the modified branch up to the root [S1, S2, unofficial].
        *   The root's `partial_image` is the final image for time step $t$ [S1, S2, unofficial].

**Optimizations for Constant Subvolumes:**
When an octant represents a subvolume with constant data values (uniform block), rendering is optimized by avoiding sample coordinate calculation and trilinear interpolation [S1, S2, unofficial].

*   **General Discretized Volume Rendering Integral:**
    The accumulated color $C$ up to $n$ sample points is:
    $$
    C = \sum_{i=1}^{n} C_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $C_i$ is the color and $\alpha_i$ is the opacity at sample $i$ [S1, S2, unofficial].

*   **Constant Subvolume Simplification:**
    If all sample points in a subvolume have identical color $C$ and opacity $\alpha$, the equation simplifies to a geometric series [S1, S2, unofficial]:
    $$
    C_{total} = C \alpha \frac{1 - (1 - \alpha)^n}{1 - (1 - \alpha)} = C \frac{1 - (1 - \alpha)^n}{\alpha} \cdot \alpha \quad \text{(Note: Source [S1, unofficial] gives } C \alpha (1 - (1-\alpha)^{n-1}) / (1 - (1-\alpha)) \text{ which simplifies to } C \frac{1-(1-\alpha)^n}{\alpha} \text{ if derived correctly from standard front-to-back, but source text explicitly states: } C \alpha (1 - (1-\alpha)^{n-1}) / \alpha \text{ is likely a typo for the standard form. Source [S2, unofficial] states: } C^n = C \frac{1-(1-a)^n}{1-(1-a)} \text{ where } a \text{ is opacity.})
    $$
    *Correction based on Source [S2, unofficial] explicit formula:*
    $$
    C^n = C \frac{1 - (1 - a)^n}{1 - (1 - a)} = C \frac{1 - (1 - a)^n}{a}
    $$
    where $C$ is the constant color, $a$ is the constant opacity, and $n$ is the number of samples collected along the ray within the subvolume [S1, S2, unofficial].
    *   Only the number of samples $n$ is required. Sample coordinates and interpolation are avoided [S1, S2, unofficial].

**Boundary Handling:**
*   Leaves in the compressed octree represent uniform blocks. However, boundaries between these blocks must be rendered carefully to avoid artifacts [S1, S2, unofficial].
*   To avoid traversal overhead for boundary values, the original volume data is kept uncompressed. The octree structure acts as a map into this volume data, allowing direct access to boundary voxels when necessary [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_time_step(compressed_octree, compositing_tree, volume_data, time_step):
    """
    Renders a time step using Compositing Tree Rendering.
    
    Args:
        compressed_octree: The compressed octree for the current time step.
        compositing_tree: The persistent compositing tree from the previous time step.
        volume_data: The uncompressed volume data array.
        time_step: The current time step index.
        
    Returns:
        final_image: The composite image for the current time step.
    """
    if time_step == 0:
        # Initialize Compositing Tree
        compositing_tree = build_compositing_tree(compressed_octree, volume_data)
        return compositing_tree.root.partial_image
    
    # Identify modified subtrees
    modified_subtrees = identify_modifications(compressed_octree, compositing_tree)
    
    for subtree in modified_subtrees:
        # Render the modified subtree
        new_branch = render_subtree(subtree, volume_data)
        
        # Splice the new branch into the compositing tree
        spliced_node = splice_into_tree(compositing_tree, subtree, new_branch)
        
        # Recomposite ancestors
        recomposite_ancestors(spliced_node)
        
    return compositing_tree.root.partial_image

def render_subtree(subtree, volume_data):
    """
    Renders a subtree, using optimization for constant subvolumes.
    """
    if subtree.is_constant():
        # Get number of samples n along the ray for this subtree
        n = count_samples(subtree, volume_data)
        C = subtree.constant_color
        a = subtree.constant_opacity
        
        # Apply simplified formula
        # C_total = C * (1 - (1 - a)^n) / a
        # Note: Handle case where a is 0 or 1 to avoid division by zero
        if a == 0:
            return empty_image()
        elif a == 1:
            return solid_color_image(C)
        else:
            color = C * (1 - (1 - a)**n) / a
            return solid_color_image(color)
    else:
        # Standard ray casting for non-constant data
        return standard_ray_cast(subtree, volume_data)

def recomposite_ancestors(node):
    """
    Recomposites the partial images up the tree from the given node.
    """
    if node.is_root():
        return
    
    parent = node.parent
    # Composite all children of the parent
    parent.partial_image = composite_images([child.partial_image for child in parent.children])
    
    # Recurse up
    recomposite_ancestors(parent)

def composite_images(images):
    """
    Performs front-to-back compositing of a list of images.
    """
    if not images:
        return empty_image()
    
    result = images[0]
    for i in range(1, len(images)):
        result = front_to_back_composite(result, images[i])
    return result

def front_to_back_composite(back_image, front_image):
    """
    Composites front_image over back_image.
    """
    # Standard front-to-back compositing
    # C_out = C_front + C_back * (1 - alpha_front)
    # alpha_out = alpha_front + alpha_back * (1 - alpha_front)
    return {
        'color': front_image.color + back_image.color * (1 - front_image.alpha),
        'alpha': front_image.alpha + back_image.alpha * (1 - front_image.alpha)
    }
```

## Operating characteristics
*   **Time Complexity:** The rendering cost is proportional to the amount of *modified* data between time steps, not the total volume size [S1, S2, unofficial]. If only a small portion of the volume changes, the cost is significantly lower than full rendering [S1, S2, unofficial].
*   **Memory Complexity:** Requires storing the Compositing Tree, which holds partial images for all nodes in the octree. This is additional memory overhead compared to standard rendering, but the tree structure is compact [S1, S2, unofficial].
*   **Performance:** Significant savings in time domain are achieved because only the difference between consecutive time steps is rendered [S1, S2, unofficial]. The time to read encoded data is also reduced in proportion to the compression rate [S1, S2, unofficial].
*   **Strengths:**
    *   Efficient for time-varying data with localized changes [S1, S2, unofficial].
    *   Optimizations for constant subvolumes eliminate interpolation and coordinate calculation overhead [S1, S2, unofficial].
    *   Front-to-back rendering and transparent space skipping reduce unnecessary computation [S1, S2, unofficial].
*   **Limitations:**
    *   **Fixed Viewing Position:** The optimization assumes a fixed viewing position. Changing the view requires rerendering the entire tree and creating a new compositing tree [S1, S2, unofficial].
    *   **Temporal Navigation:** To allow random navigation through time, complete trees must be saved at regular intervals, increasing storage requirements [S1, S2, unofficial].
    *   **Approximation:** Due to opacity accumulation obscuring back details, unchanged subtrees in the back may not be completely sampled. This is an approximation that may introduce visual artifacts if the front changes significantly [S1, S2, unofficial].
    *   **Boundary Overhead:** Handling boundaries between uniform blocks requires access to uncompressed data, adding complexity [S1, S2, unofficial].

## Background
Compositing Tree Rendering was introduced in the context of efficient encoding and rendering of time-varying volume data [S1, S2, unofficial]. It was developed to address the high computational cost of rendering large, time-varying datasets by leveraging data compression and hierarchical structures [S1, S2, unofficial]. The method was tested on datasets such as Turbulent Vortex Flow, Thermal Convection, and Heart Modeling, with spatial resolutions up to $128^3$ [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
