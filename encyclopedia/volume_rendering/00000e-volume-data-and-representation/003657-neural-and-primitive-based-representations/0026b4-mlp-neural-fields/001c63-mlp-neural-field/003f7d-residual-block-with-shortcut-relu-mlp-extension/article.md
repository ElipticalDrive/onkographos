# Residual Block with Shortcut (ReLU MLP Extension)
Also called: Analytic Marching with TResBlk and Shortcut Connections
## Overview
This method extends the analytic marching algorithm for implicit surface reconstruction to Multi-Layer Perceptrons (MLPs) constructed with residual blocks containing shortcut connections [S1]. It enables the theoretical analysis and exact linear region partitioning of such architectures, allowing for efficient meshing via analytic marching [S1].
## How it works
**Inputs and Outputs:**
- Input: An MLP $T$ composed of $L$ residual blocks with shortcut connections, where each block maps $\mathbb{R}^{n_0} \to \mathbb{R}^{n_L}$ (with $n_0 = n_L$ for the primary case) [S1].
- Output: The region-wise linear mapping coefficients $a_{lk}^r$ and the state functional $s_{\text{ResBlk}}(r)$ required for analytic marching [S1].

**Data Structures:**
- State functional $s_{\text{ResBlk}}(\mathbf{x})$: A vector containing the binary activation states of all neurons in all layers [S1].
- Region-wise linear mapping $T_{\text{ResBlk}}^r$: A matrix representing the linear transformation within a specific linear region $r$ [S1].
- Neuron-wise coefficients $a_{lk}^r$: Scalars defining the contribution of each neuron to the output [S1].

**Definitions:**
Let $\mathbf{x} \in \mathbb{R}^{n_0}$ be the input [S1]. The residual block defines a mapping $T_{\text{ResBlk}}$ as [S1]:
$$ T_{\text{ResBlk}}\mathbf{x} = g(\mathbf{x} + T_{\text{PreAct}}\mathbf{x}) $$
where $g$ is the ReLU activation function, and $T_{\text{PreAct}}$ is a stack of $L$ hidden layers [S1]:
$$ T_{\text{PreAct}}\mathbf{x} = \mathbf{W}_L g(\dots g(\mathbf{W}_1 \mathbf{x})) $$
Here, $\mathbf{W}_l \in \mathbb{R}^{n_l \times n_{l-1}}$ are the weight matrices for layer $l$, and $n_0 = n_L$ [S1].

**Step 1: Define State Functional for Final Layer**
The state functional for the output of the final layer of $T_{\text{PreAct}}$, denoted $s_L(\mathbf{x})$, is defined element-wise for $k \in \{1, \dots, n_L\}$ as [S1]:
$$ s_{Lk}(\mathbf{x}) = \begin{cases} 1 & \text{if } \pi_k(\mathbf{x} + T_{\text{PreAct}}\mathbf{x}) > 0 \\ 0 & \text{if } \pi_k(\mathbf{x} + T_{\text{PreAct}}\mathbf{x}) \leq 0 \end{cases} $$
where $\pi_k(\mathbf{v})$ extracts the $k$-th component of vector $\mathbf{v}$ [S1]. The full state functional for the residual block is $s_{\text{ResBlk}}(\mathbf{x}) = [s_1(\mathbf{x})^T, \dots, s_L(\mathbf{x})^T]^T$ [S1].

**Step 2: Define Region-wise Linear Mapping**
For a linear region $r$ labeled by $s_{\text{ResBlk}}(r)$, the region-wise linear mapping $T_{\text{ResBlk}}^r$ is defined as [S1]:
$$ T_{\text{ResBlk}}^r = \text{diag}(s_L(r)) \left( \mathbf{I} + \mathbf{W}_L \prod_{l=1}^{L-1} \mathbf{W}_l^r \right) $$
where $\mathbf{I}$ is the identity matrix of size $n_L \times n_L$, and $\mathbf{W}_l^r$ is the modified weight matrix for layer $l$ defined as [S1]:
$$ \mathbf{W}_l^r = \text{diag}(s_l(r)) \mathbf{W}_l \quad \text{for } l \in \{1, \dots, L-1\} $$

**Step 3: Define Neuron-wise Linear Mapping Coefficients**
The neuron-wise linear mapping coefficients $a_{lk}^r$ for layer $l$ and neuron $k$ are defined as [S1]:
$$ a_{lk}^r = \begin{cases} \pi_k \left( \mathbf{I} + \mathbf{W}_L \prod_{i=1}^{L-1} \mathbf{W}_i^r \right) & \text{when } l = L \\ \pi_k \left( \mathbf{W}_l \prod_{i=1}^{l-1} \mathbf{W}_i^r \right) & \text{when } l \in \{2, \dots, L-1\} \\ \pi_k \mathbf{W}_l & \text{when } l = 1 \end{cases} $$
where $\pi_k(\mathbf{M})$ extracts the $k$-th row of matrix $\mathbf{M}$ [S1].

