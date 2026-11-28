# Open-Set Semantic Fusion via Normal-Inverse-Gamma Conjugacy

Also called: NIG Update, Open-Set Bayesian Fusion, VLM Feature Fusion

## Overview
This method fuses continuous, open-set semantic feature vectors (e.g., from Vision-Language Models) into volumetric voxels by modeling each feature dimension as a Gaussian distribution with unknown mean and variance [S1]. It employs a Normal-Inverse-Gamma (NIG) conjugate prior to compute a closed-form posterior, enabling uncertainty-aware semantic mapping [S1]. Unlike closed-set fusion which uses Dirichlet-Categorical conjugacy for discrete classes, this approach models the full feature space, allowing for the representation of unseen or ambiguous semantic concepts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel $\mathbf{x}_*$ with existing NIG hyperparameters $(m_i, \lambda_i, \nu_i, \beta_i)$ for each feature dimension $i \in \{1, \dots, l\}$, and a set of new observations $\mathcal{Z}_i$ for dimension $i$.
*   **Output:** Updated posterior hyperparameters $(\tilde{m}_i, \tilde{\lambda}_i, \tilde{\nu}_i, \tilde{\beta}_i)$ and the predicted semantic class probabilities.

**Data Structures:**
*   **Voxel Storage:** Each voxel stores the mean vector $\mathbf{m}^* = (\tilde{m}_1, \dots, \tilde{m}_l)$ [S1].
*   **Covariance Tracking:** Due to the independence assumption between feature elements, only the diagonal of the covariance matrix is tracked via $\beta_i$ [S1].
*   **Optimization:** The hyperparameters $\lambda_i$ and $\nu_i$ are simplified by using the cumulative TSDF weight $W_t(\mathbf{x}_*)$ for $\lambda^*$ and $2\nu^*$, respectively [S1].

**Parameters:**
*   $l$: Dimensionality of the VLM embedding feature vector [S1].
*   $T_p$: Threshold for filtering uncertain predictions, typically $0.1$ [S1].

**Steps:**
1.  **Compute Posterior Hyperparameters:** For each feature dimension $i$, update the NIG parameters using the new observations $\mathcal{Z}_i$. Let $|\mathcal{Z}_i|$ be the number of observations and $\bar{Z}_i$ be their mean.
    *   $\tilde{\lambda}_i = \lambda_i + |\mathcal{Z}_i|$
    *   $\tilde{\nu}_i = \nu_i + \frac{|\mathcal{Z}_i|}{2}$
    *   $\tilde{m}_i = \frac{\lambda_i m_i + |\mathcal{Z}_i| \bar{Z}_i}{\tilde{\lambda}_i}$
    *   $\tilde{\beta}_i = \beta_i + \frac{1}{2} \sum_{z \in \mathcal{Z}_i} (z - \bar{Z}_i)^2 + \frac{\lambda_i |\mathcal{Z}_i|}{2 \tilde{\lambda}_i} (\bar{Z}_i - m_i)^2$
    *   *Note:* Source [S1] Equation (14) contains a typo in the denominator of the second term ($\lambda_i$ instead of $\tilde{\lambda}_i$). The implementation uses $\tilde{\lambda}_i$ [S1].

2.  **Predictive Distribution:** The predictive distribution for a new observation $Z_i$ is a Student-t distribution:
    $$p(Z_i \mid \mathcal{Z}_i, m_i, \lambda_i, \nu_i, \beta_i) = t_{2\tilde{\nu}_i} \left( Z_i \mid \tilde{m}_i, \frac{\tilde{\beta}_i(\tilde{\lambda}_i+1)}{\tilde{\lambda}_i\tilde{\nu}_i} \right)$$
    The expectation is $E(Z_i) = \tilde{m}_i$ when $\tilde{\lambda}_i > 1$ [S1].

3.  **Class Probability Estimation:** Apply a softmax function to the predicted feature vector (derived from the posterior predictive) to obtain absolute class probabilities [S1].

4.  **Uncertainty Filtering:** Filter out uncertain predictions by thresholding the highest class probability against $T_p = 0.1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_nig_posterior(m_i, lambda_i, nu_i, beta_i, observations):
    """
    Updates NIG hyperparameters for a single feature dimension.
    :param m_i: Prior mean
    :param lambda_i: Prior precision scaling
    :param nu_i: Prior degrees of freedom / 2
    :param beta_i: Prior scale parameter
    :param observations: List of new feature values Z_i
    :return: Tuple (tilde_m, tilde_lambda, tilde_nu, tilde_beta)
    """
    n = len(observations)
    if n == 0:
        return m_i, lambda_i, nu_i, beta_i

    # Calculate mean of new observations
    z_bar = sum(observations) / n

    # Update lambda and nu
    tilde_lambda = lambda_i + n
    tilde_nu = nu_i + n / 2.0

    # Update mean
    tilde_m = (lambda_i * m_i + n * z_bar) / tilde_lambda

    # Update beta
    # Sum of squared errors from the new mean
    sum_sq_err = sum((z - z_bar)**2 for z in observations)
    
    # Correction term for the shift in mean
    # Note: Source [S1] Eq 14 has a typo; denominator is tilde_lambda
    correction = (lambda_i * n / (2.0 * tilde_lambda)) * ((z_bar - m_i)**2)
    
    tilde_beta = beta_i + 0.5 * sum_sq_err + correction

    return tilde_m, tilde_lambda, tilde_nu, tilde_beta

