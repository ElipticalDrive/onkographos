# Structural and Glyph-based Visualization

Also called: Structural Focus+Context, Hybrid Rendering for Stress Tensor Fields

## Overview
Structural and Glyph-based Visualization is a family of interaction techniques that enhance volume rendering by integrating explicit geometric representations (glyphs, tensorlines, or structural graphs) with volume-rendered context. It solves the problem of visual clutter and occlusion in complex datasets (e.g., stress tensors or multimodal medical images) by allowing users to isolate specific regions of interest (focus) for detailed geometric analysis while maintaining a global context view via volume rendering [S1][S1]. Unlike pure volume rendering, which obscures internal structures, or pure glyph-based rendering, which lacks global continuity, this family dynamically adjusts opacity and geometry based on user-defined masks or structural relationships [S1][S1].

## Shared principles
The family shares a core mechanism of decoupling "focus" and "context" rendering paths to manage visual complexity.

1.  **Dual Representation:** The method maintains two parallel representations of the data:
    *   **Context:** A volume-rendered representation (e.g., ray-casted scalar invariants or multimodal textures) that provides global continuity and spatial awareness.
    *   **Focus:** An explicit geometric or structural representation (e.g., glyphs, tensorlines, or directed graphs) that provides high-fidelity detail for selected regions.

2.  **Selection-Driven Masking:** User interaction (e.g., selection in a diagram view or boolean query) defines a **mask** or **selection set**. This mask determines which voxels or features are rendered as geometry (focus) and which are rendered as volume (context).
    *   In glyph-based approaches, the mask $M(\mathbf{x})$ is binary: $M(\mathbf{x})=1$ for focus, $0$ for context [S1].
    *   In structural approaches, the mask is derived from a boolean query over classification criteria, mapped to cluster labels [S1].

3.  **Occlusion Handling:** Correct intersection between the volume-rendered context and the opaque geometric focus elements is critical. This is typically handled by:
    *   Rendering the geometry first into a depth buffer.
    *   During volume ray-casting, checking the depth buffer to stop volume traversal when a ray intersects a geometric element [S1].
    *   Alternatively, modulating volume opacity in focus regions to prevent visual overlap [S1].

4.  **Structural Graphs (for Structural SFC):** In Structural Focus+Context, the data is preprocessed into a **Directed Graph** $G$ where nodes are classification criteria and edges represent logical inclusion. This graph serves as a "grammar" for boolean arithmetic on classes, allowing the system to compute transfer functions based on structural relationships rather than independent criteria [S1].

## Subtopics

### [Hybrid Rendering for Stress Tensor Fields](8909-hybrid-rendering-for-stress-tensor-fields.md)
*   **What it is:** A technique for stress tensor fields that combines volume rendering of scalar invariants (e.g., von Mises stress) with explicit geometric representations (tensorlines or ellipsoid glyphs).
*   **What sets it apart:** It uses user selections from diagram views (scatterplots, Mohr diagrams) to define focus regions. It solves the problem of visual clutter in dense tensor fields by allowing interactive selection of specific regions for detailed geometric analysis while maintaining a global context view.
*   **When to choose it:** Use when visualizing complex tensor fields where internal structures (e.g., principal stress directions) are obscured by volume rendering, and users need to correlate tensor properties with spatial location.
*   **Link:** [Hybrid Rendering for Stress Tensor Fields](8909-hybrid-rendering-for-stress-tensor-fields.md)

### [Structural Focus+Context Rendering](14778-structural-focus-context-rendering.md)
*   **What it is:** A method for multiclassified voxel models (e.g., multimodal medical images) that outlines structural relationships between classification criteria using a directed graph and multimodal 3D texture mapping.
*   **What sets it apart:** Unlike prior systems that treat classification criteria independently, this method clusters voxels based on combined criteria, constructs a directed graph of logical inclusion relationships, and applies structural focus+context shading via transfer functions computed on the graph structure.
*   **When to choose it:** Use when exploring complex, multiclassified datasets where understanding the logical inclusion relationships between different tissue types or data modalities is critical.
*   **Link:** [Structural Focus+Context Rendering](14778-structural-focus-context-rendering.md)

