# Differentiable Rasterization via Ray Casting

Also called: CUDA rasterize kernel, voxel2mesh rasterization

## Overview
This method converts a triangle mesh into a 3D voxel volume by casting random rays from voxel centers to the top plane of the bounding box, enabling gradient-based optimization of mesh geometry from voxel supervision [S1, unofficial]. It differs from standard rasterization by providing a backward pass that computes gradients with respect to vertex positions, allowing the mesh to be refined to match a target voxel occupancy [S1, unofficial]. Unlike [Backward Differentiable Monte Carlo Voxelization](1997-backward-differentiable-monte-carlo-voxelization.md), which relies on Monte Carlo sampling for the backward pass, this method uses a deterministic ray-casting forward pass with a specialized geometric gradient approximation for the backward pass [S1, unofficial].

## How it works
**Inputs:**
- Mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$ (indices into $V$).
- Voxel grid dimensions $W, H, D$ (width, height, depth).
- Bounding box depth $D$ (used for gradient scaling).

**Outputs:**
- Voxel occupancy volume $Vol \in \{0, 1\}^{D \times H \times W}$.
- Gradients $\frac{\partial Loss}{\partial V}$ with respect to vertex positions.

**Data Structures:**
- `Vol[z][y][x]`: Boolean or integer (0/1) indicating occupancy.
- `grad_V`: Tensor of shape $(N, 3)$ storing gradients for each vertex.

**Parameters:**
- `diag`: Scaling factor $1.732 \cdot D$, approximating the diagonal of the voxel grid [S1, unofficial].
- `border_intersection`: Counter for ambiguous ray intersections.

**Steps:**

1. **Forward Pass (Occupancy):**
   For each voxel at coordinates $(x, y, z)$:
   1. Initialize a random number generator state with seed $S = W \cdot H \cdot z + W \cdot y + x$ [S1, unofficial].
   2. Generate a random point $P_1$ on the top plane ($z=D$) with coordinates:
      $$ P_1.x = W \cdot \text{curand\_uniform\_double}(S) $$
      $$ P_1.y = H \cdot \text{curand\_uniform\_double}(S) $$
      $$ P_1.z = D $$
   3. Set ray origin $P_0 = (x, y, z)$ [S1, unofficial].
   4. Initialize counters: `face_intersection = 0`, `border_intersection = 0` [S1, unofficial].
   5. For each face with vertices $V_0, V_1, V_2$:
      a. Compute edge vectors $u = V_1 - V_0$, $v = V_2 - V_0$ [S1, unofficial].
      b. Compute face normal $n = \text{cross}(u, v)$ [S1, unofficial].
      c. Compute ray direction $dir = P_1 - P_0$ and vector $w_0 = P_0 - V_0$ [S1, unofficial].
      d. Calculate $a = -\text{dot}(n, w_0)$ and $b = \text{dot}(n, dir)$ [S1, unofficial].
      e. If $|b| < 10^{-8}$, skip (ray parallel to face) [S1, unofficial].
      f. Calculate intersection distance $r = a / b$ [S1, unofficial].
      g. If $r < 0$, skip (intersection behind ray origin) [S1, unofficial].
      h. Compute intersection point $I = P_0 + r \cdot dir$ [S1, unofficial].
      i. Check if $I$ is inside triangle $V_0 V_1 V_2$ using barycentric coordinates:
         - $uu = \text{dot}(u, u)$, $vv = \text{dot}(v, v)$, $uv = \text{dot}(u, v)$ [S1, unofficial].
         - $w = I - V_0$ [S1, unofficial].
         - $wu = \text{dot}(w, u)$, $wv = \text{dot}(w, v)$ [S1, unofficial].
         - $d = uv^2 - uu \cdot vv$ [S1, unofficial].
         - $s = \frac{uv \cdot wv - vv \cdot wu}{d}$ [S1, unofficial].
         - $t = \frac{uv \cdot wu - uu \cdot wv}{d}$ [S1, unofficial].
      j. If $s < 0$ or $s > 1$ or $t < 0$ or $s + t > 1$, skip [S1, unofficial].
      k. If $s == 0$ or $t == 0$ or $s + t == 1$, increment `border_intersection` [S1, unofficial].
      l. Otherwise, increment `face_intersection` [S1, unofficial].
   6. If `border_intersection > 0`, restart the ray for this voxel (go to step 1.2) [S1, unofficial].
   7. If `face_intersection` is odd, set $Vol[z][y][x] = 1$ (inside); else set $0$ (outside) [S1, unofficial].

