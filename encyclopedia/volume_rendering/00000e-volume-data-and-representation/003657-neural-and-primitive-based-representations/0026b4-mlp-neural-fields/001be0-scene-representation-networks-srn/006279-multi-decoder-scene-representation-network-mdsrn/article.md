# Multi-Decoder Scene Representation Network (MDSRN)
Also called: multi-decoder SRN
## Overview
MDSRN is a parameter-efficient ensemble architecture for feature grid Scene Representation Networks (SRNs) that shares a single feature grid encoder among multiple member networks while employing completely separate MLP decoders for each member [S1]. It solves the problem of quantifying prediction uncertainty (variance) in neural field reconstruction without the prohibitive parameter cost of training independent full networks [S1]. Unlike Deep Ensemble (DE) methods which scale parameters linearly with the number of members, MDSRN allows for higher-resolution shared grids and larger individual decoder capacities within a fixed parameter budget, improving mean prediction accuracy while providing variance estimates for quality-informed reconstruction [S1].

## How it works
**Inputs and Outputs**
- **Input:** A set of $B$ coordinate-value pairs $(x_b, y_b)$ for training, where $x_b \in \mathbb{R}^3$ is a spatial coordinate and $y_b \in \mathbb{R}$ is the ground truth scalar value (e.g., density or color) [S1].
- **Output:** For any query coordinate $x$, the method outputs a mean prediction $\mu(x)$ and an unbiased sample variance $\sigma^2(x)$, along with the individual member predictions $f_i(x)$ [S1].

**Data Structures**
The architecture consists of two main components:
1.  **Shared Encoder $E$:** A feature grid-based encoder (e.g., from APMGSRN, NGP, or fV-SRN) that maps coordinate $x$ to a feature vector $h = E(x)$ [S1]. This structure is shared identically by all $M$ members.
2.  **Member Decoders $\{D_i\}_{i=1}^M$:** A set of $M$ distinct Multi-Layer Perceptrons (MLPs). Each decoder $D_i$ takes the feature vector $h$ and outputs a scalar prediction $f_i(x)$ [S1]. The decoders are not shared; each has its own independent weights [S1].

**Numbered Steps**
1.  **Initialize:** Construct an ensemble of $M$ member predictors. Initialize the shared encoder $E$ and $M$ separate decoders $D_1, \dots, D_M$ [S1].
2.  **Encode:** For a batch of $B$ input coordinates $\{x_b\}_{b=1}^B$, compute the shared feature representation $h_b = E(x_b)$ [S1].
3.  **Decode:** For each member $i \in \{1, \dots, M\}$, compute the prediction $f_i(x_b) = D_i(h_b)$ [S1].
4.  **Compute Mean:** Calculate the mean prediction $\mu(x_b)$ for each coordinate in the batch:
    $$ \mu(x_b) = \frac{1}{M} \sum_{i=1}^{M} f_i(x_b) $$
5.  **Compute Variance:** Calculate the unbiased sample variance $\sigma^2(x_b)$ for each coordinate:
    $$ \sigma^2(x_b) = \frac{1}{M-1} \sum_{i=1}^{M} (f_i(x_b) - \mu(x_b))^2 $$
6.  **Compute Loss:** Calculate the total training loss $L_{MDSRN}$ using the sum of mean squared errors (MSE) between each member's prediction and the ground truth $y_b$:
    $$ L_{MDSRN} = \frac{1}{B} \sum_{b=1}^{B} \sum_{i=1}^{M} (f_i(x_b) - y_b)^2 $$
7.  **Optimize:** Update the weights of the shared encoder $E$ and all decoders $D_i$ using gradient descent to minimize $L_{MDSRN}$ [S1].

**Formulas**
-   **Member Prediction:** $f_i(x) = D_i(E(x))$ for $i = 1, \dots, M$ [S1].
-   **Mean Prediction:** $\mu(x) = \frac{1}{M} \sum_{i=1}^{M} f_i(x)$ [S1].
-   **Variance:** $\sigma^2(x) = \frac{1}{M-1} \sum_{i=1}^{M} (f_i(x) - \mu(x))^2$ [S1].
-   **Training Loss:** $L_{MDSRN} = \frac{1}{B} \sum_{b=1}^{B} \sum_{i=1}^{M} (f_i(x_b) - y_b)^2$ [S1].

**Parameters**
-   **$M$ (Number of Members):** The number of member networks in the ensemble. Typical values are 3 or 5, balancing accuracy against parameter count [S1].
-   **$B$ (Batch Size):** The number of coordinate-value pairs in a training batch [S1].

