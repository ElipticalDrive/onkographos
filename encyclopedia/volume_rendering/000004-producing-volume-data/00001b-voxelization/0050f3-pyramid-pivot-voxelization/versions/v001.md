# Pyramid Pivot Voxelization

Also called: Hierarchical Pivot Sampling, P2Voxel

## Overview
Pyramid Pivot Voxelization is an adaptive voxelization strategy that exploits spatial non-uniformity by allocating finer pivot tokens to geometrically complex regions while keeping smooth regions coarse and compact [S1, unofficial]. It builds on [Voxelization](12-voxelization.md) by introducing a block-wise pyramid sampling map that assigns variable resolutions to macro-blocks based on a complexity score, balancing reconstruction fidelity and token efficiency [S1, unofficial]. Unlike fixed-resolution methods, it reserves fine-scale sampling for high-frequency details (high curvature/sharp structures) while using coarser blocks for smooth areas, improving Pareto efficiency over SDF-based and Dual Contouring baselines [S1, unofficial].

## How it works
**Inputs:**
- Input mesh (triangle soup or watertight mesh).
- Normalized bounding box: $[-0.5, 0.5]^3$ [S1, unofficial].
- Macro-block grid dimension $B$ (integer).
- Predefined resolution set $R = \{r_{min}, \dots, r_{max}\}$ [S1, unofficial].
- Hyperparameters: $\lambda_{var} = 0.5$, $\lambda_{grad} = 0.3$, $\lambda_n = 3.0$ [S1, unofficial].

**Outputs:**
- Pyramid pivot token set $T_{pyr}$ [S1, unofficial].
- Reconstructed sparse watertight mesh $\hat{M}$ (via Sparse Marching Cubes) [S1, unofficial].

**Data Structures:**
1. **Macro-block Grid $B$**: A 3D grid of size $B \times B \times B$. Each cell $g$ represents a spatial region of size $1/B$.
2. **Allocation Map $M$**: A mapping $M: g \mapsto r \in R$. Stores the assigned resolution for each active block. Missing active blocks are filled with $r_{min}$ [S1, unofficial].
3. **Pivot Token $\tau(r)$**: A tuple $\{p(r), s(r)\}$ where:
   - $p(r) \in \mathbb{R}^3$: Pivot point (closest point on surface to voxel center or similar local evidence) [S1, unofficial].
   - $s(r) \in \{-1, +1\}$: Orientation sign [S1, unofficial].
4. **Pyramid Set $T_{pyr}$**: Union of all pivot tokens from all active blocks at their assigned resolutions [S1, unofficial].
5. **Sparse High-Resolution Field**:
   - $V_{max}$: Set of voxel indices $\{v_k\}$ at resolution $r_{max}$ [S1, unofficial].
   - $D_{max}$: Set of corner distance vectors $\{d(v_k)\}$ where $d(v_k) \in \mathbb{R}^8$ [S1, unofficial].

**Steps:**

1. **Partition Domain**:
   Partition the normalized space $[-0.5, 0.5]^3$ into a macro-block grid $B$ with block size $1/B$ [S1, unofficial].

2. **Compute Complexity Score**:
   For each macro-block $g$, compute the complexity score $S_B(g)$ using surface samples $x$ mapped to blocks by $g(x) = \lfloor (x + 0.5)B \rfloor$ [S1, unofficial]:
   $$
   S_B(g) = \bar{c}(g) + \lambda_{var} \sigma_c(g) + \lambda_{grad} \|\nabla \bar{c}(g)\| + \lambda_n \bar{\eta}(g) \cdot N(g)
   $$
   Where:
   - $\bar{c}(g)$: Mean curvature magnitude in block $g$.
   - $\sigma_c(g)$: Curvature variation in block $g$.
   - $\|\nabla \bar{c}(g)\|$: Spatial curvature change (gradient magnitude) in block $g$.
   - $\bar{\eta}(g)$: Local normal deviation in block $g$.
   - $N(g)$: Number of surface samples in block $g$.
   - $\lambda_{var}, \lambda_{grad}, \lambda_n$: Weights (typically 0.5, 0.3, 3.0 respectively) [S1, unofficial].

3. **Discretize Allocation Map**:
   Discretize $S_B(g)$ into an allocation map $M: g \mapsto r \in R$ using quantile-based binning over blocks [S1, unofficial].
   - $R$ is a predefined set of sampling resolutions.
   - $M$ is restricted to surface-intersecting blocks.
   - Missing active blocks are filled with $r_{min}$ [S1, unofficial].

4. **Perform Pyramid Sampling**:
   For each active macro-block $g$, perform pivot voxelization with resolution $r = M(g)$ to produce tokens $T_r = \{\tau_n(r)\}_{n=1}^{N_r}$, where $\tau(r) = \{p(r), s(r)\}$ [S1, unofficial].
   Form the final pyramid set $T_{pyr}$ by unioning tokens from all active blocks:
   $$
   T_{pyr} = \bigcup_{g \in B_{active}} \{ \tau(M(g)) \mid \tau \text{ lies in block } g \}
   $$
   [S1, unofficial].

