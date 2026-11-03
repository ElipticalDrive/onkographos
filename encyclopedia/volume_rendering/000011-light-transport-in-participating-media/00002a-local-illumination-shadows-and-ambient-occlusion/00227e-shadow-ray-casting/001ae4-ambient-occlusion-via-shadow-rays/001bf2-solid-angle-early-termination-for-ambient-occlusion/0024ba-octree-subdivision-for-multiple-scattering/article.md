# Octree Subdivision for Multiple Scattering

Also called: Double Scattering via Tree Cut, Tree-Cut Multiple Scattering

## Overview
This method computes multiple scattering contributions in participating media by traversing an octree of volume samples to identify a "tree cut" of nodes that significantly influence a camera sample point. It addresses the computational cost of evaluating light transport from distant or less relevant volume samples by combining solid angle thresholds with phase function checks. It builds on [Solid Angle Early Termination for Ambient Occlusion](1146-solid-angle-early-termination-for-ambient-occlusion.md) by extending the early termination logic to higher-order scattering events, specifically double and multiple scattering, rather than just direct illumination or ambient occlusion.

## How it works
**Inputs:**
- Camera sample position $P_k$.
- Camera ray direction $\hat{o}$.
- Octree of volume samples, where each cell $C$ has a center $C.r$ and an incoming light direction $C.d$.
- Volume sample positions $v_i$ associated with leaf nodes or specific nodes in the hierarchy.
- Scattering coefficient $\sigma_s$.
- Extinction coefficient $\sigma_i$ for each sample $i$.
- Phase function $p(\cdot, \cdot)$.
- Precomputed multiple scattering table (for higher-order scattering).
- Thresholds $\epsilon_1$ and $\epsilon_2$.

**Outputs:**
- Multiple scattering contribution to the radiance at $P_k$ in direction $\hat{o}$.

**Data Structures:**
- **Octree Cell:** Contains center position $C.r$, incoming light direction $C.d$, and references to child cells or volume samples.
- **Volume Sample:** Contains position $v_i$, intensity $I_{v_i}$, and extinction coefficient $\sigma_i$.

**Steps:**

1. **Initialize Traversal:** Start at the root of the octree.
2. **Evaluate Node:** For the current node (cell $C$) and sample point $P_k$:
   - Compute the vector $\mathbf{u} = P_k - C.r$.
   - Compute the unit vector $\hat{u} = \mathbf{u} / \|\mathbf{u}\|$.
   - Calculate the solid angle $\Omega(P_k, C)$ subtended by the cell $C$ at $P_k$.
3. **Apply Subdivision Criteria:**
   - If $\Omega(P_k, C) > \epsilon_1$, the node is too large to be treated as a single point source; subdivide and process children.
   - Else if $\Omega(P_k, C) > \epsilon_2$ AND the phase function indicates significant scattering contribution, subdivide. The phase function check is:
     $$ p(\hat{o}, \hat{u}) > \pi/4 \quad \text{or} \quad p(\hat{u}, C.d) > \pi/4 $$
     If this condition is met, subdivide.
   - Else, the node is sufficiently small and/or oriented such that its contribution can be approximated by a single sample. Add this node to the **tree cut**.
4. **Compute Double Scattering:** For each node $v_i$ in the tree cut:
   - Compute the vector $\mathbf{r}_i = P_k - v_i$.
   - Compute the distance $r_i = \|\mathbf{r}_i\|$.
   - Compute the unit vector $\hat{r}_i = \mathbf{r}_i / r_i$.
   - Calculate the double scattering contribution using the formula:
     $$ \text{double}(P_k) = \sigma_s^2 \sum_{v_i \in \text{tree cut}} \frac{e^{-\sigma_i r_i} I_{v_i}}{r_i^2} p(\hat{o}, \hat{r}_i) p(\hat{r}_i, \hat{\alpha}_{v_i}) $$
     where $\hat{\alpha}_{v_i}$ is the direction of incoming light at $v_i$ (derived from $C.d$ or precomputed).
   - **Pitfall Handling:** Clamp the $1/r_i^2$ term to avoid weak singularities when $r_i$ is very small.
