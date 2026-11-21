# Top Level Octree Rendering

Also called: Depth-dependent Octree Traversal, Y-coordinate based Octree Cutoff

## Overview
Top Level Octree Rendering is a rendering technique that stops octree traversal at a specific depth level determined by the pixel's y-coordinate, then applies lighting [S1, unofficial]. It differs from standard recursive traversal by using the vertical screen position to limit the traversal depth, effectively culling deeper nodes based on their apparent height in the viewport [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree structure (`tree`) with a known grid length, a ray configuration (`rc`) containing camera parameters, screen dimensions (`n_y`), and a list of lights [S1, unofficial].
*   **Output:** A texture array (`texture_array`) containing the final RGB color and alpha for each pixel [S1, unofficial].

**Data Structures:**
*   `tree`: Contains `gridlength`, which defines the spatial extent of the voxel grid [S1, unofficial].
*   `rc`: Contains `n_y` (screen height in pixels), `lights` (list of light sources), and `camera` (containing `e_`, the eye position) [S1, unofficial].
*   `TreeTraverser`: A helper object initialized with the tree and a specific ray. It maintains a `stack` representing the current path in the octree [S1, unofficial].
*   `DataPoint`: Stored in leaf nodes, containing `normal` (vector) and color data [S1, unofficial].

**Parameters:**
*   `depth`: Calculated as $\log_2(\text{tree.gridlength}) + 2$ [S1, unofficial].
*   `step`: Calculated as $\text{rc.n\_y} / \text{depth}$ [S1, unofficial].

**Algorithm Steps:**
1.  Calculate `depth` and `step` using the formulas above [S1, unofficial].
2.  Initialize a parallel loop over the screen height $y$ from $0$ to $\text{rc.n\_y} - 1$ [S1, unofficial].
3.  For each $y$, iterate over the screen width $x$ from $0$ to $\text{rc.n\_x} - 1$ [G].
4.  Initialize a `TreeTraverser` instance `t` for the ray corresponding to pixel $(x, y)$ [S1, unofficial].
5.  Traverse the octree while the traverser is not terminated:
    *   Check if the current stack size is greater than or equal to the cutoff threshold: $\lfloor y / \text{step} \rfloor$ [S1, unofficial].
    *   If the condition is met, break the traversal loop [S1, unofficial].
    *   Otherwise, call `t.step()` to advance to the next node [S1, unofficial].
6.  After traversal, compute lighting for the current position `t.get_current_position()` [S1, unofficial].
7.  For each active light $i$:
    *   Compute the light vector: $\vec{to\_light} = \text{lights}[i].\text{position} - \text{current\_position}$ [S1, unofficial].
    *   Compute the half vector for specular reflection: $\vec{s} = \text{normalize}(\vec{to\_light} + (\text{camera}.e\_ - \text{current\_position}))$ [S1, unofficial].
    *   Compute distance attenuation: $\text{distancecut} = \frac{1.0}{\text{CONSTANT\_ATTENUATION} + \text{LINEAR\_ATTENUATION} \cdot \|\vec{to\_light}\| + \text{QUADRATIC\_ATTENUATION} \cdot \|\vec{to\_light}\|^2}$ [S1, unofficial].
    *   Normalize the light vector: $\vec{to\_light} = \text{normalize}(\vec{to\_light})$ [S1, unofficial].
    *   Compute diffuse factor: $\text{diffuse\_factor} = \text{data}[\text{current\_node}.\text{data}].\text{normal} \cdot \vec{to\_light}$ [S1, unofficial].
    *   Accumulate diffuse color: $r += \max(0.0, \text{diffuse\_factor}) \cdot \text{distancecut} \cdot \text{lights}[i].\text{diffuse}[0]$ (similarly for $g, b$) [S1, unofficial].
    *   Compute Phong specular factor: $\text{phong\_factor} = \text{pow}(\vec{s} \cdot \text{data}[\text{current\_node}.\text{data}].\text{normal}, \text{lights}[i].\text{SHININESS})$ [S1, unofficial].
    *   Accumulate specular color: $r += \max(0.0, \text{phong\_factor}) \cdot \text{distancecut} \cdot \text{lights}[i].\text{specular}[0]$ (similarly for $g, b$) [S1, unofficial].
