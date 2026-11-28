# Multiphase Signed Distance Field

Also called: Multiphase SDF

## Overview
Multiphase Signed Distance Fields (MP-SDFs) extend classical signed distance functions by integrating phase information directly into a construction tree, enabling the native representation of multi-material and multiphase objects [S1]. Unlike standard SDFs that model a single scalar field, MP-SDFs maintain a vector of phase functions $\Phi(\mathbf{p}) = (\phi_1(\mathbf{p}), \dots, \phi_n(\mathbf{p}))$ that partition space into distinct material regions while preserving the 1-Lipschitz property required for algorithmic compatibility with Sphere Tracing [S1]. This approach supports constructive solid geometry (CSG) workflows and novel operators like thin shells and phase merging, which are not directly supported by monolithic SDF representations.

## How it works
### Inputs and Outputs
*   **Input:** A construction tree (PhaseTree) where leaves are monophase SDFs or standard SDF primitives, and internal nodes are operators (conversion, thin shell, merging, combination).
*   **Output:** A multiphase field $\Phi: \mathbb{R}^3 \to \mathbb{R}^n$ and an associated empty-phase function $\phi_0$, allowing ray marching to determine both geometry and material identity.

### Data Structures
The core data structure is the PhaseTree, a binary or n-ary tree where each node evaluates to a multiphase model.
*   **Leaf Node:** Represents a monophase SDF $f$. It is converted into a multiphase model with $n$ phases (where $n$ is the global number of phases in the scene or subtree).
*   **Internal Node:** Represents an operator combining child nodes. It maintains the 1-Lipschitz property if its children do and the operator is Lipschitz-preserving.
*   **Phase Vector:** At any point $\mathbf{p}$, the state is defined by the vector $\Phi(\mathbf{p}) = [\phi_1(\mathbf{p}), \dots, \phi_n(\mathbf{p})]$.

### Definitions and Invariants
The MP-SDF model relies on the **Single-Negative-Phase Invariant**: for any point $\mathbf{p} \in \mathbb{R}^3$, there exists a unique index $k \in [1, n]$ such that $\phi_k(\mathbf{p}) < 0$ [S1]. If no phase is negative, the point is in the empty region $\Omega_0$.

The **Phase Type** $\mu(\mathbf{p})$ at a point is the index of the minimum phase function value:
$$ \mu(\mathbf{p}) = \arg \min_{i \in [1,n]} \phi_i(\mathbf{p}) $$
If $\min_i \phi_i(\mathbf{p}) > 0$, then $\mu(\mathbf{p}) = 0$ (empty space) [S1].

The **Phase Regions** $\Omega_i$ are defined as the partition of space where phase $i$ is negative and all other phases $j \neq i$ are positive:
$$ \Omega_i = \{\mathbf{p} \in \mathbb{R}^3 \mid \phi_i(\mathbf{p}) < 0, \text{ and } \phi_j(\mathbf{p}) > 0 \forall j \neq i\} $$
The **Empty Region** $\Omega_0$ consists of points not belonging to any phase:
$$ \Omega_0 = \mathbb{R}^3 - \bigcup_{i \in [1,n]} \Omega_i - \bigcup_{(i,j) \in \text{Interfaces}} \Omega_{ij} $$
The signed distance to the outside (empty space) is computed as:
$$ \phi_0(\mathbf{p}) = - \min_{i \in [1,n]} \phi_i(\mathbf{p}) $$
Note that $\phi_0(\mathbf{p}) \ge 0$ inside the object union, and $\phi_0(\mathbf{p}) > 0$ in empty space [S1].

The **Interface** between phase $i$ and $j$ is the set of points where both phase functions are zero:
$$ \Omega_{ij} = \{\mathbf{p} \in \mathbb{R}^3 \mid \phi_i(\mathbf{p}) = 0 \text{ and } \phi_j(\mathbf{p}) = 0\} $$
At these interfaces, the gradients $\nabla \phi_i$ and $\nabla \phi_j$ are collinear, ensuring consistent normal directions $\mathbf{n}_i(\mathbf{p}) = -\mathbf{n}_j(\mathbf{p})$ [S1].

### Operators
The PhaseTree supports several operators. All operators described below preserve the 1-Lipschitz property if their inputs are 1-Lipschitz [S1].

#### 1. Set $k$ Conversion Operator
Converts a standard SDF $f$ into a multiphase model where phase $k$ is $f$ and all other phases $i \neq k$ are infinity. This allows standard SDF sub-trees to be integrated into the multiphase framework.
$$ \Phi = \begin{cases} \phi_k = f \\ \phi_i = \infty \quad \forall i \neq k \end{cases} $$
Defining $\phi_i(\mathbf{p}) = \infty$ indicates that phase $i$ does not exist at $\mathbf{p}$ [S1].

#### 2. Thin Shell Operator
Generates two phases from a control shape SDF $f$ and a thickness parameter $t$. Phase $i$ represents the shell surface, and phase $j$ represents the interior/exterior depending on the sign convention.
$$ \Phi = \begin{cases} \phi_i = |f(\mathbf{p})| - t \\ \phi_j = f + t \end{cases} $$
Here, $t$ is the thickness of the generated shell [S1]. The shell exists where $\phi_i < 0$.

