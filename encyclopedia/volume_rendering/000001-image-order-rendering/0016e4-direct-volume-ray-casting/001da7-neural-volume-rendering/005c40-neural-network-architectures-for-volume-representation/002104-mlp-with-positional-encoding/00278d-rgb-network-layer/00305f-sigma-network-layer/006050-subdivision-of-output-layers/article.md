# Subdivision of Output Layers

Also called: Subdiv output layers

## Overview
Subdivision of Output Layers is a reparameterization technique used in neural volume rendering to increase the number of depth samples (output layers) during training without altering the final rendered color or density of existing rays [S1, unofficial]. It modifies the last layer of each subnetwork by duplicating and rearranging weight and bias tensors, ensuring that the network's output remains consistent with the previous, coarser resolution [S1, unofficial]. This method is distinct from Subdivision of Input Layers, which preserves input ray colors by modifying the first and last layers to handle increased input dimensionality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Original weight tensor $W_{org}$ of shape $[n_{subnetworks}, d_{out}, d_{in}]$ and bias tensor $B_{org}$ of shape $[n_{subnetworks}, d_{out}]$.
*   **Output:** Modified weight tensor $W_{new}$ and bias tensor $B_{new}$ with the same shapes as the inputs, but with reparameterized values.

**Data Structures:**
The method operates on PyTorch-style tensor layouts where dimensions are explicitly split and concatenated.
*   $n_{subnetworks}$: Number of subnetworks.
*   $d_{out}$: Original output dimension (number of depth samples/channels).
*   $d_{in}$: Input dimension.

**Steps:**
1.  **Bias Rearrangement:** Split the output dimension $d_{out}$ of the bias tensor into $c=4$ channels. The shape $[n_{subnetworks}, d_{out}]$ becomes $[n_{subnetworks}, d_{out}/4, 4]$.
2.  **Bias Duplication:** Concatenate the rearranged bias with itself along the channel dimension (dim=2). The shape becomes $[n_{subnetworks}, d_{out}/4, 8]$.
3.  **Bias Reshape:** Rearrange the concatenated bias back to the original output dimension structure $[n_{subnetworks}, d_{out}]$.
4.  **Weight Rearrangement:** Split the output dimension $d_{out}$ of the weight tensor into $c=2$ channels. The shape $[n_{subnetworks}, d_{out}, d_{in}]$ becomes $[n_{subnetworks}, d_{out}/2, 2, d_{in}]$.
5.  **Weight Duplication:** Concatenate the rearranged weight with itself along the channel dimension (dim=2). The shape becomes $[n_{subnetworks}, d_{out}/2, 4, d_{in}]$.
6.  **Weight Reshape:** Rearrange the concatenated weight back to the original output dimension structure $[n_{subnetworks}, d_{out}, d_{in}]$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subdiv_output_layers(org_weight, org_bias):
    """
    Reparameterizes the last layer of neural subnetworks to subdivide output layers.
    
    Args:
        org_weight: Tensor of shape [n_subnetworks, d_out, d_in]
        org_bias: Tensor of shape [n_subnetworks, d_out]
        
    Returns:
        weight: Modified weight tensor of shape [n_subnetworks, d_out, d_in]
        bias: Modified bias tensor of shape [n_subnetworks, d_out]
    """
    # Step 1-3: Subdivide Bias
    # Rearrange bias from [n, d_out] to [n, d_out/4, 4]
    # Note: d_out must be divisible by 4
    bias = rearrange(org_bias, 'n (d c) -> n d c', c=4)
    # Concatenate with itself along dim=2 (channel dimension)
    bias = concatenate([bias, bias], dim=2)
    # Rearrange back to [n, d_out]
    bias = rearrange(bias, 'n d c -> n (d c)')
    
    # Step 4-6: Subdivide Weight
    # Rearrange weight from [n, d_out, d_in] to [n, d_out/2, 2, d_in]
    # Note: d_out must be divisible by 2
    weight = rearrange(org_weight, 'n (d c) i -> n d c i', c=2)
    # Concatenate with itself along dim=2 (channel dimension)
    weight = concatenate([weight, weight], dim=2)
    # Rearrange back to [n, d_out, d_in]
    weight = rearrange(weight, 'n d c i -> n (d c) i')
    
    return weight, bias
```

**Parameters:**
*   $c=4$ for bias splitting.
*   $c=2$ for weight splitting.
*   These constants are fixed in the source description [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Divisibility:** The original output dimension $d_{out}$ must be divisible by 4 for the bias operation and by 2 for the weight operation. If $d_{out}$ is not divisible by 4, the bias rearrangement will fail [S1, unofficial].
*   **Precision:** The method relies on exact tensor reshaping and concatenation. No averaging or scaling factors are applied to the output layers, unlike the input layer subdivision which uses a factor of 0.5 [S1, unofficial].

## Subtopics

### Subdivision of Input Layers
A reparameterization technique to maintain unchanged input ray colors after depth subdivision by modifying the weight matrices and bias vectors of the first and last layer functions of the networks [S1, unofficial]. It splits the input dimension $d_{in}$ into $c=2$ channels, duplicates the weights, and averages them by multiplying by 0.5 to ensure hidden features remain unaffected [S1, unofficial]. The bias remains unchanged. This validates that the color of each input ray is kept unchanged [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The operation is $O(n_{subnetworks} \cdot d_{out} \cdot d_{in})$ for weights and $O(n_{subnetworks} \cdot d_{out})$ for biases, as it involves tensor rearrangement and concatenation. Memory usage remains constant as the output tensor shapes are identical to the input shapes.

**Measured Performance:**
Not given in the sources.

**Scaling:**
The method allows for progressive refinement of depth samples ($D_k = D \times 2^k$) during training stages [S1, unofficial]. It scales with the number of subnetworks and the depth resolution.

**Strengths and Limitations:**
*   **Strengths:** Ensures consistency of rendered colors and densities when increasing depth resolution, preventing artifacts from sudden changes in sampling density [S1, unofficial].
*   **Limitations:** Requires $d_{out}$ to be divisible by 4 for bias operations. Only applies to the last layer of the subnetworks [S1, unofficial].

**When to use it and when not:**
Use when progressively increasing the number of depth samples in a neural volume rendering pipeline to improve rendering quality without disrupting the learned representation [S1, unofficial]. Do not use if the output dimension is not divisible by 4 or if the goal is to modify input ray properties rather than output depth samples.

## Background
Originated in the "Progressively-connected Light Field Network for Efficient View Synthesis" paper (2022) [S1, unofficial]. The technique is part of a broader strategy involving progressive connection and depth subdivision to enhance rendering quality in neural view synthesis [S1, unofficial]. It is used in conjunction with consistency losses for regularization [S1, unofficial].

## Sources
[S1] Progressively-connected Light Field Network for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2207.04465, fetched 2026-10-27 (unofficial)
