# DLR-NeRF Construction

Also called: DLR-NeRF Initialization, Distilled Low Rank Neural Radiance Field, Network Distillation for DLR-NeRF, Network Distillation via Tensor Train (TT) Decomposition

## Overview
DLR-NeRF Construction is a network distillation method that transforms a low-rank NeRF (LR-NeRF) into a distilled, slimmer network (DLR-NeRF) by decomposing its weight tensors into Tensor Train (TT) components [S1, unofficial]. It solves the problem of preserving low-rank properties during subsequent weight quantization, which would otherwise break the rank constraint during fine-tuning [S1, unofficial]. Unlike its siblings, it acts as a bridge between low-rank approximation and quantization, ensuring the distilled network maintains the structural benefits of low-rank optimization while enabling efficient compression [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** The weight matrices $\mathbf{W}_r^i$ and biases $\mathbf{b}_i$ of each layer $L_i$ in a pre-trained LR-NeRF, where $\mathbf{W}_r^i \in \mathbb{R}^{n_1 \times n_2}$ has been reduced to rank $r$ [S1, unofficial].
*   **Output:** A DLR-NeRF model consisting of layers initialized with TT components, ready for fine-tuning [S1, unofficial].

**Data Structures**
*   **TT Components:** Each weight matrix $\mathbf{W}_r^i$ is decomposed into two tensors $\mathbf{Q}_{1i}$ and $\mathbf{Q}_{2i}$ such that $\mathbf{W}_r^i = \mathbf{Q}_{1i} \cdot \mathbf{Q}_{2i}$ [S1, unofficial].
    *   $\mathbf{Q}_{1i} \in \mathbb{R}^{n_1 \times r}$
    *   $\mathbf{Q}_{2i} \in \mathbb{R}^{r \times n_2}$
*   **DLR-NeRF Layers:**
    *   If parameter reduction is achieved, the layer is split into two cascaded fully-connected layers: $L'_{i1}$ with weights $\mathbf{Q}_{1i}$ and $L'_{i2}$ with weights $\mathbf{Q}_{2i}$ and bias $\mathbf{b}_i$ [S1, unofficial].
    *   If parameter reduction is not achieved, the layer remains a single fully-connected layer $L'_i$ with weights $\mathbf{W}_r^i$ and bias $\mathbf{b}_i$ [S1, unofficial].

**Steps**
1.  **Decomposition:** For each layer $i$ in the LR-NeRF, decompose the low-rank weight matrix $\mathbf{W}_r^i$ into TT components $\mathbf{Q}_{1i}$ and $\mathbf{Q}_{2i}$ [S1, unofficial].
2.  **Parameter Count Check:** Calculate the number of parameters in the TT components versus the original weight matrix.
    *   TT parameters: $(n_1 + n_2) \times r$
    *   Original parameters: $n_1 \times n_2$
3.  **Layer Initialization:**
    *   **Case 1 (Split):** If $(n_1 + n_2) \times r < n_1 \times n_2$, initialize two cascaded fully-connected layers $L'_{i1}$ and $L'_{i2}$ in the DLR-NeRF. Set the weights of $L'_{i1}$ to $\mathbf{Q}_{1i}$ and the weights of $L'_{i2}$ to $\mathbf{Q}_{2i}$ with bias $\mathbf{b}_i$ [S1, unofficial].
    *   **Case 2 (Keep):** If $(n_1 + n_2) \times r \ge n_1 \times n_2$, initialize one layer $L'_i$ in the DLR-NeRF with the original weight matrix $\mathbf{W}_r^i$ and bias $\mathbf{b}_i$ [S1, unofficial].
4.  **Fine-tuning:** Fine-tune the DLR-NeRF using the same loss function as the original LR-NeRF training (Eq. 2 in [S1, unofficial]) to update its parameters [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_dlr_nerf(lr_nerf):
    """
    Constructs DLR-NeRF from LR-NeRF by decomposing weights into TT components.
    
    Args:
        lr_nerf: An object containing layers with attributes .weight (W_r) and .bias (b).
                 W_r is a 2D tensor of shape (n1, n2) with rank r.
    
    Returns:
        dlr_nerf: A new network structure with layers initialized according to TT decomposition.
    """
    dlr_nerf = NeuralNetwork()
    
    for layer in lr_nerf.layers:
        W_r = layer.weight  # Shape: (n1, n2)
        b = layer.bias      # Shape: (n2,)
        n1, n2 = W_r.shape
        r = W_r.rank        # Target rank of the low-rank approximation
        
        # Decompose W_r into TT components Q1 and Q2
        # W_r = Q1 @ Q2
        Q1, Q2 = decompose_tt(W_r, r)
        
        # Check parameter count
        # TT params: (n1 + n2) * r
        # Original params: n1 * n2
        tt_params = (n1 + n2) * r
        orig_params = n1 * n2
        
        if tt_params < orig_params:
            # Split into two cascaded layers
            # Layer 1: weights Q1, no bias (bias is applied in Layer 2)
            layer_1 = FullyConnectedLayer(weights=Q1)
            # Layer 2: weights Q2, bias b
            layer_2 = FullyConnectedLayer(weights=Q2, bias=b)
            
            dlr_nerf.add_layer(layer_1)
            dlr_nerf.add_layer(layer_2)
        else:
            # Keep as single layer with original low-rank weights
            layer = FullyConnectedLayer(weights=W_r, bias=b)
            dlr_nerf.add_layer(layer)
            
    return dlr_nerf

def decompose_tt(W, r):
    """
    Decomposes a matrix W into two tensors Q1 and Q2 such that W = Q1 @ Q2.
    This assumes W is already low-rank with rank r.
    
    Args:
        W: Matrix of shape (n1, n2).
        r: Target rank.
        
    Returns:
        Q1: Matrix of shape (n1, r).
        Q2: Matrix of shape (r, n2).
    """
    # Perform low-rank decomposition (e.g., SVD)
    # W = U @ S @ V^T
    # Q1 = U @ sqrt(S)
    # Q2 = sqrt(S) @ V^T
    U, S, Vt = torch.linalg.svd(W, full_matrices=False)
    
    # Keep top r components
    U_r = U[:, :r]
    S_r = torch.diag(S[:r])
    Vt_r = Vt[:r, :]
    
    # Construct Q1 and Q2
    # Q1 has shape (n1, r)
    Q1 = U_r @ torch.sqrt(S_r)
    # Q2 has shape (r, n2)
    Q2 = torch.sqrt(S_r) @ Vt_r
    
    return Q1, Q2
```

**Parameters**
*   **Rank $r$:** The target rank for the low-rank approximation. Typical values depend on the desired compression ratio and reconstruction quality [S1, unofficial].
*   **Loss Function:** The same loss function used in the LR-NeRF training (Eq. 2 in [S1, unofficial]), typically combining photometric loss and regularization terms [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Training from Scratch:** Training a distilled NeRF from scratch with random weights yields poor reconstruction quality because the weight distribution differs from low-rank weights [S1, unofficial].
*   **Parameter Count Threshold:** If the TT decomposition does not reduce the parameter count (i.e., $(n_1 + n_2) \times r \ge n_1 \times n_2$), the layer is kept as is to avoid unnecessary complexity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The decomposition step is dominated by SVD, which is $O(n_1 n_2 \min(n_1, n_2))$ per layer. The fine-tuning step has the same complexity as standard NeRF training.
*   **Memory Complexity:** The DLR-NeRF has fewer parameters than the original LR-NeRF if the TT decomposition condition is met, reducing memory footprint [S1, unofficial].
*   **Strengths:** Preserves low-rank properties during quantization, enabling efficient compression without degrading reconstruction quality [S1, unofficial].
*   **Limitations:** Requires a pre-trained LR-NeRF; not suitable for direct training from scratch [S1, unofficial].
*   **When to Use:** When compressing NeRFs via quantization while maintaining low-rank structure is required [S1, unofficial].
*   **When Not to Use:** When high-quality reconstruction is needed without compression constraints, or when training from scratch is preferred [S1, unofficial].

## Background
DLR-NeRF Construction was introduced by J. Shi et al. in "Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression" [S1, unofficial]. It is part of the QDLR-NeRF pipeline, which combines low-rank approximation, network distillation, and quantization for light field compression [S1, unofficial]. The method bridges the gap between low-rank optimization and quantization, ensuring that the distilled network retains the benefits of low-rank representation [S1, unofficial].

## Sources
[S1] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
