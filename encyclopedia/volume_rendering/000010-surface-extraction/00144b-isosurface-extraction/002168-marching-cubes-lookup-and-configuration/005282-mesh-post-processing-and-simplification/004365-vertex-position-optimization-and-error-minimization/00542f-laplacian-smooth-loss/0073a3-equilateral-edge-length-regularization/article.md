# Equilateral Edge Length Regularization

Also called: Edge Length Variance Regularization

## Overview
Equilateral Edge Length Regularization is a mesh post-processing technique that penalizes the variance of edge lengths on an extracted surface to encourage equilateral triangles, which are preferred for applications like physics simulation [S1]. Unlike rigid isosurface methods such as Marching Cubes (MC) or DMTet, which suffer significant geometric degradation (e.g., increased Chamfer Distance) when applying this regularizer, flexible representations like FlexiCubes can adapt their topology to maintain high reconstruction quality while improving triangle metrics [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A mesh $M$ extracted from a scalar field (typically via FlexiCubes, MC, or DMTet), defined by vertices $V$, edges $E$, and triangles $T$.
- **Output:** Optimized vertex positions $V'$ that minimize the regularization loss.

**Data Structures:**
- $E$: Set of all unique edges in the mesh.
- $T$: Set of all triangles in the mesh.
- $|e|$: Euclidean length of edge $e$.

**Formulas:**
The average edge length $\bar{e}$ is computed as the mean of the squared edge lengths:
$$ \bar{e} = \frac{1}{|E|} \sum_{e \in E} |e|^2 $$
The regularization loss $L_{edge}$ is the mean squared deviation of each edge length from this average:
$$ L_{edge} = \frac{1}{|T|} \sum_{t \in T} \sum_{e \in t} (|e| - \bar{e})^2 $$
where $|T|$ is the number of triangles, and the inner sum iterates over the three edges of each triangle $t$ [S1].

**Parameters:**
- Pre-optimization steps: 1000 [S1].
- Post-optimization steps with regularization: 300 [S1].
- Maximum regularization weight: 100 [S1].

**Algorithm:**
1. **Pre-optimization:** Run gradient-based optimization to reconstruct the input mesh using only the reconstruction loss (e.g., distance to ground truth) for 1000 steps. Do not apply regularization [S1].
2. **Regularized Optimization:** Continue optimization for an additional 300 steps. In each step, the total loss is a combination of the reconstruction loss and $L_{edge}$ [S1].
3. **Weight Schedule:** Linearly increase the regularization weight $\lambda$ from 0 to 100 over the 300 post-optimization steps [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def equilateral_edge_length_regularization(mesh, reconstruction_loss_fn, steps_pre=1000, steps_post=300, max_weight=100):
    """
    Optimizes mesh vertices to minimize edge length variance after initial reconstruction.
    
    Args:
        mesh: Object containing vertices V, edges E, and triangles T.
        reconstruction_loss_fn: Function returning reconstruction loss given current V.
        steps_pre: Number of steps without regularization.
        steps_post: Number of steps with regularization.
        max_weight: Maximum weight for the regularization term.
        
    Returns:
        Optimized vertices V.
    """
    V = mesh.vertices.clone()
    E = mesh.edges
    T = mesh.triangles
    
    # Step 1: Pre-optimization without regularization
    for step in range(steps_pre):
        # Compute reconstruction loss based on current V
        loss_rec = reconstruction_loss_fn(V)
        # Backpropagate and update V
        loss_rec.backward()
        # Update V using optimizer (e.g., Adam)
        optimizer.step()
        optimizer.zero_grad()
        
    # Step 2 & 3: Regularized optimization with linearly increasing weight
    for step in range(steps_post):
        # Compute average edge length squared
        # Note: Formula uses sum of squared lengths for average, then subtracts average from length
        sum_sq_lengths = 0.0
        count_edges = 0
        for e in E:
            p1, p2 = e.vertices
            length = torch.norm(p2 - p1)
            sum_sq_lengths += length ** 2
            count_edges += 1
            
        avg_sq_length = sum_sq_lengths / count_edges
        # The formula defines average edge length as mean of squared lengths
        # However, the loss subtracts this average from the linear length.
        # We must compute the scalar value bar_e = avg_sq_length
        bar_e = avg_sq_length 
        
        # Compute regularization loss
        loss_reg = 0.0
        count_triangles = 0
        for t in T:
            edges_in_t = t.edges
            for e in edges_in_t:
                p1, p2 = e.vertices
                length = torch.norm(p2 - p1)
                diff = length - bar_e
                loss_reg += diff ** 2
            count_triangles += 1
            
        loss_reg = loss_reg / count_triangles
        
        # Compute reconstruction loss
        loss_rec = reconstruction_loss_fn(V)
        
        # Linearly schedule weight from 0 to max_weight
        weight = max_weight * (step / steps_post)
        
        # Total loss
        total_loss = loss_rec + weight * loss_reg
        
        # Backpropagate and update V
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return V
```

**Edge Cases and Pitfalls:**
- **Precision:** Computing squared lengths and subtracting a squared average from a linear length involves mixed powers. Ensure consistent data types to avoid precision loss [S1].
- **Boundary Conditions:** The method assumes a closed or well-defined mesh. Degenerate edges (length 0) may cause numerical instability in gradients [G].
- **Rigid Representations:** Applying this regularizer to Marching Cubes or DMTet causes significant drops in geometric quality (Chamfer Distance, Inaccurate Normals) because these representations lack the degrees of freedom to adapt to the objective [S1].

## Subtopics

### FlexiCubes
A flexible isosurface representation that supports gradient-based mesh optimization. It allows the mesh topology to adapt, making it suitable for this regularizer without sacrificing reconstruction quality. See [FlexiCubes](52-flexicubes.md).

### Marching Cubes (MC)
A rigid isosurface extraction method. Adding this regularizer to MC leads to poor geometric metrics (11.46% Aspect Ratio > 4, 11.43% Radius Ratio > 4) compared to FlexiCubes [S1]. See [Marching Cubes](52-marching-cubes.md).

### DMTet
A differentiable tetrahedron-based isosurface method. Similar to MC, it suffers from geometric degradation when regularized, though slightly better than MC in some metrics (0.29% Aspect Ratio > 4). See [DMTet](52-dmtet.md).

## Operating characteristics
**Time and Memory Complexity:**
- The regularization loss requires iterating over all edges and triangles, resulting in $O(|E| + |T|)$ complexity per optimization step.
- Memory overhead is minimal, storing only edge and triangle indices [S1].

**Measured Performance:**
- On the test set (Table 4 in [S1]), FlexiCubes + Reg. achieves 0.39% Aspect Ratio > 4, 0.69% Radius Ratio > 4, and 0.24% Min Angle < 10°, compared to MC + Reg. (11.46%, 11.43%, 11.62%) and DMTet(80) + Reg. (0.29%, 0.46%, 0.26%) [S1].
- FlexiCubes + Reg. has a Chamfer Distance (CD) of 5.46, compared to MC + Reg. (8.56) and DMTet(80) + Reg. (6.69) [S1].

**Strengths and Limitations:**
- **Strengths:** Significantly improves triangle quality metrics (Aspect Ratio, Radius Ratio, Min Angle) for flexible representations [S1].
- **Limitations:** Incompatible with rigid representations like MC and DMTet, which suffer significant geometric quality drops [S1]. Requires gradient-based optimization framework [S1].

**When to Use:**
- Use when equilateral triangles are required for downstream applications like physics simulation [S1].
- Use with flexible isosurface representations like FlexiCubes [S1].

**When Not to Use:**
- Do not use with Marching Cubes or DMTet if geometric accuracy (CD, Normals) is critical, as the regularizer degrades these metrics significantly [S1].

## Background
Equilateral Edge Length Regularization was introduced in the context of gradient-based mesh optimization for flexible isosurface extraction [S1]. It addresses the issue of thin or distorted triangles in extracted meshes, which are problematic for physics simulations [S1]. The method was evaluated alongside other regularizers in the FlexiCubes paper, demonstrating its effectiveness when combined with a flexible representation [S1].

## Sources
[S1] Flexible Isosurface Extraction for Gradient-Based Mesh Optimization, https://arxiv.org/abs/2308.05371v1, fetched 2026-10-26