**Edge Cases and Pitfalls**
-   **Homogenous Variance:** Using identical or too-similar decoders leads to spatially homogenous variance, which fails to distinguish challenging regions from easy ones [S1]. This is prevented by using completely separate MLPs for different members [S1].
-   **Training Time:** Training time scales with the number of decoder members $M$, as each member requires forward and backward passes [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MDSRN:
    def __init__(self, encoder, decoder_class, num_members=3):
        """
        Initialize MDSRN.
        
        Args:
            encoder: A shared feature grid encoder module.
            decoder_class: The class for the MLP decoder (e.g., nn.Sequential).
            num_members (int): Number of member networks M.
        """
        self.encoder = encoder
        self.num_members = num_members
        # Initialize M separate decoders
        self.decoders = nn.ModuleList([decoder_class() for _ in range(num_members)])

    def forward(self, x):
        """
        Forward pass for a batch of coordinates x.
        
        Args:
            x: Tensor of shape (B, 3) containing batch coordinates.
            
        Returns:
            mean_pred: Tensor of shape (B,) containing mean predictions.
            variance: Tensor of shape (B,) containing variance estimates.
            member_preds: Tensor of shape (B, M) containing all member predictions.
        """
        # Step 1: Encode using shared encoder
        # E(x) returns feature vectors of shape (B, feature_dim)
        features = self.encoder(x)
        
        # Step 2: Decode using separate decoders
        member_preds_list = []
        for i in range(self.num_members):
            # D_i(E(x)) returns shape (B, 1)
            pred = self.decoders[i](features)
            member_preds_list.append(pred.squeeze(-1))
            
        # Stack predictions to shape (B, M)
        member_preds = torch.stack(member_preds_list, dim=1)
        
        # Step 3: Compute Mean
        # Mean over dim=1 (members)
        mean_pred = torch.mean(member_preds, dim=1)
        
        # Step 4: Compute Variance (Unbiased sample variance)
        # sigma^2 = 1/(M-1) * sum((f_i - mu)^2)
        # torch.var with unbiased=True uses 1/(N-1)
        variance = torch.var(member_preds, dim=1, unbiased=True)
        
        return mean_pred, variance, member_preds

    def compute_loss(self, x, y):
        """
        Compute the MDSRN training loss.
        
        Args:
            x: Tensor of shape (B, 3).
            y: Tensor of shape (B,) containing ground truth values.
            
        Returns:
            loss: Scalar tensor representing L_MDSRN.
        """
        # Get predictions from all members
        # member_preds shape: (B, M)
        _, _, member_preds = self.forward(x)
        
        # Expand y to match member_preds shape for broadcasting
        # y shape: (B,) -> (B, 1)
        y_expanded = y.unsqueeze(1)
        
        # Compute MSE for each member: (f_i(x) - y)^2
        # Shape: (B, M)
        mse_per_member = (member_preds - y_expanded) ** 2
        
        # Sum over members and batch, then divide by B
        # L_MDSRN = (1/B) * sum_b sum_i (f_i(x_b) - y_b)^2
        loss = torch.mean(mse_per_member)
        
        return loss
```

## Subtopics

### Adaptively Placed Multi-Grid Scene Representation Network (APMGSRN)
An SRN variant using adaptively placed multi-grids for feature storage. MDSRN can be applied to APMGSRN by sharing its multi-grid encoder among multiple decoders [S1].

### SRN+Linear
A variant of SRN. MDSRN is applicable to this architecture by sharing its linear encoder components among separate decoders [S1].

## Operating characteristics
-   **Parameter Efficiency:** MDSRN shares the feature grid encoder, avoiding the linear parameter scaling of Deep Ensemble (DE) methods. This allows for higher-resolution grids and larger individual decoder capacities within a fixed parameter budget [S1].
-   **Accuracy:** For the same total parameter count, an ensemble of fewer, larger member SRNs (enabled by sharing) often yields more accurate mean predictions than an ensemble of many smaller independent SRNs [S1].
-   **Training Time:** Training time scales linearly with the number of decoder members $M$, as each member requires independent forward and backward passes through its decoder [S1].
-   **Variance Quality:** Without additional regularization, the spatial pattern of variance may not always align with the actual error pattern [S1]. This limitation requires post-training confidence evaluation or additional regularization losses (e.g., RMDSRN) to mitigate [S1].
-   **Applicability:** Can be applied to state-of-the-art feature grid SRNs including APMGSRN, NGP, and fV-SRN [S1].

## Background
MDSRN was proposed to address the poor parameter scaling of Deep Ensemble (DE) methods in the context of Scene Representation Networks (SRNs) [S1]. The method was introduced in the paper "Regularized Multi-Decoder Ensemble for an Error-Aware Scene Representation Network" [S1]. It builds on the observation that the bottleneck for ensemble accuracy in SRNs is often the capacity of individual member networks, which is constrained by total parameter budgets [S1]. By sharing the feature grid encoder, MDSRN enables the use of larger, more accurate member decoders [S1].

## Sources
[S1] Regularized Multi-Decoder Ensemble for an Error-Aware Scene Representation Network, https://doi.org/10.1109/tvcg.2024.3456357, fetched 2026-10-26