### Similarity-Enhanced Isosurface Visualization
*   **Definition:** A focus+context direct volume rendering technique where the isosurface at a user-selected isovalue $h_u$ is the focus, and contextual regions are depicted sparsely based on local similarity stability [S1].
*   **Mechanism:** It uses an importance function $\gamma(\mathbf{x})$ to control the degree-of-interest. The function is defined as:
    $$ \gamma(\mathbf{x}) = \prod_{\mathbf{y} \in C(\mathbf{x})} \text{SMV}(h_u, f(\mathbf{y})) $$
    where $f$ is the scalar field and $C(\mathbf{x})$ is a local neighborhood of samples around $\mathbf{x}$ (typically reused from gradient estimation using central differences) [S1].
*   **Rendering:** The opacity $\alpha(\mathbf{x})$ is set to $1$ if $f(\mathbf{x}) \ge h_u$, and $\gamma(\mathbf{x})$ otherwise. The function $\gamma(\mathbf{x})$ also controls color and surface shading to distinguish focus from context. The directional occlusion model of Schott et al. is applied uniformly [S1].
*   **Source:** [S1]

### Hierarchical Cluster Tree Widget
*   **Definition:** A radial 2D node-link diagram representing the cluster hierarchy, where each node is assigned a sector of a disc proportional to the number of descendants, with an opening angle restricted to be no greater than $\pi$ to allow proper visualization of size adjustment sliders [S2, unofficial].
*   **Mechanism:**
    *   Nodes are drawn as circles with radius proportional to cluster size.
    *   Initial opacity is $0.3$ for each node [S2, unofficial].
    *   **Color Assignment:** Leaf nodes are assigned hues from the HSV color wheel based on their position in the radial layout. Inner node hues are the average of their children's hues. Saturation is determined by distance from the center; Value $V=1$ [S2, unofficial].
    *   **Interaction:** Users adjust cluster size by changing density limits using sliders. For inner nodes, an arc slider intersects edges connecting to children, defining density limits as $[d_{min}, d_{slider}]$ by default or $[d_{slider}, d_{max}]$ on click, where $d_{slider}$ is determined by linear interpolation on $[d_{min}, d_{max}]$ [S2, unofficial].
*   **Source:** [S2, unofficial]

## Background
The concept of separating focus and context in volume rendering has evolved from simple opacity modulation to more complex structural and geometric integrations. Early work focused on isosurface similarity to indicate stability [S1]. Later, hybrid approaches combined volume rendering with explicit geometry (glyphs, tensorlines) to handle dense tensor fields [S1]. Concurrently, structural approaches emerged to handle multiclassified data by modeling logical relationships between classification criteria as directed graphs, enabling boolean queries over structural relationships [S1]. The integration of interactive widgets, such as radial cluster trees, allows users to manipulate these structural parameters directly [S2, unofficial].

## Sources
[S1] Isosurface Similarity Maps, https://doi.org/10.1111/j.1467-8659.2009.01689.x, fetched 2026-10-27
[S1] Hybrid Rendering for Stress Tensor Fields, Not given in the sources, fetched 2026-10-27
[S1] Structural Focus+Context Rendering, Not given in the sources, fetched 2026-10-27
[S2] A Cluster Hierarchy-based Volume Rendering Approach for Interactive Visual Exploration of Multi-variate Volume Data, https://doi.org/10.2312/pe/vmv/vmv11/137-144, fetched 2026-10-26 (unofficial)
[S2] A Cluster Hierarchy-based Volume Rendering Approach for Interactive Visual Exploration of Multi-variate Volume Data, https://doi.org/10.2312/pe/vmv/vmv11/137-144, fetched 2026-10-26 (unofficial)