def fuse_semantics(voxel_state, new_observations_per_dim):
    """
    Fuses new semantic observations into a voxel.
    :param voxel_state: Dict containing m, lambda, nu, beta for each dim
    :param new_observations_per_dim: Dict mapping dim index to list of observations
    :return: Updated voxel state and predicted class probabilities
    """
    updated_state = {}
    predicted_features = []

    for dim, obs in new_observations_per_dim.items():
        old_m = voxel_state['m'][dim]
        old_lambda = voxel_state['lambda'][dim]
        old_nu = voxel_state['nu'][dim]
        old_beta = voxel_state['beta'][dim]

        new_m, new_lambda, new_nu, new_beta = update_nig_posterior(
            old_m, old_lambda, old_nu, old_beta, obs
        )

        updated_state['m'][dim] = new_m
        updated_state['lambda'][dim] = new_lambda
        updated_state['nu'][dim] = new_nu
        updated_state['beta'][dim] = new_beta
        
        # Store mean for softmax input
        predicted_features.append(new_m)

    # Apply softmax to get class probabilities
    # Note: Implementation details of softmax normalization are standard
    probs = softmax(predicted_features)

    # Filter uncertain predictions
    max_prob = max(probs)
    if max_prob < 0.1: # T_p = 0.1
        return updated_state, None # Filtered out
    
    return updated_state, probs
```

**Edge Cases and Pitfalls:**
*   **Precision:** The expectation $E(Z_i)$ is valid only when $\tilde{\lambda}_i > 1$ [S1].
*   **Covariance:** The method assumes independence between feature elements, tracking only the diagonal of the covariance matrix $\beta$ [S1].
*   **Formula Typo:** Implementers must correct the denominator in the $\tilde{\beta}_i$ update from $\lambda_i$ to $\tilde{\lambda}_i$ as per the logical derivation of conjugate priors and the note in [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The update is closed-form and linear in the number of observations $|\mathcal{Z}_i|$ for the sum-of-squares calculation [S1].
*   Memory usage is reduced by storing only the mean vector $\mathbf{m}^*$ and diagonal $\beta$, and by using cumulative TSDF weights for $\lambda$ and $\nu$ [S1].
*   Source [S1] reports average FPS and memory usage in Table I, showing competitive performance compared to non-Bayesian heuristics (SLIM-VDB*), though specific numbers depend on the dataset (KITTI, SemanticKITTI, SceneNet) and hardware [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides a probabilistic framework for open-set semantics, handling uncertainty explicitly via the predictive Student-t distribution [S1].
*   **Limitations:** Relies on the independence assumption between feature dimensions [S1]. Requires thresholding ($T_p$) to handle uncertainty, which may discard valid but ambiguous data [S1].

## Background
**Origin:**
The method is part of the SLIM-VDB framework, introduced in "SLIM-VDB: A Real-Time 3D Probabilistic Semantic Mapping Framework" [S1]. It extends Bayesian semantic mapping from closed-set (Dirichlet-Categorical) to open-set (Normal-Inverse-Gamma) semantics [S1].

**History and Development:**
The approach leverages conjugate priors from probability theory, where the posterior belongs to the same family as the prior, allowing for recursive updates of hyperparameters [S1]. It builds on previous work in closed-set semantic fusion [18, 9] and extends it to continuous feature spaces [S1].

**Usage:**
Used in real-time 3D probabilistic semantic mapping, particularly for integrating Vision-Language Model (VLM) embeddings into voxel grids [S1].

## Subtopics

### Closed-Set Semantic Fusion
Uses Dirichlet-Categorical conjugacy for discrete classes. Choose when semantic labels are from a fixed, predefined set. See [Closed-Set Semantic Fusion](16202-closed-set-semantic-fusion.md).

### Uncertainty Guided Policy with Region Clustering
Uses uncertainty for view selection and clustering. Choose when active view planning is required. See [Uncertainty Guided Policy with Region Clustering](17856-uncertainty-guided-policy-with-region-clustering.md).

## Sources
[S1] SLIM-VDB: A Real-Time 3D Probabilistic Semantic Mapping Framework, https://doi.org/10.1109/lra.2026.3652875, fetched 2026-10-26