#### 3. Phase Merging Operator
Fuses a subset of phases $\mathcal{K}$ into a single standard SDF $f$. This is useful for applying single-phase algorithms to a subset of materials.
$$ f = \min_{i \in \mathcal{K}} \phi_i $$
The resulting function $f$ is 1-Lipschitz. However, $f(\mathbf{p}) = 0$ at the interfaces $\partial\Omega_{ij}$ between the original phases in $\mathcal{K}$, which may require special handling for smooth rendering or further operations [S1].

#### 4. Under Operator ($\cup_-$)
A non-commutative combination operator that performs a union of phases while replacing the phases of the first argument with those of the second. It is defined as:
$$ \mathcal{A} \cup_- \mathcal{B} = \left\{ (\mathcal{A}_i \cup \mathcal{B}_i) - \bigcup_{j \neq i} \mathcal{B}_j \right\}_{i \in [1, n]} $$
The corresponding phase functions for the result are computed as:
$$ \forall i \in [1, n], \quad \phi_i = \max(\min(\phi_{\mathcal{A}_i}, \phi_{\mathcal{B}_i}), -\min(\phi_{\mathcal{B}_i})) $$
This operator allows for complex layering and material replacement effects [S1].

### Pseudocode
The following Python-like pseudocode implements the evaluation of a multiphase SDF at a point $\mathbf{p}$, assuming a tree structure where each node has an `evaluate` method.

```python
import math
import numpy as np

def evaluate_multiphase_sdf(node, p, n_phases):
    """
    Evaluates the multiphase SDF at point p for a given tree node.
    
    Args:
        node: The current node in the PhaseTree.
        p: The 3D query point (numpy array).
        n_phases: The total number of phases in the scene.
        
    Returns:
        phi: A numpy array of size n_phases + 1, containing [phi_1, ..., phi_n, phi_0].
    """
    
    # Base case: Leaf node representing a standard SDF f
    if node.is_leaf():
        f = node.evaluate_standard_sdf(p)
        # Convert to multiphase: phase k is f, others are infinity
        # Note: The specific phase index k is determined by the leaf's metadata
        k = node.phase_index
        phi = np.full(n_phases, np.inf)
        phi[k - 1] = f  # 0-indexed array, so phase k is at index k-1
        # Compute phi_0 based on the invariant
        phi_0 = -np.min(phi)
        return np.append(phi, phi_0)

    # Internal node: Apply operator
    elif node.operator == 'under':
        # Evaluate children
        phi_a = evaluate_multiphase_sdf(node.left, p, n_phases)
        phi_b = evaluate_multiphase_sdf(node.right, p, n_phases)
        
        # Extract phase vectors (exclude phi_0)
        phases_a = phi_a[:-1]
        phases_b = phi_b[:-1]
        
        # Compute result phases using the Under operator formula
        result_phases = np.zeros(n_phases)
        for i in range(n_phases):
            # phi_i = max(min(phi_A_i, phi_B_i), -min(phi_B_i))
            # Note: min(phi_B_i) is just phases_b[i]
            val = math.min(phases_a[i], phases_b[i])
            val = math.max(val, -phases_b[i])
            result_phases[i] = val
            
        # Compute phi_0
        phi_0 = -np.min(result_phases)
        return np.append(result_phases, phi_0)

    elif node.operator == 'thin_shell':
        # Evaluate the control shape SDF
        f = node.left.evaluate_standard_sdf(p)
        t = node.thickness
        
        # Determine phase indices for the shell
        i_idx = node.phase_i - 1
        j_idx = node.phase_j - 1
        
        phi = np.full(n_phases, np.inf)
        phi[i_idx] = abs(f) - t
        phi[j_idx] = f + t
        
        phi_0 = -np.min(phi)
        return np.append(phi, phi_0)

    elif node.operator == 'merge':
        # Merge operator returns a single SDF, but for consistency in tree evaluation,
        # we might need to handle it differently. 
        # However, merge is typically used to extract a single SDF, not to produce a multiphase vector.
        # If used as a node in a multiphase tree, it might be a leaf for a specific phase.
        # For this implementation, we assume merge is not a branching node in the multiphase evaluation
        # but rather a post-processing step or a leaf that sets one phase.
        # If it is a node, it likely sets one phase to the min of children and others to inf.
        # Let's assume it sets phase k to the min of its children's phase k.
        # This is a simplification; the source describes merge as converting back to standard SDF.
        # For a full tree evaluation, merge is not a standard internal node for multiphase output.
        # We will treat it as a leaf that computes a standard SDF and converts it.
        f = node.evaluate_standard_sdf(p) # Assumes merge node can evaluate to a scalar
        k = node.phase_index
        phi = np.full(n_phases, np.inf)
        phi[k - 1] = f
        phi_0 = -np.min(phi)
        return np.append(phi, phi_0)

    else:
        raise ValueError(f"Unknown operator: {node.operator}")

def ray_march_mp_sdf(origin, direction, tree, max_steps=100, max_dist=100.0):
    """
    Ray marches using the multiphase SDF.
    
    Args:
        origin: Ray origin.
        direction: Ray direction.
        tree: The root of the PhaseTree.
        max_steps: Maximum number of march steps.
        max_dist: Maximum ray distance.
        
    Returns:
        hit: Boolean, whether a hit occurred.
        t: Distance to hit.
        phase: Phase index of the hit.
        normal: Normal at the hit point.
    """
    t = 0.0
    for _ in range(max_steps):
        if t > max_dist:
            return False, t, 0, np.zeros(3)
            
        p = origin + t * direction
        phi = evaluate_multiphase_sdf(tree, p, tree.n_phases)
        
        # The distance to the closest surface is the minimum of all phase distances
        # Note: phi[:-1] are the phase distances, phi[-1] is phi_0
        # However, phi_0 is defined as -min(phi_i), so min(phi_i) = -phi_0
        # The distance to the surface is min(phi_i) if inside, or min(phi_i) if outside?
        # Actually, for sphere tracing, we need the distance to the zero level set.
        # The distance to the surface is min(phi_i) for all i.
        # If min(phi_i) > 0, we are outside. If min(phi_i) < 0, we are inside.
        # The step size is min(phi_i).
        
        min_phi = np.min(phi[:-1])
        
        if min_phi < 1e-4: # Hit threshold
            # We are on or inside the surface
            # The phase is the index of the minimum phi
            phase_idx = np.argmin(phi[:-1]) + 1
            # Compute normal using gradient of the active phase
            # For simplicity, assume we can compute gradient of the active phase
            # In practice, this requires differentiating the tree
            normal = compute_normal(tree, p, phase_idx)
            return True, t, phase_idx, normal
        
        t += min_phi
        
    return False, t, 0, np.zeros(3)
```