2. **Backward Pass (Gradients):**
   For each voxel $(x, y, z)$:
   1. Retrieve $voxel = Vol[z][y][x]$ and $grad\_voxel = \frac{\partial Loss}{\partial Vol[z][y][x]}$ [S1, unofficial].
   2. For each face with vertices $V_0, V_1, V_2$:
      a. For each vertex $V_k$ in $\{V_0, V_1, V_2\}$, compute gradient contribution using `compute_grad` [S1, unofficial].
      b. In `compute_grad` for vertex $V_2$ (relative to ray origin $P$ and other vertices $V_0, V_1$):
         i. Define plane through $P, V_0, V_1$. Compute normal $n = \text{cross}(V_0 - P, V_1 - P)$ [S1, unofficial].
         ii. For each axis $axis \in \{x, y, z\}$:
             - Calculate potential shifted position $V_2^{shifted}$ by intersecting the plane with a line parallel to the axis passing through $V_2$.
             - Example for $x$-axis shift:
               $$ V_2^{shifted}_x = -\frac{n_y V_2.y + n_z V_2.z}{n_x} $$
               $$ V_2^{shifted}_y = V_2.y $$
               $$ V_2^{shifted}_z = V_2.z $$
               (Similar formulas apply for $y$ and $z$ shifts, swapping indices appropriately) [S1, unofficial].
             - Check if $V_2^{shifted}$ lies on the face triangle $V_0, V_1, V_2^{shifted}$ using `P_is_on_face` [S1, unofficial].
               - Solve $a(V_0 - V_2^{shifted}) + b(V_1 - V_2^{shifted}) = P - V_2^{shifted}$ for coefficients $a, b$ [S1, unofficial].
               - If $n_{axis} \neq 0$ and the shift is non-zero and `P_is_on_face` returns true:
                 - Update gradient:
                   $$ \text{grad}_{V2}[axis] = -\text{grad}_{\text{voxel}} \cdot \frac{|V_2^{shifted} - V_2|}{\text{diag}} $$
                   if the sign of the shift relative to the voxel boundary matches the gradient direction (i.e., $d \cdot \text{grad}_{\text{voxel}} < 0$ for outside voxels, $> 0$ for inside), where $d = V_2^{shifted} - V_2$ [S1, unofficial].
             - Else, gradient contribution for this axis is 0 [S1, unofficial].

**Pitfalls:**
- **Border Intersections:** If the ray intersects any triangle edge or vertex, the `border_intersection` counter increments. If this counter is $> 0$ after checking all faces, the voxel result is discarded and the ray is re-cast to avoid ambiguity [S1, unofficial].
- **Precision:** The threshold $10^{-8}$ for parallel rays is critical; too small may cause division by zero, too large may miss valid intersections [S1, unofficial].
- **Gradient Scaling:** The `diag` factor $1.732 \cdot D$ approximates the grid diagonal; incorrect scaling may lead to unstable gradients [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot D \cdot M)$ per forward pass, where $M$ is the number of faces. The backward pass is also $O(W \cdot H \cdot D \cdot M)$ due to per-voxel gradient computation [S1, unofficial].
- **Memory Complexity:** $O(W \cdot H \cdot D)$ for the voxel volume and $O(N)$ for vertex gradients [S1, unofficial].
- **Performance:** Performance depends heavily on the number of faces and voxel resolution. The random ray casting allows for parallelization on GPUs [S1, unofficial].
- **Strengths:** Provides differentiable mesh optimization from voxel supervision; handles complex geometries via ray casting [S1, unofficial].
- **Limitations:** Stochastic nature requires multiple samples or careful seed management for reproducibility; border intersection handling adds complexity [S1, unofficial].
- **When to use:** When optimizing mesh geometry to match a target voxel volume is required [S1, unofficial].
- **When not to use:** When real-time rendering without gradients is needed; when exact analytical gradients are preferred over approximations [S1, unofficial].

## Background
This method is part of the Voxel2Mesh framework, which aims to generate meshes from voxel data using deep learning [S1, unofficial]. It was introduced in the context of differentiable rendering for 3D shape optimization [S1, unofficial]. The technique leverages CUDA for efficient parallel ray casting and gradient computation [S1, unofficial].

## Sources
[S1] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