5. **Compute Higher-Order Multiple Scattering:** For each node $v_i$ in the tree cut, use the precomputed table to estimate scattering events beyond double scattering. The table is generated via Monte Carlo simulation of photon propagation in an infinite medium, mapping incident direction and distance to accumulated radiance.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multiple_scattering(P_k, o_hat, octree_root, epsilon_1, epsilon_2, sigma_s, phase_function, precomputed_table):
    tree_cut = []
    
    def traverse(node):
        # Compute vector from sample to node center
        u_vec = P_k - node.center
        u_hat = normalize(u_vec)
        dist_u = length(u_vec)
        
        # Compute solid angle subtended by node at P_k
        # Implementation detail: solid angle depends on node geometry and distance
        omega = compute_solid_angle(node, P_k)
        
        # Check subdivision criteria
        if omega > epsilon_1:
            # Subdivide
            for child in node.children:
                traverse(child)
        elif omega > epsilon_2:
            # Check phase function conditions
            # p(o_hat, u_hat) > pi/4 or p(u_hat, node.incoming_light_dir) > pi/4
            cond1 = phase_function(o_hat, u_hat) > math.pi / 4
            cond2 = phase_function(u_hat, node.incoming_light_dir) > math.pi / 4
            if cond1 or cond2:
                # Subdivide
                for child in node.children:
                    traverse(child)
            else:
                # Add to tree cut
                tree_cut.append(node.sample)
        else:
            # Add to tree cut
            tree_cut.append(node.sample)

    traverse(octree_root)
    
    double_scattering = 0.0
    for sample in tree_cut:
        r_vec = P_k - sample.position
        r_dist = length(r_vec)
        r_hat = normalize(r_vec)
        
        # Clamp 1/r^2 to avoid singularity
        # [G] Typical clamp value is a small epsilon, e.g., 1e-6, or min distance squared
        r_dist_sq = max(r_dist ** 2, 1e-6)
        
        # Double scattering term
        # I_v_i is sample.intensity, sigma_i is sample.extinction
        term = (sigma_s ** 2) * (exp(-sample.extinction * r_dist) * sample.intensity / r_dist_sq)
        term *= phase_function(o_hat, r_hat)
        term *= phase_function(r_hat, sample.incoming_light_dir)
        
        double_scattering += term
        
        # Higher-order scattering from precomputed table
        # [G] Table lookup depends on specific implementation, e.g., distance and angles
        # multiple_scattering += lookup_table(r_dist, o_hat, r_hat, sample.incoming_light_dir)
        
    return double_scattering
```

**Parameters:**
- $\epsilon_1$: Threshold for solid angle to force subdivision. [S1]
- $\epsilon_2$: Secondary threshold for solid angle combined with phase function check. [S1]
- $\sigma_s$: Scattering coefficient. [S1]
- $\sigma_i$: Extinction coefficient for sample $i$. [S1]

**Edge Cases and Pitfalls:**
- **Weak Singularities:** The $1/r^2$ term in the double scattering formula can cause numerical instability when $r_i$ is very small. This is handled by clamping the distance squared. [S1]
- **Degenerate Input:** If $P_k$ coincides with a sample position, $r_i = 0$. Clamping prevents division by zero.

## Subtopics

### Double Scattering via Tree Cut
This variant focuses specifically on the double scattering component. It uses the same tree cut mechanism but computes only the first two scattering events (sample to $P_k$ to camera). It is chosen when computational resources are limited and higher-order scattering is negligible or approximated separately. It sets itself apart by explicitly computing the $1/r^2$ attenuation and phase function evaluations for each node in the cut. [S1]

## Operating characteristics
- **Time Complexity:** Depends on the depth of the octree and the number of nodes in the tree cut. In the worst case, it traverses the entire tree. In practice, the solid angle and phase function checks prune the tree significantly.
- **Memory Complexity:** Linear in the number of volume samples stored in the octree.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Efficiently approximates multiple scattering by ignoring distant or poorly oriented samples. Reduces the number of expensive phase function and attenuation calculations.
- **Limitations:** Accuracy depends on the thresholds $\epsilon_1$ and $\epsilon_2$. Poorly chosen thresholds can lead to over-subdivision (slow) or under-subdivision (artifacts). The method assumes homogeneous media or precomputed per-sample properties.
- **When to Use:** When simulating participating media with significant multiple scattering and performance is a concern.
- **When Not to Use:** When high accuracy is required for near-field scattering or when the media is highly heterogeneous and cannot be well-represented by the octree samples.

## Background
The method originates from the work on point-based rendering for homogeneous participating media with refractive boundaries. It extends the PBGI (Point-Based Global Illumination) approach to multiple scattering events. The tree cut mechanism is inspired by solid angle-based early termination techniques used in ambient occlusion. The precomputed table for higher-order scattering is generated via Monte Carlo simulation of photon propagation in an infinite medium. [S1]

## Sources
[S1] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