5. **Lift to Maximum Resolution**:
   Lift all tokens in $T_{pyr}$ to the maximum resolution $r_{max} = \max(R)$ with voxel size $h_{max} = 1/r_{max}$ for unified Sparse-MC reconstruction [S1, unofficial].
   For each token $\tau(r) = \{p(r), s(r)\} \in T_{pyr}$:
   a. Recover its induced plane $\hat{S}(r) = (p(r), n(r))$, where $n(r)$ is the normal derived from the pivot [S1, unofficial].
   b. Subdivide the original voxel into $(r_{max}/r)^3$ fine voxels [S1, unofficial].
   c. Evaluate corner SDF samples to populate the sparse high-resolution field $(V_{max}, D_{max})$ [S1, unofficial].
      - $V_{max} = \{v_k\}_{k=1}^K$, where $v_k \in \{0, \dots, r_{max}-1\}^3$ [S1, unofficial].
      - $D_{max} = \{d(v_k)\}_{k=1}^K$, where $d(v_k) = [d_\delta(v_k)]_{\delta \in \{0,1\}^3} \in \mathbb{R}^8$ [S1, unofficial].
      - The signed distance at corner $\delta$ of voxel $v_k$ is:
        $$
        d_\delta(v_k) = (v_{k,\delta} - p(r))^\top n(r)
        $$
        Where $v_{k,\delta}$ is the corner coordinate, $p(r)$ is the pivot point, and $n(r)$ is the normal [S1, unofficial].

6. **Reconstruct Mesh**:
   Apply Sparse Marching Cubes (Sparse-MC) to the unified high-resolution field $(V_{max}, D_{max})$ to reconstruct the sparse watertight mesh $\hat{M}$ [S1, unofficial].

**Parameters:**
- **Resolution Levels**:
  - Pyramid-512: $\{128, 256, 512\}$ with allocation ratios $[0.85, 0.10, 0.05]$ [S1, unofficial].
  - Pyramid-1024: $\{128, 256, 512, 1024\}$ with allocation ratios $[0.50, 0.35, 0.12, 0.03]$ [S1, unofficial].
- **Hyperparameters**: $\lambda_{var} = 0.5$, $\lambda_{grad} = 0.3$, $\lambda_n = 3.0$ [S1, unofficial].
- **Default Max Resolution**: 512, as gains become marginal beyond this point [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missing Active Blocks**: Must be filled with $r_{min}$ to ensure consistent coverage [S1, unofficial].
- **Precision**: Corner distance evaluation relies on accurate pivot normals $n(r)$. Errors in normal estimation can lead to topological artifacts [S1, unofficial].
- **Resolution Scaling**: Artifact ratios (TopoC, HighCurv, All) decrease as resolution increases from 64 to 1024, but marginal gains beyond 512 suggest diminishing returns [S1, unofficial].

## Subtopics

### [Pyramid VAE](14370-pyramid-vae.md)
A Variational Autoencoder that learns compact multi-resolution latent codes over locally reconstructable pivot blocks, avoiding the need to model the entire high-resolution voxelized shape as a dense global field.

## Operating characteristics
- **Time Complexity**: Not explicitly given in closed form, but runtime scales with resolution. On RTX 4090, Pyramid voxelization at resolution 512 takes 8.39s (Sampling) + 8.98s (Pyramid map) + 5.96s (Resolution Sparse MC) + 14.55s (Voxelization Lifting) + 0.01s + 0.08s + 0.60s + 2.34s. At 1024, times increase proportionally [S1, unofficial].
- **Memory Complexity**: Not explicitly given, but the method is designed to be more memory-efficient than dense global voxel grids by using sparse pivot tokens [S1, unofficial].
- **Measured Performance**: On RTX 4090, inference speed is 1.69s per shape with 4,119 MiB peak VRAM [S1, unofficial].
- **Strengths**: Improves Pareto efficiency over SDF-based and Dual Contouring baselines by allocating more tokens to coarse levels and reserving fine-scale sampling for structurally informative regions [S1, unofficial].
- **Limitations**: Not a single-pass adaptive implementation; relies on closest-point queries and triangle normals from optimized C++ backends [S1, unofficial].
- **When to Use**: When geometric information density is non-uniform (high-frequency details concentrated around high-curvature, sharp, or thin structures) and token efficiency is important [S1, unofficial].
- **When Not to Use**: When uniform resolution is required or when the overhead of computing complexity scores and lifting tokens outweighs the benefits of adaptive sampling [S1, unofficial].

## Background
Pyramid Pivot Voxelization was introduced in the P2Voxel paper [S1, unofficial]. It is used as the core discretization method in P2Voxel, benchmarked against SDF-based methods (VecSet, DORA, Hunyuan3D-2.1) and Dual Contouring methods (FaithC, TRELLIS 2) on ABO, Objaverse, and Wild datasets [S1, unofficial]. The method is based on two key assumptions: Local Planarity (Assumption 1) and Spatial Complexity (Assumption 2) [S1, unofficial].

## Sources
[S1] P2Voxel: Pyramid Pivot Voxelization for 3D Mesh Tokenization, https://doi.org/10.48550/arxiv.2608.07549, fetched 2026-10-26 (unofficial)