**Step 4: Alternative Residual Block (Dimension Mismatch)**
If the input dimension $n_0$ differs from the output dimension $n_L$, the shortcut path is replaced by a linear mapping $\mathbf{V}\mathbf{x}$, where $\mathbf{V} \in \mathbb{R}^{n_L \times n_0}$ [S1]. The mapping becomes [S1]:
$$ T_{\text{ResBlk}}\mathbf{x} = g(\mathbf{V}\mathbf{x} + T_{\text{PreAct}}\mathbf{x}) $$
Definitions similar to those above apply, with $\mathbf{I}$ replaced by $\mathbf{V}$ in the relevant terms [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_residual_block_state(x, weights, num_layers):
    """
    Compute the state functional for a residual block with shortcut connections.
    Args:
        x: Input vector of shape (n0,)
        weights: List of weight matrices [W1, W2, ..., WL]
        num_layers: Number of hidden layers L
    Returns:
        s_resblk: State functional vector containing states for all layers
    """
    current_input = x
    layer_states = []
    
    for l in range(num_layers - 1):
        W_l = weights[l]
        linear_out = W_l @ current_input
        current_input = np.maximum(0, linear_out)
        state = (current_input > 0).astype(float)
        layer_states.append(state)
        
    W_L_pre = weights[num_layers - 1]
    linear_out_L = W_L_pre @ current_input
    pre_act_output = linear_out_L
    
    shortcut_sum = x + pre_act_output
    s_L = (shortcut_sum > 0).astype(float)
    
    s_resblk = np.concatenate([np.array(layer_states), [s_L]])
    return s_resblk

def compute_region_wise_mapping(s_resblk, weights, num_layers):
    """
    Compute the region-wise linear mapping T_ResBlk^r for a given region r.
    Args:
        s_resblk: State functional vector for region r
        weights: List of weight matrices [W1, ..., WL]
        num_layers: Number of hidden layers L
    Returns:
        T_resblk_r: Region-wise linear mapping matrix
    """
    nL = weights[-1].shape[0]
    I = np.eye(nL)
    
    modified_weights = []
    for l in range(num_layers - 1):
        s_l = s_resblk[l]
        W_l = weights[l]
        W_l_r = np.diag(s_l) @ W_l
        modified_weights.append(W_l_r)
        
    P = modified_weights[0]
    for i in range(1, len(modified_weights)):
        P = P @ modified_weights[i]
        
    s_L = s_resblk[num_layers - 1]
    W_L = weights[num_layers - 1]
    
    T_resblk_r = np.diag(s_L) @ (I + W_L @ P)
    return T_resblk_r

def compute_neuron_wise_coefficients(s_resblk, weights, num_layers):
    """
    Compute neuron-wise linear mapping coefficients a_{lk}^r.
    Args:
        s_resblk: State functional vector for region r
        weights: List of weight matrices [W1, ..., WL]
        num_layers: Number of hidden layers L
    Returns:
        coefficients: Dict mapping (l, k) to row vector a_{lk}^r
    """
    coefficients = {}
    nL = weights[-1].shape[0]
    I = np.eye(nL)
    
    modified_weights = []
    for l in range(num_layers - 1):
        s_l = s_resblk[l]
        W_l = weights[l]
        modified_weights.append(np.diag(s_l) @ W_l)
        
    P_full = modified_weights[0]
    for i in range(1, len(modified_weights)):
        P_full = P_full @ modified_weights[i]
        
    for l in range(1, num_layers + 1):
        if l == 1:
            M = weights[0]
        elif l == num_layers:
            M = I + weights[-1] @ P_full
        else:
            P_partial = modified_weights[0]
            for i in range(1, l - 1):
                P_partial = P_partial @ modified_weights[i]
            M = weights[l - 1] @ P_partial
            
        for k in range(M.shape[0]):
            coefficients[(l, k)] = M[k, :]
    return coefficients
```

**Parameters:**
- $L$: Number of hidden layers in the residual block [S1].
- $n_l$: Number of neurons in layer $l$ [S1].
- $\mathbf{W}_l$: Weight matrices [S1].
- $\mathbf{V}$: Linear mapping matrix for dimension mismatch (if used) [S1].

**Edge Cases and Pitfalls:**
- Dimension Mismatch: If $n_0 \neq n_L$, the shortcut path must be replaced by a linear mapping $\mathbf{V}\mathbf{x}$ [S1].

## Operating characteristics
Not given in the sources.

## Background
This method was introduced in the paper "Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching" [S1]. It extends the analytic marching algorithm, originally developed for standard MLPs, to architectures with residual blocks and shortcut connections [S1].

## Sources
[S1] Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching, https://doi.org/10.1109/tpami.2021.3135007, fetched 2026-10-26
