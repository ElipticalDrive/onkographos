# Gaussian Mixture Tree Construction

Also called: Hierarchically Merging Gaussians

## Overview
Gaussian Mixture Tree (GMT) Construction is a post-processing method that organizes 3D Gaussian primitives into a hierarchical tree structure to identify distinct 3D plane instances in a scene [S1]. It solves the problem of spatially scattered Gaussian centers by grouping them into coherent planar surfaces, enabling higher-level semantic or geometric understanding [S1]. Unlike sibling methods that focus on rendering efficiency or compression, GMT explicitly models the scene as a composition of 3D planes using Gaussian Mixture Models (GMM) to join nodes recursively from leaves to root [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N_s$ leaf nodes, where each leaf node $G_i$ represents a local cluster of Gaussian primitives. Each node $G_i$ is defined by mean $\mu_i$, covariance $\Sigma_i$, normal vector $n_i \in \mathbb{R}^3$, and plane descriptor $z_i \in \mathbb{R}^k$ (typically $k=3$) [S1].
*   **Output:** A set of parent nodes $G_s$ representing merged plane instances, and the root of the Gaussian Mixture Tree [S1].

**Data Structures**
Each node in the GMT stores:
*   $\mu \in \mathbb{R}^3$: Mean position.
*   $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix.
*   $n \in \mathbb{R}^3$: Surface normal vector.
*   $z \in \mathbb{R}^k$: Plane descriptor vector (learnable or derived from 2D segmentation) [S1].

**Algorithm Steps**
The construction follows a greedy hierarchical merging strategy [S1]:

1.  Initialize counter $L = 0$ and output set $G_s = \emptyset$.
2.  Iterate through each leaf node $i$ from $1$ to $N_s$:
    a. Check if node $i$ is already a descendant of any existing node in $G_s$. If yes, skip to the next $i$.
    b. If node $i$ is not yet merged, initialize it as a candidate for a new parent node.
    c. Iterate through each subsequent leaf node $j$ from $i+1$ to $N_s$:
        i. Check if node $j$ is already a descendant of any existing node in $G_s$. If yes, skip to the next $j$.
        ii. Calculate the Bhattacharya distance $D_B(G_i, G_j)$ and the cosine similarity difference between descriptors $z_i$ and $z_j$.
        iii. If $D_B(G_i, G_j) \leq \epsilon_B$ AND $1 - \langle z_i \cdot z_j \rangle \leq \epsilon_z$:
            1. Merge $G_i$ and $G_j$ into a parent node $G_{ij}^p$ with parameters $(\mu_p, \Sigma_p)$ using the GMM merging formulas.
            2. Update $G_i \leftarrow G_{ij}^p$.
            3. Break the inner loop (stop checking further $j$ for this $i$).
    d. Insert the resulting node $G_i$ into $G_s$.
    e. Increment $L \leftarrow L + 1$.

**Formulas**
The merging of two Gaussian distributions $G_i(\mu_i, \Sigma_i)$ and $G_j(\mu_j, \Sigma_j)$ into a parent $G_p(\mu_p, \Sigma_p)$ uses the following equations [S1]:

$$
\mathbf{\Sigma}_p = \mathbf{\Sigma}_j \cdot (\mathbf{\Sigma}_i + \mathbf{\Sigma}_j)^{-1} \cdot \mathbf{\Sigma}_i
$$

$$
\mathbf{\mu}_p = \mathbf{\Sigma}_j \cdot (\mathbf{\Sigma}_i + \mathbf{\Sigma}_j)^{-1} \cdot \mathbf{\mu}_i + \mathbf{\Sigma}_i \cdot (\mathbf{\Sigma}_i + \mathbf{\Sigma}_j)^{-1} \cdot \mathbf{\mu}_j
$$

The merging criterion relies on the Bhattacharya distance $D_B$ [S1]:

$$
D_B(G_i, G_j) = \frac{1}{8}(\mathbf{\mu}_i-\mathbf{\mu}_j)^{T}\mathbf{\Sigma}^{-1}(\mathbf{\mu}_i-\mathbf{\mu}_j) + \frac{1}{2}\ln \left(\frac {\det \mathbf{\Sigma }}{\sqrt {\det \mathbf{\Sigma }_i \det \mathbf{\Sigma }_j}}\right)
$$

where $\mathbf{\Sigma} = (\mathbf{\Sigma}_i + \mathbf{\Sigma}_j)/2$ [S1].

**Parameters**
*   $\epsilon_B$: Threshold for the Bhattacharya distance. Not given in the sources.
*   $\epsilon_z$: Threshold for the cosine similarity difference between plane descriptors $z_i$ and $z_j$. Not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_gaussian_mixture_tree(leaf_nodes, epsilon_b, epsilon_z):
    """
    Constructs a Gaussian Mixture Tree by hierarchically merging leaf nodes.
    
    Args:
        leaf_nodes: List of Node objects, each with attributes:
            - mu: numpy array (3,)
            - Sigma: numpy array (3, 3)
            - z: numpy array (k,)
            - is_merged: boolean (initially False)
        epsilon_b: float, Bhattacharya distance threshold.
        epsilon_z: float, cosine similarity difference threshold.
            
    Returns:
        parent_nodes: List of merged Node objects representing plane instances.
    """
    parent_nodes = []
    n = len(leaf_nodes)
    
    for i in range(n):
        node_i = leaf_nodes[i]
        
        # Skip if already merged into a previous parent
        if node_i.is_merged:
            continue
            
        # Start a new potential parent node with node_i
        current_node = node_i
        
        for j in range(i + 1, n):
            node_j = leaf_nodes[j]
            
            # Skip if already merged into a previous parent
            if node_j.is_merged:
                continue
                
            # Calculate Bhattacharya distance
            # Sigma_avg = (Sigma_i + Sigma_j) / 2
            sigma_avg = (current_node.Sigma + node_j.Sigma) / 2.0
            
            # Term 1: Mahalanobis distance part
            diff_mu = current_node.mu - node_j.mu
            try:
                inv_sigma_avg = np.linalg.inv(sigma_avg)
            except np.linalg.LinAlgError:
                # Handle singular matrix if necessary [G]
                continue
                
            term1 = 0.25 * diff_mu.T @ inv_sigma_avg @ diff_mu
            
            # Term 2: Determinant part
            det_sigma_avg = np.linalg.det(sigma_avg)
            det_sigma_i = np.linalg.det(current_node.Sigma)
            det_sigma_j = np.linalg.det(node_j.Sigma)
            
            # Avoid log of non-positive numbers [G]
            if det_sigma_avg <= 0 or det_sigma_i <= 0 or det_sigma_j <= 0:
                continue
                
            term2 = 0.5 * np.log(det_sigma_avg / np.sqrt(det_sigma_i * det_sigma_j))
            
            db_dist = term1 + term2
            
            # Calculate cosine similarity difference
            # Normalize descriptors if not already normalized [G]
            z_i_norm = current_node.z / (np.linalg.norm(current_node.z) + 1e-8)
            z_j_norm = node_j.z / (np.linalg.norm(node_j.z) + 1e-8)
            cosine_sim = np.dot(z_i_norm, z_j_norm)
            cosine_diff = 1.0 - cosine_sim
            
            # Check merging criteria
            if db_dist <= epsilon_b and cosine_diff <= epsilon_z:
                # Merge node_j into current_node
                # Sigma_p = Sigma_j * (Sigma_i + Sigma_j)^-1 * Sigma_i
                sum_sigma = current_node.Sigma + node_j.Sigma
                try:
                    inv_sum_sigma = np.linalg.inv(sum_sigma)
                except np.linalg.LinAlgError:
                    continue
                    
                current_node.Sigma = node_j.Sigma @ inv_sum_sigma @ current_node.Sigma
                
                # Mu_p = Sigma_j * (Sigma_i + Sigma_j)^-1 * Mu_i + Sigma_i * (Sigma_i + Sigma_j)^-1 * Mu_j
                term_mu1 = node_j.Sigma @ inv_sum_sigma @ current_node.mu
                term_mu2 = current_node.Sigma @ inv_sum_sigma @ node_j.mu
                current_node.mu = term_mu1 + term_mu2
                
                # Update descriptors? The source does not specify how z_p is calculated.
                # Assuming z is inherited or updated by a separate learning process [G]
                # For this implementation, we keep current_node.z as is or average it?
                # Source says "plane descriptor z" is introduced for each primitive.
                # Merging logic for z is not explicitly defined in the provided text.
                # We will assume z is not updated in the merge step itself, 
                # or handled by the "Learning Plane Descriptors" section separately.
                
                # Mark node_j as merged
                node_j.is_merged = True
                
                # Break inner loop as node_i has found a partner and merged
                break
        
        # Add the resulting node to parent nodes
        parent_nodes.append(current_node)
        
    return parent_nodes
```

## Operating characteristics
*   **Complexity:** The algorithm involves nested loops over $N_s$ leaf nodes, resulting in $O(N_s^2)$ comparisons in the worst case. Each comparison requires matrix inversion and determinant calculation, which is $O(1)$ for fixed $3 \times 3$ matrices but adds constant overhead.
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides a hierarchical representation of 3D planes, facilitating semantic understanding and compact scene representation.
*   **Limitations:** Computationally expensive if the number of leaf nodes is large, as raw Gaussian primitives are first clustered into leaves. The quality depends on the accuracy of the plane descriptors $z$ and the thresholds $\epsilon_B, \epsilon_z$.

## Background
The method was introduced in "Planar Gaussian Splatting" [S1]. It addresses the issue that optimizing 3D Gaussian parameters leads to centers moving close to object surfaces, making it difficult to identify distinct planar structures. The approach uses a probabilistic model (Gaussian Mixture Tree) to recursively join nodes from leaves to root, identifying 3D plane instances. Plane descriptors are learned by lifting 2D information (normals from Omnidata, masks from SAM) to 3D.

## Sources
[S1] Planar Gaussian Splatting, https://doi.org/10.1109/wacv61041.2025.00863, fetched 2026-10-26
