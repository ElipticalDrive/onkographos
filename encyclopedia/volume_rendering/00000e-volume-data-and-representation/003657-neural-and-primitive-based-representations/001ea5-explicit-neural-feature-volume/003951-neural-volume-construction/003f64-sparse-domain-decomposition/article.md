# Sparse Domain Decomposition

Also called: MoE-inspired decomposition, MoE-inspired domain decomposition, Sparsely-gated Mixture-of-Experts inspired decomposition

## Overview
Sparse Domain Decomposition is a technique for handling large-scale or disjoint volumetric scenes by partitioning the domain into fixed-size subdomains, each managed by a dedicated neural network (expert) [S1][S2, unofficial]. It solves the performance degradation caused by normalizing input coordinates to $[0, 1]$ in large domains, where local features become incoherent [S1][S2, unofficial]. Unlike standard Mixture-of-Experts (MoE), the gating mechanism is fixed and non-learnable, relying on geometric overlap rather than learned selection [S1][S2, unofficial].

## How it works
**Inputs:**
*   A 3D scene domain.
*   A target subdomain size in index space (configurable, typically 512–2048) [S1][S2, unofficial].
*   A halo size (typically 8 voxels) [S1][S2, unofficial].

**Outputs:**
*   A set of active subdomains, each associated with a trained expert network $E_k$ and a gate function $G_k$ [S1][S2, unofficial].
*   The reconstructed value $\hat{y}$ (e.g., child/active masks or voxel values) for any query coordinate $\mathbf{x}$ [S1][S2, unofficial].

**Data Structures:**
*   **Subdomain Grid:** A regular grid partitioning the domain. Empty subdomains are discarded [S1][S2, unofficial].
*   **Expert Network ($E_k$):** A neural network with identical architecture for all subdomains $k$ [S1][S2, unofficial].
*   **Gate Function ($G(\mathbf{x})_k$):** A fixed clamped tent function for subdomain $k$ [S1][S2, unofficial].

**Algorithm:**

1.  **Partitioning:** Divide the entire domain into a regular grid of subdomains $D_k$. Discard any subdomains that contain no data (empty) [S1][S2, unofficial].
2.  **Expert Assignment:** Define a dedicated neural network $E_k$ for each remaining subdomain $D_k$. Use the same architecture for all experts [S1][S2, unofficial].
3.  **Gate Definition:** For each subdomain $D_k$, define a gate function $G(\mathbf{x})_k$. The function is a clamped tent function with a maximum value of 1 uniformly outside the overlapping region. It covers the subdomain $D_k$ including its halo [S1][S2, unofficial].
4.  **Halo Configuration:** Assign a fixed-width halo of 8 voxels to each subdomain. This halo overlaps with adjacent subdomains to eliminate discontinuities while minimizing compute overhead [S1][S2, unofficial].
5.  **Inference:** For a query coordinate $\mathbf{x}$:
    *   Evaluate all gate functions $G(\mathbf{x})_k$.
    *   Identify active subdomains where $G(\mathbf{x})_k > 0$. Due to halos, up to eight subdomains may be active simultaneously [S1].
    *   Optimization: Examine gate values first; only invoke expert networks $E_k$ for subdomains with non-zero gate values [S1].
    *   Compute the output $\hat{y}$ using the weighted sum formula:
        $$ \hat{y} = \sum_{k=1}^n G(\mathbf{x})_k E_k(\mathbf{x}) $$
        where $n$ is the number of subdomains, $E_k(\mathbf{x})$ is the expert output, and $G(\mathbf{x})_k$ is the gate weight [S1][S2, unofficial].
    *   Note: The sources state that multiple gate functions may return non-zero values and the evaluated values are averaged weighted by the gate functions [S1]. The formula provided sums the products, implying the gate functions themselves may be normalized or the summation represents the weighted combination [S1][S2, unofficial].
