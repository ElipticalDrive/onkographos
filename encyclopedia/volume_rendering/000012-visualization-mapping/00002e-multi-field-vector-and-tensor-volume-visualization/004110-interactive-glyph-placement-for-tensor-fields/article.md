# Interactive Glyph Placement for Tensor Fields

Also called: Glyph Packing, Glyph Packing with Delaunay Triangulation, Glyph Packing with Delaunay Acceleration, Hlawitschka et al. Glyph Placement

## Overview
Interactive Glyph Placement for Tensor Fields is an iterative particle-system algorithm that densely distributes tensor glyphs throughout a volume or slice to minimize overlap while preserving local tensor information [S1, S2]. It solves the problem of visual clutter and regular-grid artifacts in tensor visualization by using a force-based repulsion model accelerated by a Delaunay triangulation [S1]. Unlike previous methods that require manual tuning of bin sizes or force parameters, this approach uses a parameter-less acceleration structure and intuitive visual parameters (glyph scaling and inter-glyph distance) [S1]. It is specifically designed for interactive exploration, allowing planes to move through a volume without recomputing the acceleration structure [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Discrete tensor field data $D$ (e.g., Diffusion Tensor MRI) [S1].
*   Confidence mask $M$ indicating valid tensor regions [S2].
*   Glyph scaling parameter $\alpha$ (transforms tensor units to image length units) [S2].
*   Force width parameter $\gamma$ (defines the attractive interval, typically 0.5) [S2].
*   Drag coefficient $C_{drag}$ and mask coefficient $C_{mask}$ [S2].
*   Optional: Target number of glyphs or area selection [S1].

**Outputs:**
*   A set of particle positions $\{p_i\}$ representing glyph centers [S1].
*   A Delaunay triangulation of these positions for neighbor search [S1].

### Data Structures
1.  **Particle List:** Array of particles, each storing position $p \in \mathbb{R}^3$, velocity $v \in \mathbb{R}^3$, and tensor reference [S1, S2].
2.  **Spatial Bins (Optional/Alternative):** Dynamic lists of particles in a grid [S2]. Bin edge length $L_{bin} = 2\alpha\lambda_{max}(1 + \gamma)$, where $\lambda_{max}$ is the maximal eigenvalue of the tensor field [S2]. This ensures all interacting particles are in the same or immediately neighboring bins [S2].
3.  **Delaunay Triangulation:** A triangulation of the particle positions [S1]. Used for neighbor search via breadth-first search (BFS) on the triangulation graph [S1]. Direct edge neighbors are sufficient for convergence; neighbors with an edge distance of two are included for stability against triangulation flips [S1].
4.  **Interpolated Tensor Field:** Continuous function $D(p)$ obtained via cubic spline interpolation (C2 field) using pre-filtering and convolution-based separable reconstruction with uniform cubic B-splines (local four-sample support along each axis) [S2].

### Initialization
1.  Traverse the dataset to find the maximal determinant $\det_{max}$ and maximal eigenvalue $\lambda_{max}$ [S2].
2.  Determine initial particle count and distribution based on glyph size relative to cell size [S1]:
    *   **Sparse Glyphs (<1 per cell):**
        1.  Calculate cell density ratio $p[v] = A[v]/a[v]$ for each cell $v$, where $A[v]$ is cell area and $a[v]$ is estimated glyph area [S1].
        2.  Integrate density: $p[v] = p[v-1] + p[v]$ for $v=1 \dots N_{cells}$ [S1].
        3.  Normalize: $p[v] = p[v]/P$, where $P = p[N_{cells}-1]$ [S1].
        4.  Place $P$ random glyphs: For each glyph, generate $d \in [0, 1)$ [S1]. Binary search for cell $i$ such that $p[i-1] \le d < p[i]$ (with $p[0]=0$) [S1]. Place glyph randomly within cell $i$ [S1].
    *   **Dense Glyphs (>1 per cell):**
        1.  Determine glyphs per voxel as $\text{round}(p[v])$ [S1].
        2.  Fill each voxel with a uniform distribution of this amount using pre-computed strata (e.g., Poisson-disc) or interleaved sampling via hash map using cell index [S1].
3.  **Rejection Sampling (S2 variant):** Initialize particles at random locations [S2]. Reject candidates outside confidence mask $M$ [S2]. Reject candidate $p$ with probability $\det(D(p))/\det_{max}$ to approximate correct density [S2].

### Force Model
The motion of particles is governed by the second-order ordinary differential equation with unit mass [S2]:
$$ \sum_{b \neq a} f_{ab} - C_{drag} \frac{dp_a}{dt} + C_{mask} \nabla M(p_a) = \frac{d^2 p_a}{dt^2} $$
where $f_{ab}$ is the inter-particle force, $C_{drag} \frac{dp_a}{dt}$ is the drag force, and $C_{mask} \nabla M(p_a)$ is the mask force [S2].

**1. Inter-particle Force ($f_{ab}$):**
The force is derived from an energy function $E_{ab}$ [S2]:
$$ E_{ab} = \phi(|x_{ab}|) \cdot g(D_{ab}^{-1}, y_{ab}) $$
where $x_{ab} = p_a - p_b$ (or $y_{ab}$ in S1 notation, vector from b to a), and $g$ is the glyph surface function [S2]. For ellipsoidal glyphs, $g(y) = \|D^{-1} y\|$ [S1]. The factor of 2 in the denominator of $g$ (in S2's specific formulation) ensures mutually tangent glyphs map to spheres of radius 1/2 with unit distance between centers [S2].

The force is the negative gradient of energy with respect to $y_{ab}$ [S2]:
$$ f_{ab} = - \frac{\partial E_{ab}}{\partial y_{ab}} $$
Using the chain rule, the full derivative involves a third-order tensor gradient $\frac{d D^{-1}}{dy}$ [S2]:
$$ \frac{\partial E}{\partial y_i} = \frac{\phi'(|x|)}{2\alpha|x|} \sum_j \sum_k \left( D^{-1}_{ij} x_j + \sum_k D^{-1}_{ijk} x_j y_k \right) $$
**Mitigation:** To avoid the complexity of the third-order tensor gradient, the tensor field is treated as locally constant [S2]. The force simplifies to [S2]:
$$ f_{ab} = - \frac{\phi'(|x_{ab}|)}{2\alpha|x_{ab}|} D_{ab}^{-1} x_{ab} $$
where $D_{ab}^{-1}$ is the inverse tensor at the midpoint [S2]. A fast approximation for early iterations is $D_{ab}^{-1} \approx \frac{D^{-1}(p_a) + D^{-1}(p_b)}{2}$ [S2].

**2. Potential Energy Function $\phi(r)$:**
The function $\phi$ is based on Hooke's law with a tunable width parameter $\gamma$ to ensure $C^1$ continuity [S2]. The derivative $\phi'(r)$ is defined piecewise [S1, S2]:
$$ \phi'(r) = \begin{cases} 0 & 0 < r < 1 \\ (r - 1)(1 + \gamma - r)^2 / \gamma^2 & 1 \le r \le 1 + \gamma \\ 0 & r > 1 + \gamma \end{cases} $$
The optimum distance is $r=1$ [S1]. Glyphs farther than $r = 1 + \gamma$ do not influence each other [S1]. Typical value for $\gamma$ is 0.5 [S2].

**3. Mask Force ($f_{mask}$):**
Pushes particles towards the interior of the confidence mask $M$ [S2]:
$$ f_{mask} = C_{mask} \nabla M(p) $$

### Integration and Update
1.  **Euler Integration:** Update particle positions using a small time step (step size 0.2 is empirically appropriate) [S1]. The goal is minimum-energy state, not physical path accuracy [S2].
2.  **2D Projection:** If generating 2D slices, compute 3D forces but project particle positions back onto the slice plane at every iteration [S1, S2].
3.  **Acceleration Structure Update:** Recompute the Delaunay triangulation after moving points [S1].

### Addition and Removal of Glyphs
If the number of glyphs is allowed to change [S1]:
1.  **Addition:** Find triangles in the triangulation with at least two long edges (indicating large distances between neighbors) [S1]. Place new glyphs in these cells [S1].
2.  **Removal:** Mark glyphs for removal that have high repelling forces on all their edges [S1]. Remove a random subset of these vertices to avoid removing all glyphs at once in dense areas [S1].

### Interactive Group Movement
1.  Place a set of glyphs around a pointing device (cursor) [S1].
2.  Initially place glyphs randomly around the cursor and optimize the grid while displayed [S1].
3.  When the cursor moves, move all points the same way [S1].
4.  Fix one glyph to the cursor to prevent diffusion [S1].
5.  Apply additional small forces pointing to the cursor to prevent asymmetry [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

# Parameters
ALPHA = 1.0
GAMMA = 0.5
C_DRAG = 1.0
C_MASK = 1.0
STEP_SIZE = 0.2
ALLOW_GLYPH_CHANGES = False
MODE_2D_SLICE = True
SLICE_Z = 0.0

class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vec3(self.x+other.x, self.y+other.y, self.z+other.z)
    def __sub__(self, other): return Vec3(self.x-other.x, self.y-other.y, self.z-other.z)
    def __mul__(self, s): return Vec3(self.x*s, self.y*s, self.z*s)
    def __truediv__(self, s): return Vec3(self.x/s, self.y/s, self.z/s)
    def length(self): return math.sqrt(self.x**2 + self.y**2 + self.z**2)
    def dot(self, other): return self.x*other.x + self.y*other.y + self.z*other.z

class Particle:
    def __init__(self, pos):
        self.pos = pos
        self.vel = Vec3(0,0,0)
        self.force = Vec3(0,0,0)
        self.neighbors = []

def sample_tensor_inverse(p):
    # Cubic spline interpolation of pre-filtered tensor field [S2]
    return [[1.0, 0.0, 0.0], [0.0, 1.0, 0.0], [0.0, 0.0, 1.0]] # Placeholder for actual interpolation [G]

def sample_mask_gradient(p):
    # Gradient of confidence mask M [S2]
    return Vec3(0.0, 0.0, 0.0) # Placeholder for actual gradient [G]

def mat_vec_mul(M, v):
    return Vec3(M[0][0]*v.x + M[0][1]*v.y + M[0][2]*v.z,
                M[1][0]*v.x + M[1][1]*v.y + M[1][2]*v.z,
                M[2][0]*v.x + M[2][1]*v.y + M[2][2]*v.z)

def phi_prime(r):
    if r < 1: return 0.0
    if r <= 1 + GAMMA: return (r - 1) * (1 + GAMMA - r)**2 / GAMMA**2
    return 0.0

def compute_force(p_a, neighbors):
    f_total = Vec3(0,0,0)
    for p_b in neighbors:
        y = p_a.pos - p_b.pos
        r = y.length()
        if r > 1 + GAMMA or r < 1e-6: continue
        fp = phi_prime(r)
        D_inv_a = sample_tensor_inverse(p_a.pos)
        D_inv_b = sample_tensor_inverse(p_b.pos)
        D_inv_mid = [[(D_inv_a[i][j] + D_inv_b[i][j])/2.0 for j in range(3)] for i in range(3)]
        D_inv_mid_y = mat_vec_mul(D_inv_mid, y)
        force_mag = - (fp / (2 * ALPHA * r))
        f_inter = D_inv_mid_y * force_mag
        f_total = f_total + f_inter
    grad_M = sample_mask_gradient(p_a.pos)
    f_mask = grad_M * C_MASK
    f_total = f_total + f_mask
    f_drag = p_a.vel * (-C_DRAG)
    f_total = f_total + f_drag
    return f_total

def initialize_particles():
    particles = []
    # S1/S2 initialization logic: traverse data, find det_max, lambda_max, 
    # apply sparse/dense or rejection sampling rules [S1, S2]
    return particles

class DelaunayTriangulation:
    def __init__(self, particles):
        self.particles = particles
        self.adj = {id(p): [] for p in particles}
        # Build adjacency based on proximity or triangulation [S1]
        for i, p1 in enumerate(particles):
            for j, p2 in enumerate(particles):
                if i != j and p1.pos.length() < 2.0: # Simplified neighbor check [G]
                    self.adj[id(p1)].append(p2)
    def get_neighbors(self, p, max_edge_dist=2):
        # BFS on triangulation graph up to max_edge_dist [S1]
        visited = set()
        queue = [(p, 0)]
        visited.add(id(p))
        neighbors = []
        while queue:
            curr, dist = queue.pop(0)
            if dist > 0: neighbors.append(curr)
            if dist >= max_edge_dist: continue
            for nb in self.adj[id(curr)]:
                if id(nb) not in visited:
                    visited.add(id(nb))
                    queue.append((nb, dist+1))
        return neighbors

def check_convergence(particles):
    # Check if energy change or max step reached [S1]
    return False

def remove_random_subset(to_remove, subset_size=10):
    if len(to_remove) > subset_size:
        random.shuffle(to_remove)
        for p in to_remove[subset_size:]:
            particles.remove(p)

def main_loop():
    global particles
    particles = initialize_particles()
    delaunay = DelaunayTriangulation(particles)
    converged = False
    while not converged:
        for p in particles:
            p.neighbors = delaunay.get_neighbors(p, max_edge_dist=2)
            p.force = compute_force(p, p.neighbors)
        for p in particles:
            p.vel = p.vel + p.force * STEP_SIZE
            p.pos = p.pos + p.vel * STEP_SIZE
        if MODE_2D_SLICE:
            for p in particles:
                p.pos.z = SLICE_Z
        delaunay = DelaunayTriangulation(particles)
        if ALLOW_GLYPH_CHANGES:
            triangles = delaunay.get_triangles()
            for tri in triangles:
                if len([e for e in tri.edges if e.length > 1.5]) >= 2:
                    particles.append(Particle(tri.center))
            to_remove = [p for p in particles if all(nb.force.dot(p.force) > 0 for nb in p.neighbors)]
            remove_random_subset(to_remove)
        converged = check_convergence(particles)
```

## Operating characteristics

**Time and Memory Complexity:**
*   **Force Computation:** $O(N)$ per iteration assuming constant-time neighbor lookup via spatial binning or Delaunay adjacency [S1].
*   **Memory:** $O(N)$ for particles and triangulation [S1].
*   **Parallelization:** Spatial bins can be assigned to worker threads on a first-come-first-served basis in a multi-threaded implementation [S2].

**Measured Performance:**
*   **Hlawitschka et al. (S1):** Up to 8000 glyphs optimized interactively on a desktop PC [S1]. Complete slices take about one minute for first results, up to four minutes for stability [S1]. Convergence is fast: almost stable after about 15 steps, stable after several hundred steps [S1].
*   **Kindlmann et al. (S2):** 4000 iterations took about 1.5 hours on a dual processor 2.0 GHz PowerPC G5 for 3D packing [S2]. 2D packing (2000 iterations) took about 8 minutes on a single processor 1.67 GHz PowerPC G4 [S2].

**Strengths and Limitations:**
*   **Strengths:** Removes visual emphasis of regular sampling grids; preserves continuous features; interactive frame rates for exploration; parameter-less acceleration structure (Delaunay) avoids bin size tuning; supports arbitrary glyph shapes via surface function $g(y)$ [S1, S2].
*   **Limitations:** Tensor inverse and interpolation at midpoints are expensive (mitigated by approximation in early iterations) [S2]; 2D projection artifacts may occur as 3D forces do not always separate 3D glyphs cleanly when viewed from above [S2]; holes in initial seeding can remain stable if forces are weak across large distances (mitigated by improved initial distribution) [S1].

## Background

**Origin:**
The method builds upon "Glyph Packing" introduced by Kindlmann and Westin [S1, S2]. Hlawitschka, Scheuermann, and Hamann presented the interactive variant with Delaunay acceleration in 2007 [S1]. Kindlmann, Grimson, and Enquobahrie presented the foundational particle system approach in 2006 [S2].

**History and Development:**
Glyph-based visualization has a long history in medical imaging [S1]. The initial glyph packing method by Kindlmann and Westin used a regular grid and force-based repulsion but required manual parameter tuning and suffered from visual artifacts due to regular sampling [S1, S2]. Hlawitschka et al. improved this by introducing a Delaunay triangulation for neighbor search, which does not require bin size selection and allows for interactive plane movement without recomputation [S1]. Kindlmann et al. further refined the physics by introducing an intrinsic scale based on Hooke's law and using cubic spline interpolation for the tensor field [S2].

**Usage:**
Used for interactive visualization of tensor fields in medical imaging, specifically Diffusion Tensor Imaging (DTI) [S1]. Applications include neurosurgery planning for tumor resection, assessing tumor extent, and planning minimally damaging approaches through brain white matter [S2]. It is used in visualization systems for interactive probes to inspect areas of interest in medical data [S1].

## Sources

[S1] Interactive Glyph Placement for Tensor Fields, https://doi.org/10.1007/978-3-540-76858-6_33, fetched 2026-10-27
[S2] Diffusion tensor visualization with glyph packing, http://lmi.bwh.harvard.edu/papers/pdfs/2006/KindlmannTVCG2006.pdf, fetched 2026-10-29
