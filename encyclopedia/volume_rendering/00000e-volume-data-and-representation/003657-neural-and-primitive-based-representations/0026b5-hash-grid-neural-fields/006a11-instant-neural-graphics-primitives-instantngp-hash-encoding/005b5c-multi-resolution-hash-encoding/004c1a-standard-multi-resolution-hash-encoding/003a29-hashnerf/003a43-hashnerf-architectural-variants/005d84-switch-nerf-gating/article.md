# Switch-NeRF++ Gating
Also called: MLP-based gating
## Overview
Switch-NeRF++ Gating is a mechanism for selecting a single neural expert from a collection of heterogeneous experts for a given 3D point in large-scale neural radiance fields. It solves the efficiency bottleneck of traditional Mixture of Experts (MoE) routing by integrating point dispatching directly into the hash operation, avoiding extra memory for reorganizing inputs and eliminating the need for a capacity factor. It sets itself apart from its sibling [Switch-NeRF++ Forward Pass and Loss](17395-switch-nerf-forward-pass-and-loss.md) by focusing specifically on the gating network's architecture, and from [GP-NeRF](17415-gp-nerf.md) by using a deterministic Top-1 selection rather than probabilistic mixing. It differs from its parent's general variants by specifically employing hash encoding for the gating decision to align with the hash experts.
## How it works
**Inputs and Outputs:**
- Input: A 3D point coordinate $x \in \mathbb{R}^3$.
- Output: An index $i$ identifying the selected expert $E_i$, or the final rendered value $\hat{E}(x)$.

**Data Structures:**
- **Experts:** A set of $n$ neural networks $\{E_i\}_{i=1}^n$. In Switch-NeRF++, these are typically heterogeneous hash experts.
- **Gating Network:** A network that maps the input to a probability distribution over the $n$ experts.

**Method:**
The gating mechanism determines which expert processes the input point $x$. The core formula for the gate value $G(x)_i$ for expert $i$ is:
$$ G(x)_i = \text{Top-1}(\text{Softmax}(\text{MLP}(h))) $$
where $h$ is the hash encoding of $x$, denoted as $\text{Hash}(x)$.

1. **Hash Encoding:** Compute the hash encoding $h = \text{Hash}(x)$ of the input point $x$. This maps the continuous coordinate to a discrete set of hash table entries.
2. **MLP Processing:** Pass the hash encoding $h$ through a Multi-Layer Perceptron (MLP). The MLP outputs a vector of logits.
3. **Softmax Normalization:** Apply the Softmax function to the MLP outputs to normalize them into an $n$-dimensional probability distribution.
4. **Top-1 Selection:** Select the expert with the highest probability. The gate value $G(x)_i$ is 1 for the selected expert and 0 for others.
   $$ i^* = \arg\max_i G(x)_i $$
5. **Expert Selection:** The final output is computed by the selected expert:
   $$ \hat{E}(x) = E_{i^*}(x) $$

**Parameters:**
- $n$: The number of experts. Not given in the sources.
- MLP architecture: The specific layers and dimensions of the gating MLP are not detailed in the provided notes, but it is described as "much more efficient than a wide MLP head" [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The use of hash encoding for gating ensures consistency with the hash experts, potentially reducing misalignment errors [S1].
- **Degenerate Input:** The Top-1 selection is deterministic, avoiding the ambiguity of soft routing.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def switch_nerfpp_gating(x, hash_encoding_func, gating_mlp, experts):
    """
    Selects an expert for a 3D point x using Switch-NeRF++ gating.
    
    Args:
        x: 3D point coordinate (tensor of shape [3]).
        hash_encoding_func: Function to compute hash encoding of x.
        gating_mlp: MLP network for gating.
        experts: List of expert networks.
        
    Returns:
        The output of the selected expert.
    """
    # Step 1: Hash Encoding
    h = hash_encoding_func(x)
    
    # Step 2: MLP Processing
    logits = gating_mlp(h)
    
    # Step 3: Softmax Normalization
    probabilities = softmax(logits)
    
    # Step 4: Top-1 Selection
    selected_expert_index = argmax(probabilities)
    
    # Step 5: Expert Selection
    selected_expert = experts[selected_expert_index]
    output = selected_expert(x)
    
    return output
```
## Subtopics

### [Hash-based Gating](17446-hash-based-gating.md)
Hash-based Gating is a mechanism for selecting one of $N$ hash experts for a given 3D point $x$ by integrating point dispatching directly into the hash operation. It solves the efficiency bottleneck of traditional Mixture of Experts (MoE) routing in large-scale neural radiance fields by avoiding extra memory for reorganizing inputs and eliminating the need for a capacity factor. It sets itself apart from MLP-based gating (used in Switch-NeRF) by providing sufficient network capacity for accurate selection while reducing training time and memory usage.

## Operating characteristics
- **Time Complexity:** More efficient than a wide MLP head used in previous works [S1].
- **Memory Complexity:** Reduces training time and memory usage compared to traditional MoE routing [S1].
- **Accuracy:** Produces clearly higher accuracy than Switch-NeRF [S1].
- **Strengths:** Efficiency, consistency with hash experts, joint optimization of experts and gating network [S1].
- **Limitations:** Not given in the sources.
## Background
Switch-NeRF++ was introduced to address the challenges of representing large-scale scenes with neural radiance fields. The use of hash encoding for gating in Switch-NeRF++ instead of a pure MLP (as in Switch-NeRF) provides two advantages: efficiency and consistency with hash experts, leading to higher accuracy [S1]. The framework enables joint optimization of both the experts and the gating network during backpropagation, allowing the system to inherently learn scene decomposition through direct gradient-based training [S1].
## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
