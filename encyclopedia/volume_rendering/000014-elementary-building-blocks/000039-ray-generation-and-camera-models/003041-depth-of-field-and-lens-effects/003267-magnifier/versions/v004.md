# Magnifier

Also called: Magnification lens, Magnifier volume renderings, Virtual Lens Optics, Volumetric lenses

## Overview
The Magnifier is a focus+context technique for volume rendering that magnifies regions of interest by modifying ray directions to pass through a virtual focal point, while using a transition region to compress context and prevent loss of spatial information [S2]. Unlike the sibling [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md), which blurs out-of-focus areas, the Magnifier geometrically distorts the surrounding view to preserve spatial relationships [S1, book][S2]. It differs from the [Lens Shift](1851-lens-shift.md) sibling by actively changing ray convergence to create a magnification effect [S1, book].

## How it works
The method operates within a ray-casting rendering framework [S2]. It modifies the direction of rays originating from the image plane to create a "ray cone" that converges at a virtual focal point, thereby magnifying the volume data intersected by these rays [S2].

### Inputs and Outputs
*   **Inputs:**
    *   Volume dataset $V$.
    *   Image plane points $PI$ (ray origins) and $PR$ (screen points) [S2].
    *   Lens parameters: Center $LC$, Radius $lr$, Transition Width $lb$, Virtual Focal Point $F$ [S2].
    *   Camera parameters: Orthogonal rays for the base model [S1, book].
*   **Outputs:**
    *   Rendered image with magnified focal region and compressed peripheral region [S2].

### Data Structures
The method relies on the following geometric parameters, typically stored as 3D vectors or scalars:
*   $LC \in \mathbb{R}^3$: Center point of the lens on the image plane [S2].
*   $lr \in \mathbb{R}$: Radius of the lens [S2].
*   $lb \in \mathbb{R}$: Width of the transition region [S2].
*   $F \in \mathbb{R}^3$: Virtual focal point. The source specifies $F$ is parallel to the image plane [S2].
*   $PI \in \mathbb{R}^3$: Point of origin on the image plane for a specific ray [S2].
*   $PF \in \mathbb{R}^3$: Point on the virtual lens focus plane where the ray passes through the focal plane [S2].

### Steps
1.  **Initialize Lens:** Position the lens on the image plane with center $LC$ and radius $lr$. Define the virtual focal point $F$. The position of $F$ relative to the image plane controls the magnification factor; moving $F$ closer to the image plane increases magnification [S2].
2.  **Determine Region:** For each pixel/point $PI$ on the image plane, calculate the distance $d = |PI - LC|$.
    *   If $d < lr - lb$: The point is in the **magnification region**.
    *   If $lr - lb \le d \le lr$: The point is in the **transition region**.
    *   If $d > lr$: The point is in the **peripheral/normal region**.
3.  **Compute Ray Direction:**
    *   **Magnification Region:** Modify the ray direction so it passes through the virtual focal point $F$. The ray forms a cone between the lens and $F$. The direction is effectively determined by the vector from $PI$ to $F$ [S1, book][S2].
    *   **Transition Region:** Compute the ray direction by interpolating between the focused direction (towards $F$) and the orthogonal direction (normal to the image plane). This is achieved by calculating an intermediate point $PF$ on the virtual focus plane [S2].
        1.  Calculate the magnification factor $mf$ for the point $PR$ (see Formula below).
        2.  Compute $PF$ using the interpolation formula derived from the geometric constraint that the ray passes through the focus plane [S2]:
            $$ PF = F + \frac{PI - LC}{|PI - LC|} \cdot |PF - F| $$
            The distance $|PF - F|$ is determined by the linear interpolation of the ray's path from the edge of the magnification region ($lr-lb$) to the edge of the lens ($lr$) [S2].
        3.  Calculate the final ray direction:
            $$ \text{ray dir} = PF - PI \quad [S2] $$
    *   **Peripheral Region:** Rays remain orthogonal to the image plane (normal magnification) [S1, book].
4.  **Ray Casting:** Cast rays with the computed directions through the volume dataset $V$ [S2].
5.  **Anti-Aliasing:** Apply volume texture mip-mapping to handle aliasing in the transition region, where the ray sampling rate is relatively low [S2]. Calculate the required mip-map level based on the magnification factor $mf$ [S2].

### Formulas
**1. Ray Direction in Transition Region:**
The source provides the following relationship for the geometry of the transition region [S2]:
$$ \frac{|PF - F|}{|PI - LC| - (lr - lb)} = \frac{|PF - F|}{lb} $$
The explicit formula for the final ray direction is [S2]:
$$ \text{ray dir} = PF - PI $$
Where $PF$ is calculated as [S2]:
$$ PF = F + \frac{PI - LC}{|PI - LC|} \cdot |PF - F| $$
The magnitude $|PF - F|$ must be derived from the linear interpolation of the ray's intersection with the focal plane. In the magnification region ($d < lr-lb$), the ray goes directly to $F$. In the transition region, the ray goes to a point $PF$ on the focal plane. The distance $|PF - F|$ scales linearly with the distance from the inner boundary of the transition region [S2].

**2. Magnification Factor for Anti-Aliasing:**
To determine the appropriate mip-map level, calculate the magnification factor $mf$ for a point $PR$ (referred to as $PR$ in the source text for this specific formula) [S2]:
$$ mf = \left( \frac{|PR - PRI|}{|F - LC|} \cdot \frac{lb}{lr} \right) + 1 $$
where $PRI$ is the orthogonal projection of $PR$ on the image plane [S2].

## Subtopics

### [Sampling-rate-based lens](6758-sampling-rate-based-lens.md)
The sampling-rate-based lens modifies the spatial density of rays cast into a volume to achieve focus+context magnification without altering ray directions. Unlike the parent [Magnifier](6754-magnifier.md), which changes ray directions to pass through a virtual focal point, this method keeps rays parallel but varies their start positions on the image plane to create dense sampling in the center and coarse sampling at the boundary. It differs from the sibling [Feature-based lens](7436-feature-based-lens.md) by relying on a user-defined or arbitrary sampling function rather than feature detection to determine ray distribution.

### [Feature-based lens](7436-feature-based-lens.md)
The feature-based lens is a focus+context technique for volume rendering that dynamically defines the shape of the magnification region based on the geometry of specific features identified via segmentation, rather than a fixed geometric primitive. It solves the problem of highlighting semantically important structures (e.g., bones, tumors) by dedicating more screen area to them while compressing surrounding context to preserve spatial relationships. Unlike the [Sampling-rate-based lens](6758-sampling-rate-based-lens.md), which varies ray density, the feature-based lens modifies ray directions to pass through a virtual focal point, but restricts this modification to rays intersecting the segmented feature and its immediate transition boundary.

## Operating characteristics
Not given in the sources.

## Background
The Magnifier is part of a framework for illustrative focus+context approaches in interactive volume visualization, introduced by S. Bruckner, M. E. Gröller, K. Mueller, B. Preim, and D. Silver [S1, book]. It is based on geometric optics and conforms to sampling theory [S2]. Unlike physical optics which are hard to fabricate, this method provides volumetric lenses in software [S1, book].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