### Parameters
*   **Thickness ($t$):** Used in the Thin Shell operator to define the thickness of the generated shell [S1].
*   **Phase Indices ($i, j, k$):** Integer indices identifying specific phases in the vector $\Phi$ [S1].

### Edge Cases and Pitfalls
*   **Merging Interfaces:** When using the Phase Merging operator, the resulting function satisfies $f(\mathbf{p}) = 0$ at the interfaces $\partial\Omega_{ij}$ between the original phases. This can cause artifacts in rendering or further operations if not handled [S1].
*   **Conservative Functions:** Conservative functions (where $|\phi_i(\mathbf{p})| \leq d(\mathbf{p}, \partial\Omega_i)$) can be used but generally do not accommodate smooth or extended blending operations, unlike 1-Lipschitz functions. The MP-SDF formulation prioritizes 1-Lipschitz continuity to ensure robust CSG and blending [S1].
*   **Infinity Handling:** The Set $k$ conversion operator uses $\infty$ for non-active phases. Implementations must handle floating-point infinity correctly to avoid NaN propagation during min/max operations [S1].

## Operating characteristics
*   **Time Complexity:** Evaluation of a point $\mathbf{p}$ is proportional to the depth of the PhaseTree. Each node performs constant-time operations (min, max, abs) on vectors of size $n$ (number of phases). Thus, evaluation is $O(D \cdot n)$, where $D$ is tree depth. Ray marching step count is similar to standard SDFs, depending on the Lipschitz constant (which is 1).
*   **Memory Complexity:** The memory footprint is proportional to the number of nodes in the PhaseTree and the number of phases $n$. Each node stores operator metadata and references to children. The phase vector size is $O(n)$.
*   **Strengths:** Native support for multi-material objects; compatibility with Sphere Tracing; supports complex operators like thin shells and phase merging; preserves 1-Lipschitz property for robust marching.
*   **Limitations:** Increased computational cost per evaluation due to vector operations on $n$ phases; complexity in implementing gradient computation for normals in multiphase contexts; merging phases can introduce zero-level set artifacts at interfaces.
*   **When to Use:** When rendering scenes with multiple distinct materials or phases that interact geometrically (e.g., shells, composites) and require efficient ray marching.
*   **When Not to Use:** For simple single-material scenes where standard SDFs are sufficient; when smooth blending between phases is required without the overhead of multiphase management (though MP-SDF supports blending, it is more complex).

## Background
The Multiphase Signed Distance Field, also known as the PhaseTree, was introduced by Hubert-Brierre et al. in 2025 [S1]. It extends the classical SDF framework to support multi-material objects while maintaining the efficiency and robustness of sphere tracing. The work builds on the BlobTree [Wyvill et al. 1999] and addresses limitations of conservative distance fields in handling smooth blending [Dekkers et al. 2004]. It is used in computer graphics for modeling and rendering complex multiphase objects.

## Sources
[S1] The PhaseTree: Multiphase Signed Distance Fields, https://doi.org/10.1145/3811379, fetched 2026-10-26