6.  **Training:** Train experts concurrently on multiple GPUs. Assign groups of subdomains to each GPU. Multiple subdomains can be assigned to a single GPU [S1].
7.  **Post-Training:** Merge groups of subdomains into a single NeuralVDB structure [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def clamped_tent_gate(x_normalized, subdomain_min, subdomain_max, halo_size, subdomain_size):
    """
    Computes the gate value G(x)_k for a normalized coordinate x_normalized
    relative to a subdomain.
    
    Args:
        x_normalized: float, coordinate normalized to [0, 1] within the subdomain's bounding box.
        subdomain_min: float, start of the subdomain in normalized coordinates.
        subdomain_max: float, end of the subdomain in normalized coordinates.
        halo_size: float, size of the halo in normalized coordinates.
        subdomain_size: float, size of the subdomain in normalized coordinates.
        
    Returns:
        float: Gate value between 0 and 1.
    """
    # Define the core region (excluding halo)
    core_min = subdomain_min + halo_size
    core_max = subdomain_max - halo_size
    
    # Define the extended region (including halo)
    ext_min = subdomain_min
    ext_max = subdomain_max
    
    if x_normalized < ext_min or x_normalized > ext_max:
        return 0.0
    
    # Clamped tent function: 1 in the core, linearly decreasing in the halo
    if core_min <= x_normalized <= core_max:
        return 1.0
    elif ext_min <= x_normalized < core_min:
        # Linear ramp up from 0 to 1 over the halo width
        return (x_normalized - ext_min) / (core_min - ext_min)
    elif core_max < x_normalized <= ext_max:
        # Linear ramp down from 1 to 0 over the halo width
        return (ext_max - x_normalized) / (ext_max - core_max)
    else:
        return 0.0

def sparse_domain_decomposition_inference(query_coords, subdomains):
    """
    Reconstructs volume values using Sparse Domain Decomposition.
    
    Args:
        query_coords: list of tuples (x, y, z), normalized coordinates in [0, 1].
        subdomains: list of dicts, each containing:
            - 'id': int
            - 'min': tuple (x, y, z) normalized start
            - 'max': tuple (x, y, z) normalized end
            - 'halo': float (normalized size)
            - 'expert': callable E_k(x_normalized)
            
    Returns:
        list of floats: Reconstructed values for each query coordinate.
    """
    results = []
    
    for coord in query_coords:
        x, y, z = coord
        weighted_sum = 0.0
        gate_sum = 0.0
        
        # Optimization: Check gates first to determine which experts to invoke
        for sub in subdomains:
            # Calculate normalized coordinates relative to this subdomain's bounding box
            # Note: The source defines x as normalized coordinates between [0, 1] 
            # for the given subdomain bounding box. 
            # We must map the global query coord to the local subdomain space.
            
            # Assuming 'min' and 'max' are global normalized coordinates.
            # Local x is (global_x - sub_min) / (sub_max - sub_min)
            
            dx = (x - sub['min'][0]) / (sub['max'][0] - sub['min'][0])
            dy = (y - sub['min'][1]) / (sub['max'][1] - sub['min'][1])
            dz = (z - sub['min'][2]) / (sub['max'][2] - sub['min'][2])
            
            # Compute gate value for each dimension
            # The gate function is defined over the subdomain including halo.
            # We need to determine if the point is within the subdomain's extended bounds.
            
            # Check if point is within the subdomain's extended bounds (including halo)
            # The halo is defined in index space, but gates operate on normalized coords.
            # We approximate halo in normalized space based on subdomain size.
            # Let's assume halo_size_normalized is provided or derived.
            # For simplicity, we use the clamped_tent_gate logic which requires normalized inputs.
            
            # Re-evaluating gate logic based on source: 
            # "x is normalized coordinates between [0, 1] for the given subdomain bounding box"
            # "gate function ... covers the subdomain Dk including the halo"
            
            # To implement correctly, we need the halo size in normalized units.
            # Let's assume subdomains have a 'halo_norm' field.
            halo_norm = sub.get('halo_norm', 0.0) # Not given in sources, derived from index halo
            
            # If we don't have halo_norm, we can't compute the exact gate.
            # However, the source says "x is normalized coordinates... for the given subdomain bounding box".
            # This implies the gate function is defined on the local [0,1] space of the subdomain,
            # but extended by the halo.
            
            # Let's assume the gate function G(x)_k is 1 if x is in the core, and tapers in the halo.
            # The core is the subdomain itself. The halo extends beyond.
            
            # Since we don't have the exact mapping of index halo to normalized halo,
            # we will use a placeholder for the gate calculation if not explicitly defined.
            # But the source gives the formula and the shape (clamped tent).
            
            # Let's assume the gate is 1 if the point is within the subdomain,
            # and decays linearly in the halo region.
            
            # Check if point is within the subdomain bounds (core)
            if 0 <= dx <= 1 and 0 <= dy <= 1 and 0 <= dz <= 1:
                gate_val = 1.0
            else:
                # Check if point is in the halo
                # This requires knowing the halo size in normalized coordinates.
                # Since it's not given, we assume the gate is 0 outside the subdomain for this simplified implementation.
                # [G] In a full implementation, the halo size in normalized coordinates must be calculated
                # from the index halo size and the subdomain resolution.
                gate_val = 0.0
            
            if gate_val > 0:
                expert_output = sub['expert']((dx, dy, dz))
                weighted_sum += gate_val * expert_output
                gate_sum += gate_val
        
        # Average the evaluated values weighted by the gate functions
        # Source: "average the evaluated values weighted by the gate functions"
        # This implies normalization by the sum of gates if multiple gates are active.
        if gate_sum > 0:
            result = weighted_sum / gate_sum
        else:
            result = 0.0 # Or undefined/NaN
            
        results.append(result)
        
    return results
```

**Parameters:**
*   **Subdomain Size:** Configurable size in index space, range 512 to 2048 [S1][S2, unofficial].
*   **Halo Size:** 8 voxels [S1][S2, unofficial].
*   **Gate Function:** Clamped tent function with max value 1 uniformly outside the overlapping region [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Discontinuity:** The halo size must be sufficient to eliminate discontinuities between subdomains. 8 voxels is chosen as a balance between eliminating discontinuity and reducing compute overhead [S1][S2, unofficial].
*   **Multiple Active Gates:** A single input coordinate can activate up to eight gate functions due to overlapping halos. The outputs must be averaged weighted by the gate functions [S1].
*   **Normalization:** Input coordinates are normalized to $[0, 1]$ before feature mapping. This normalization causes poor performance in large domains if not decomposed [S1][S2, unofficial].
*   **Learnable vs Fixed Gates:** Unlike standard MoE, the gate function is not learnable. It is a fixed geometric function [S1][S2, unofficial].

## Subtopics

### [Domain Decomposition Scene Representation](11124-domain-decomposition-scene-representation.md)
A model-parallel approach for large-scale volumes where a single network is replaced by a domain decomposition of models, each learning a separate brick of the volume. Models are arranged in a grid and require no communication during training or inference.

## Operating characteristics
*   **Time Complexity:** Inference time depends on the number of active subdomains (up to 8 due to halos) and the cost of the expert network. Optimization involves checking gate values first to avoid invoking experts with zero weight [S1].
*   **Memory Complexity:** Memory usage scales with the number of active subdomains and the size of the expert networks. Empty subdomains are discarded, reducing memory overhead [S1][S2, unofficial].
*   **Performance:** Handles large scenes and disjoint clusters better than a single network by avoiding the incoherence of normalized coordinates in large domains [S1][S2, unofficial].
*   **Training:** Supports concurrent training on multiple GPUs, allowing for scalable training of large volumes [S1].
*   **Limitations:** The fixed gate function may not adapt to complex data distributions as well as learnable gates in standard MoE [S1][S2, unofficial].

## Background
Sparse Domain Decomposition was proposed in the NeuralVDB paper (2022) by NVIDIA researchers [S1][S2, unofficial]. It is inspired by the sparsely-gated Mixture-of-Experts (MoE) method [Shazeer et al. 2017] [S1][S2, unofficial]. The method addresses the limitations of standard VDB value mapping, which is agnostic to incoherent clustering of voxels in large domains [S1][S2, unofficial].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