8.  Write the final color to the texture array: $\text{texture\_array}[\text{index}] = \text{clamp}(255 \cdot r, 0, 255)$ [S1, unofficial].
9.  Set alpha: $\text{texture\_array}[\text{index}+3] = 1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    return [c / math.sqrt(sum(c*c for c in v)) for c in v]
def length(v):
    return math.sqrt(sum(c*c for c in v))
def dot(a, b):
    return sum(x*y for x, y in zip(a, b))
def clamp(val, lo, hi):
    return max(lo, min(hi, val))

class TreeTraverser:
    def __init__(self, tree, ray):
        self.tree = tree
        self.ray = ray
        self.stack = []
        self.current_node = tree.root
        self.current_pos = ray.origin
        self.terminated = False

    def is_terminated(self):
        return self.terminated

    def step(self):
        # Standard octree traversal logic not detailed in sources [G]
        pass

    def get_current_position(self):
        return self.current_pos

    def get_current_node(self):
        return self.current_node

def top_level_octree_rendering(tree, rc, texture_array, n_x):
    depth = math.log2(tree.gridlength) + 2
    step = rc.n_y / depth

    for y in range(rc.n_y):
        for x in range(n_x):
            ray = rc.get_ray_for_pixel(x, y)
            t = TreeTraverser(tree, ray)

            while not t.is_terminated():
                if len(t.stack) >= (y / step):
                    break
                t.step()

            current_pos = t.get_current_position()
            current_node = t.get_current_node()
            data = current_node.data

            r, g, b = 0.0, 0.0, 0.0

            for light in rc.lights:
                to_light = [light.position[j] - current_pos[j] for j in range(3)]
                view_dir = [rc.camera.e_[j] - current_pos[j] for j in range(3)]
                s = normalize([to_light[j] + view_dir[j] for j in range(3)])

                dist = length(to_light)
                dist_sq = dist * dist
                distancecut = 1.0 / (light.CONSTANT_ATTENUATION + 
                                     light.LINEAR_ATTENUATION * dist + 
                                     light.QUADRATIC_ATTENUATION * dist_sq)

                to_light = normalize(to_light)
                normal = data.normal
                diffuse_factor = dot(normal, to_light)

                r += max(0.0, diffuse_factor) * distancecut * light.diffuse[0]
                g += max(0.0, diffuse_factor) * distancecut * light.diffuse[1]
                b += max(0.0, diffuse_factor) * distancecut * light.diffuse[2]

                phong_factor = pow(max(0.0, dot(s, normal)), light.SHININESS)

                r += max(0.0, phong_factor) * distancecut * light.specular[0]
                g += max(0.0, phong_factor) * distancecut * light.specular[1]
                b += max(0.0, phong_factor) * distancecut * light.specular[2]

            index = (y * n_x + x) * 4
            texture_array[index] = clamp(255 * r, 0, 255)
            texture_array[index + 1] = clamp(255 * g, 0, 255)
            texture_array[index + 2] = clamp(255 * b, 0, 255)
            texture_array[index + 3] = 1
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The formula $\text{depth} = \log_2(\text{tree.gridlength}) + 2$ ensures `depth` is at least $2$ for any valid `gridlength >= 1`, preventing division by zero during `step` calculation [S1, unofficial].
*   **Type Comparison:** The cutoff condition `len(t.stack) >= (y / step)` compares an integer stack size to a floating-point quotient, matching the source's C++ implementation where `stack.size()` (unsigned int) is compared against `y / step` (float) [S1, unofficial].
*   **Boundary Conditions:** The loop bounds for $x$ and $y$ must match the screen resolution provided by `rc.n_y` and `rc.n_x` [S1, unofficial].

## Subtopics

### [Fixed-Level Octree Traversal](5000-fixed-level-octree-traversal.md)
Stops traversal at a fixed maxlevel depth regardless of pixel position. Choose when uniform detail is required across the entire image.

### [Recursive Octree Traversal with Face Processing](2997-recursive-octree-traversal-with-face-processing.md)
Processes faces of octree nodes during traversal. Choose when face-based shading or culling is needed.

## Operating characteristics
Not given in the sources.

## Background
This technique is described in the `cpu_voxel_raycaster` project by Forceflow [S1, unofficial]. It appears to be a custom optimization for CPU-based voxel raycasting, leveraging the parallelization capabilities of OpenMP [S1, unofficial]. The method is not a standard academic algorithm but rather a practical implementation detail from a specific codebase [S1, unofficial].

## Sources
[S1] cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
