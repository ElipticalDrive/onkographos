# Switch-NeRF++ Gating
Also called: MLP-based gating
## Overview
Switch-NeRF++ Gating selects one of $n$ neural experts for a given 3D point $x$ by using a gating network based on hash encoding [S1]. It sets itself apart from [Switch-NeRF++ Forward Pass and Loss](17395-switch-nerf-forward-pass-and-loss.md) by defining the gating architecture, and from [GP-NeRF](17415-gp-nerf.md) by employing deterministic Top-1 routing [G].
## How it works
**Inputs and Outputs:**
- Input: A 3D point coordinate $x \in \mathbb{R}^3$ [S1].
- Output: The final rendered value $\hat{E}(x)$ from the selected expert [S1].

**Data Structures:**
- **Experts:** A set of $n$ neural networks $\{E_i\}_{i=1}^n$ [S1].
- **Gating Network:** A network that maps the input to a probability distribution over the $n$ experts [S1].

**Method:**
The gating mechanism determines which expert processes the input point $x$ [S1]. The core formula for the gate value $G(x)_i$ for expert $i$ is:
$$ G(x)_i = \text{Top-1}(\text{Softmax}(\text{MLP}(\text{Hash}(x)))) $$
where $\text{Hash}(x)$ is the hash encoding of $x$ [S1].

1. **Hash Encoding:** Compute the hash encoding $h = \text{Hash}(x)$ of the input point $x$ [S1].
2. **MLP Processing:** Pass the hash encoding $h$ through a Multi-Layer Perceptron (MLP) to produce a vector of logits [S1].
3. **Softmax Normalization:** Apply the Softmax function to the MLP outputs to normalize them into an $n$-dimensional probability distribution [S1].
4. **Top-1 Selection:** Select the expert with the highest probability [S1]. The gate value $G(x)_i$ is 1 for the selected expert and 0 for others [S1].
   $$ i^* = \arg\max_i G(x)_i $$
5. **Expert Selection:** The final output is computed by the selected expert [S1].
   $$ \hat{E}(x) = E_{i^*}(x) $$

**Parameters:**
- $n$: The number of experts. Not given in the sources.
- MLP architecture: The specific layers and dimensions of the gating MLP are not detailed in the provided notes, but it is described as "much more efficient than a wide MLP head" [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The use of hash encoding for gating ensures consistency with the hash experts, potentially reducing misalignment errors [S1].
- **Degenerate Input:** The Top-1 selection is deterministic, avoiding the ambiguity of soft routing [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

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
    max_l = max(logits)
    exp_l = [math.exp(l - max_l) for l in logits]
    sum_exp = sum(exp_l)
    probabilities = [e / sum_exp for e in exp_l]
    
    # Step 4: Top-1 Selection
    selected_expert_index = max(range(len(probabilities)), key=lambda i: probabilities[i])
    
    # Step 5: Expert Selection
    selected_expert = experts[selected_expert_index]
    output = selected_expert(x)
    
    return output
```
## Subtopics          

### [Hash-based Gating](17446-hash-based-gating.md)
Hash-based gating is a mechanism for selecting one of $N$ hash experts for a given 3D point $x$ by integrating the dispatching logic directly into the hash table lookup operation. It solves the efficiency bottleneck of traditional Mixture-of-Experts (MoE) routing by eliminating explicit input feature reorganization and the associated matrix multiplications. Unlike MLP-based gating used in Switch-NeRF, it reduces training time and memory overhead while maintaining or improving rendering quality. Choose it when minimizing routing overhead is critical for large-scale scenes.

## Operating characteristics
- **Time Complexity:** More efficient than a wide MLP head used in previous works [S1].
- **Accuracy:** Produces clearly higher accuracy than Switch-NeRF [S1].
- **Strengths:** Efficiency, consistency with hash experts, and joint optimization of experts and gating network during backpropagation [S1].
- **Limitations:** Not given in the sources.
- **When to use:** Large-scale scenes requiring efficient expert routing [S1].
## Background
Switch-NeRF++ was introduced to address the challenges of representing large-scale scenes with neural radiance fields [S1]. The use of hash encoding for gating in Switch-NeRF++ instead of a pure MLP (as in Switch-NeRF) provides two advantages: efficiency and consistency with hash experts, leading to higher accuracy [S1]. The framework enables joint optimization of both the experts and the gating network during backpropagation, allowing the system to inherently learn scene decomposition through direct gradient-based training [S1].
## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
