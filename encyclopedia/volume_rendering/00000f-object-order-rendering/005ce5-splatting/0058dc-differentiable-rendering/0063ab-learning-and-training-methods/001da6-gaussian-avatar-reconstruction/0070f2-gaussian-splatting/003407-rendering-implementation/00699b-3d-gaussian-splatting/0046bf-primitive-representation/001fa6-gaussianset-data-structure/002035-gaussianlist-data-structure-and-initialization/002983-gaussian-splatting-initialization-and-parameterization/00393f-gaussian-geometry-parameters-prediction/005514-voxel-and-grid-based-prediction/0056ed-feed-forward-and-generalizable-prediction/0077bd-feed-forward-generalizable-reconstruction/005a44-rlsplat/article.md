# RLSplat

Also called: Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting

## Overview
RLSplat is a resource-linear, budget-controllable feed-forward 3D Gaussian Splatting (FF3DGS) framework that generates exactly $B$ Gaussians for a given input, ensuring that inference latency and peak memory scale linearly with the requested budget [S1, unofficial]. Unlike prior budget-controllable methods that predict a dense set of candidates and prune them (resulting in constant high cost), RLSplat uses a learned allocation policy to distribute the budget across local scene regions and a resource-linear decoder that refines Gaussians via local cross-attention [S1, unofficial]. It solves the problem of adapting to diverse resource constraints in real-world deployment (e.g., AR/VR, robotics) where both output size and inference speed must be controlled [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $V$ uncalibrated context images $\{I_i\}_{i=1}^V$, and a target Gaussian budget $B$ (integer).
*   **Outputs:** A set of $B$ 3D Gaussians $\mathcal{G} = \{g_b\}_{b=1}^B$ and camera parameters $\{P_i\}_{i=1}^V$ for the context views. Each Gaussian $g_b$ consists of a 3D center, opacity, scale, rotation, and color coefficients [S1, unofficial].

**Data Structures:**
1.  **Anchor Tokens:** A set of $N$ tokens $\{a_n\}_{n=1}^N$ representing local 3D regions, extracted from the geometry backbone.
2.  **Allocation Vector:** A discrete vector $k \in \mathbb{Z}_{\ge 0}^N$ where $k_n$ is the number of Gaussians assigned to anchor $n$, such that $\sum_{n=1}^N k_n = B$.
3.  **Gaussian Tokens:** A sequence of $B$ tokens, where anchor $n$ spawns $k_n$ tokens.
4.  **Hierarchical Neighbor Search Structure:** Used to find the $K_{gs}$ nearest Gaussian neighbors for each token efficiently.

**Parameters:**
*   $B$: Target Gaussian budget. Typical values: 2K (2,048), 8K, 64K [S1, unofficial].
*   $K_{gs}$: Number of nearest Gaussian neighbors for local refinement. Default ablation value: 16. Can be changed at inference without retraining [S1, unofficial].
*   $K_{anc}$: Number of anchor neighbors in hierarchical search. Fixed value in ablations: 16 [S1, unofficial].
*   $C$: Number of rollouts (candidate allocations) drawn per training step for GSPO optimization [S1, unofficial].

**Algorithm Steps:**

1.  **Geometry Encoding:** The geometry backbone encodes the $V$ input images into $N$ anchor tokens $\{a_n\}_{n=1}^N$ and predicts camera parameters $\{P_i\}_{i=1}^V$ [S1, unofficial].
2.  **Gaussian Allocation:** The allocation policy $\pi_\psi$ assesses the local significance of each anchor based on image features (e.g., texture complexity) and predicts a discrete allocation $k \in \mathbb{Z}_{\ge 0}^N$ such that $\sum_{n=1}^N k_n = B$. This policy is trained using Group Sequence Policy Optimization (GSPO) [S1, unofficial].
3.  **Token Spawning:** Each anchor $n$ spawns $k_n$ Gaussian tokens. The total number of Gaussian tokens is exactly $B$ [S1, unofficial].
4.  **Resource-Linear Decoding:**
    *   **Initialization:** The decoder instantiates initial Gaussian states (center, scale, rotation, opacity, color) from the $B$ Gaussian tokens.
    *   **Hierarchical Neighbor Search:** For each Gaussian token, the system finds its $K_{gs}$ nearest neighbors among the other $B-1$ Gaussian tokens using a hierarchical neighbor search. This search operates over the anchor tokens first (using $K_{anc}$ neighbors) to narrow down the candidate Gaussian tokens, ensuring efficiency [S1, unofficial].
    *   **Local Refinement:** Each Gaussian token refines its state via local cross-attention with its $K_{gs}$ nearest Gaussian neighbors. This replaces global self-attention, reducing complexity from $O(B^2)$ to $O(B \cdot K_{gs})$ [S1, unofficial].
5.  **Rendering:** The final set of $B$ Gaussians is rendered into target views using differentiable rasterization [S1, unofficial].

**Training (GSPO):**
To optimize the non-differentiable allocation policy $\pi_\psi$:
1.  Draw $C$ candidate allocations $k^{(c)} \sim \pi_{\text{old}} = \text{Mult}(B, \text{sg}(p))$, where $p$ are the policy probabilities and $\text{sg}(\cdot)$ stops gradients [S1, unofficial].
2.  For each rollout $c$, generate the 3DGS representation $\mathcal{G}^{(c)}$ and render an image $\hat{I}^{(c)}$ at novel target views.
3.  Compute reconstruction loss $\mathcal{L}_{\text{recon}}$ against ground-truth images $I$.
4.  Use $\mathcal{L}_{\text{recon}}$ as the reward signal to update the policy [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
from typing import List, Tuple

# Standard math/vector operations assumed available
# e.g., torch.norm, torch.softmax, torch.matmul

class RLSplat:
    def __init__(self, geometry_backbone, allocation_policy, decoder, num_anchors: int):
        self.geometry_backbone = geometry_backbone
        self.allocation_policy = allocation_policy
        self.decoder = decoder
        self.num_anchors = num_anchors
        # Hierarchical neighbor search structure
        self.hierarchical_search = HierarchicalNeighborSearch()

    def forward(self, images: torch.Tensor, budget: int) -> Tuple[torch.Tensor, torch.Tensor]:
        """
        Args:
            images: Tensor of shape (V, H, W, 3) or (V, 3, H, W)
            budget: Integer B, total number of Gaussians to generate
        Returns:
            gaussians: Tensor of shape (B, 14) or similar, containing Gaussian parameters
            cameras: Tensor of camera parameters for V views
        """
        # Step 1: Geometry Encoding
        # Encodes images into N anchor tokens and predicts camera parameters
        anchor_tokens, cameras = self.geometry_backbone(images)
        # anchor_tokens shape: (N, D_anchor)
        
        # Step 2: Gaussian Allocation
        # Predicts discrete allocation k summing to budget
        # Policy outputs probabilities p, sampled to get k
        # k is a tensor of shape (N,) with integer values >= 0
        k = self.allocation_policy(anchor_tokens, budget)
        
        # Step 3: Token Spawning
        # Create B Gaussian tokens. Each anchor n spawns k[n] tokens.
        # We need to expand anchor tokens to match the budget
        gaussian_tokens = []
        for n in range(self.num_anchors):
            count = k[n].item()
            # Expand anchor token 'count' times
            # Add small noise or variation if needed, but source implies direct spawning
            expanded_tokens = anchor_tokens[n].unsqueeze(0).expand(count, -1)
            gaussian_tokens.append(expanded_tokens)
        
        gaussian_tokens = torch.cat(gaussian_tokens, dim=0)
        # gaussian_tokens shape: (B, D_gaussian)
        
        # Step 4: Resource-Linear Decoding
        # Initialize Gaussians from tokens
        initial_gaussians = self.decoder.initialize(gaussian_tokens)
        
        # Hierarchical Neighbor Search
        # Find K_gs nearest neighbors for each of the B Gaussians
        # This uses the hierarchical structure to avoid O(B^2)
        neighbor_indices = self.hierarchical_search.find_neighbors(
            initial_gaussians, 
            k_gs=self.decoder.k_gs, 
            k_anc=self.decoder.k_anc
        )
        # neighbor_indices shape: (B, K_gs)
        
        # Local Refinement via Cross-Attention
        final_gaussians = self.decoder.refine(
            initial_gaussians, 
            neighbor_indices
        )
        
        return final_gaussians, cameras

class HierarchicalNeighborSearch:
    def __init__(self):
        # Implementation details of hierarchical search (e.g., KD-Tree or FAISS-like structure)
        # Source mentions it is 27x faster than exact search at 65,536 Gaussians
        pass

    def find_neighbors(self, gaussians: torch.Tensor, k_gs: int, k_anc: int) -> torch.Tensor:
        """
        Finds k_gs nearest Gaussian neighbors for each Gaussian.
        Uses hierarchical search over anchors first.
        """
        # Pseudocode for hierarchical search:
        # 1. Map each Gaussian to its parent anchor.
        # 2. For each Gaussian, find k_anc nearest parent anchors.
        # 3. Collect all Gaussians belonging to those anchors.
        # 4. Compute distances to these candidate Gaussians.
        # 5. Select the k_gs nearest ones.
        
        # Note: Exact implementation depends on the specific hierarchical structure used.
        # The source emphasizes efficiency: O(B * K_gs) vs O(B^2).
        pass

class AllocationPolicy(nn.Module):
    def __init__(self, num_anchors: int):
        super().__init__()
        self.num_anchors = num_anchors
        # Policy network to output probabilities for allocation
        self.network = nn.Sequential(
            nn.Linear(128, 256), # Example hidden size
            nn.ReLU(),
            nn.Linear(256, num_anchors)
        )
    
    def forward(self, anchor_tokens: torch.Tensor, budget: int) -> torch.Tensor:
        """
        Predicts allocation k summing to budget.
        Uses GSPO during training.
        """
        # Compute probabilities
        logits = self.network(anchor_tokens)
        probs = torch.softmax(logits, dim=-1)
        
        # Sample discrete allocation
        # In training, this is sampled from Multinomial distribution
        # In inference, it might be deterministic or sampled
        # Source says "predicts a discrete allocation k"
        # We assume a sampling step here for the policy
        k = torch.multinomial(probs, num_samples=budget, replacement=True)
        
        # Count occurrences to get k_n for each anchor
        # k is a list of anchor indices, length B
        # We need to count how many times each anchor index appears
        k_counts = torch.zeros(self.num_anchors, dtype=torch.long)
        k_counts.scatter_add_(0, k, torch.ones_like(k))
        
        return k_counts
```

**Edge Cases and Pitfalls:**
*   **Precision:** The allocation policy produces discrete integers. Gradient estimation via GSPO is required for training [S1, unofficial].
*   **Boundaries:** The hierarchical neighbor search must correctly handle boundary cases where fewer than $K_{gs}$ neighbors are available (though with $B$ Gaussians and $K_{gs} \ll B$, this is rare) [S1, unofficial].
*   **Degenerate Input:** If input images are uncalibrated, the geometry backbone must robustly predict camera parameters. The projection loss $\mathcal{L}_{proj}$ is critical to tie anchors to image regions; removing it causes training failure [S1, unofficial].

## Subtopics

### Gaussian Allocation Policy
A learned policy that distributes the Gaussian budget $B$ across anchors based on local scene significance (texture complexity). It differs from uniform allocation by concentrating capacity on high-frequency structures (e.g., thin balusters) and smoothing low-frequency areas (e.g., walls). It is chosen when resource efficiency and high-fidelity reconstruction of complex details are required [S1, unofficial].

### Resource-Linear Decoder
A refinement module that uses local cross-attention with hierarchical neighbor search instead of global self-attention. It differs from global attention by reducing complexity from $O(B^2)$ to $O(B \cdot K_{gs})$. It is chosen when inference latency and memory must scale linearly with the output size [S1, unofficial].

### Hierarchical Neighbor Search
An efficient search structure to find $K_{gs}$ nearest neighbors among $B$ Gaussians. It differs from exact search by being 27x faster at large budgets (e.g., 65,536 Gaussians). It is chosen to enable real-time or near-real-time decoding for large Gaussian sets [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Decoding cost is $O(B \cdot K_{gs})$, scaling linearly with the Gaussian budget $B$. This is a significant improvement over global self-attention, which costs $O(B^2)$ [S1, unofficial].
*   **Memory Complexity:** Peak memory usage scales linearly with $B$. At 2K Gaussians, RLSplat uses 9.6× less peak memory than F4 Splat [S1, unofficial].
*   **Measured Performance:**
    *   **ACID Dataset (B=8,192):** PSNR 22.84 dB, SSIM 0.643, LPIPS 0.353 [S1, unofficial].
    *   **RE10K Dataset (B=8,000):** Full-view PSNR 26.51 dB, compared to 23.42 dB for F4 Splat and 21.82 dB for EcoSplat [S1, unofficial].
    *   **Speed:** At 2K Gaussians, RLSplat runs 4.1× faster than F4 Splat [S1, unofficial].
    *   **Quality vs. Budget:** RLSplat achieves higher PSNR than F4 Splat at 2K Gaussians while using significantly less memory and time [S1, unofficial].
*   **Strengths:** Linear scaling of inference cost with budget; high-quality reconstruction of thin, high-frequency structures; no need for per-scene optimization.
*   **Limitations:** Requires training of the allocation policy and decoder; performance depends on the accuracy of the geometry backbone and camera prediction [S1, unofficial].
*   **When to Use:** Applications requiring limited resource budgets, such as AR/VR, robotics, and digital content creation, where both output size and inference speed must be controlled [S1, unofficial].
*   **When Not to Use:** When maximum possible quality is required regardless of resource constraints (e.g., offline rendering with unlimited memory/time), traditional 3DGS with per-scene optimization may be better [S1, unofficial].

## Background
RLSplat was proposed by Bae, Park, and Kim from KAIST. The paper "RLSplat: Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting" was published on arXiv in October 2026 [S1, unofficial]. It addresses the limitations of prior budget-controllable FF3DGS methods like EcoSplat and F4 Splat, which predict dense sets of Gaussians and then prune them, resulting in constant inference latency and memory usage regardless of the target budget [S1, unofficial]. RLSplat introduces a resource-linear approach by learning a Gaussian allocation policy and using a resource-linear decoder with local cross-attention [S1, unofficial].

## Sources
[S1] RLSplat: Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting, https://arxiv.org/abs/2610.27712, fetched 2026-10-26 (unofficial)
