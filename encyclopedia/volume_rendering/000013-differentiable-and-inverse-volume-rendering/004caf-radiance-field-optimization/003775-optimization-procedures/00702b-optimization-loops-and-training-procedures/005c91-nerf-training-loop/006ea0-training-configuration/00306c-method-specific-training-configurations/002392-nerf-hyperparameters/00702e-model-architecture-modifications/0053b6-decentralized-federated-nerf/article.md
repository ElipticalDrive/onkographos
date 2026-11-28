# Decentralized Federated NeRF

Also called: DecentNeRF, Decentralized Neural Radiance Fields from Crowdsourced Images

## Overview
DecentNeRF is a decentralized framework for learning global 3D scene representations from crowd-sourced images by distributing training computation to users and aggregating only global radiance field models to minimize server compute and personal data leakage [S1]. It solves the problem of achieving photorealistic 3D reconstruction of global scene content while preventing the server from reconstructing transient personal content (e.g., users, objects) present in crowdsourced images taken at different times [S1]. Unlike centralized approaches like NeRF-W which require all personal user images to be transferred to a central server, or existing federated learning NeRF techniques which assume static scenes devoid of personal information, DecentNeRF handles personal content and uses secure aggregation [S1]. It differs from standard federated learning by employing a Personal MLP, appearance embedding, secure aggregation, and learned weighted averaging instead of standard FedAvg aggregation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $K$ users, each possessing a local dataset of images $\mathcal{I}_k$ with known camera poses. The images contain both static global content (e.g., monuments) and dynamic personal content (e.g., people, transient objects) [S1].
*   **Outputs:** A globally aggregated global MLP $\mathbf{G}$ held by the server and distributed to users, and locally held personal MLPs $\mathbf{P}_k$ for each user [S1].

**Data Structures:**
Each user $k$ maintains two Multi-Layer Perceptrons (MLPs):
1.  **Global MLP:** Weights $\mathbf{g}_k$. Captures shared global content. These weights are federated to the server.
2.  **Personal MLP:** Weights $\mathbf{p}_k$. Captures user-specific transient content. These weights remain local to the user device and are never sent to the server [S1].

Both MLPs follow the NeRF backbone architecture [G]:
*   **Input:** A 5D vector consisting of 3D spatial coordinate $\mathbf{x} = (x, y, z)$ and viewing direction parameters pitch $\theta$ and yaw $\phi$ [S1].
*   **Output:** Scalar density $\sigma$ and RGB color vector $\mathbf{c} = (r, g, b)$ [S1].

**Parameters:**
*   $\alpha_k$: A per-user weighting factor applied to local weights/gradients during obfuscation and aggregation. It is learned via gradient descent to maximize visual fidelity, implicitly handling user importance based on factors like occlusion levels [S1].
*   Typical values: Not given in the sources.

**Algorithm Steps:**

1.  **Initialization:** Randomly initialize global MLP weights $\mathbf{G}^{(0)}$ and personal MLP weights $\mathbf{P}^{(0)}$ at all $K$ users [S1].
2.  **Local Training (User Side):** For each training round $m$:
    a.  Each user $k$ trains their local Global MLP ($\mathbf{g}_k$) and Personal MLP ($\mathbf{p}_k$) on their local images $\mathcal{I}_k$.
    b.  The rendering process synthesizes images via volume rendering by sampling predicted color and density at many points along rays traversing the scene [S1].
    c.  The final rendered image is an alpha-composite of the output images from both the global and personal MLPs [S1].
    d.  The user calculates the loss $L_k$ by comparing the predicted color with ground truth RGB values [S1].
    e.  The user computes the gradient of the loss with respect to the global MLP weights: $\frac{\partial L_k}{\partial \mathbf{g}^{(m-1)}}$ [S1].
    f.  The user updates the weighting factor $\alpha_k$ based on the gradient updates [S1].
3.  **Obfuscation (User Side):**
    a.  **Weight Obfuscation (`send_wts`):** Each user obfuscates their local global weights $\mathbf{g}_k$ by multiplying with $\alpha_k$ and applying a random mask or encrypting with additively homomorphic encryption [S1]. This ensures original values cannot be recovered during aggregation while enabling accurate weighted averaging [S1].
    b.  **Gradient Obfuscation (`send_grads`):** Each user communicates local gradients by performing randomized obfuscation similar to the weight obfuscation process [S1].
4.  **Secure Aggregation (Server Side):**
    a.  The server receives the obfuscated weights and gradients from all users.
    b.  The server uses a Secure Multi-Party Computation (SMPC) protocol to securely aggregate the weights and previous rounds' gradients [S1].
    c.  The function `weight_avg(g)` executes the computation of the weighted average of the obfuscated weights sent by the users [S1].
    d.  The server distributes the updated global MLP back to each user [S1].
5.  **Update (User Side):**
    a.  Each user receives the updated global MLP.
    b.  Each user calculates the update step for $\alpha_k$ and updates $\alpha_k$ accordingly [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global variables
K = number of users
M = number of federation rounds
alpha = [1.0] * K  # Initial per-user weighting factors

# User-side function
def user_train_and_obfuscate(user_id, local_images, global_mlp_weights, personal_mlp_weights, alpha_k):
    """
    Trains local MLPs and obfuscates global weights for transmission.
    """
    # 1. Local Training
    # Train global_mlp and personal_mlp on local_images
    # Loss is computed on alpha-composite of global and personal outputs
    # Compare rendered color with ground truth RGB
    global_mlp_weights, personal_mlp_weights = train_local_mlp(
        local_images, global_mlp_weights, personal_mlp_weights
    )
    
    # 2. Compute Gradients for alpha update
    # Calculate loss L_k and gradient w.r.t global weights
    loss_k, grad_global = compute_loss_and_gradient(
        global_mlp_weights, personal_mlp_weights, local_images
    )
    
    # 3. Update alpha_k
    # Update alpha_k based on gradient updates (Eq 2 in source)
    alpha_k = update_alpha(alpha_k, grad_global)
    
    # 4. Obfuscate Weights for Server
    # Multiply weights by alpha_k and apply random mask or homomorphic encryption
    obfuscated_weights = obfuscate(global_mlp_weights, alpha_k)
    
    # 5. Obfuscate Gradients for Server
    obfuscated_grads = obfuscate(grad_global, alpha_k)
    
    return obfuscated_weights, obfuscated_grads, alpha_k, global_mlp_weights, personal_mlp_weights

# Server-side function
def server_aggregate(obfuscated_weights_list, obfuscated_grads_list):
    """
    Securely aggregates obfuscated weights and gradients from all users.
    """
    # Use SMPC protocol to compute weighted average
    # The server does not see individual user weights
    aggregated_global_mlp = secure_weighted_average(obfuscated_weights_list)
    
    # Aggregate gradients similarly if needed for further updates
    aggregated_grads = secure_weighted_average(obfuscated_grads_list)
    
    return aggregated_global_mlp

# Main Loop
def decernet_training(K, M, local_datasets):
    # Initialization
    global_mlp_weights = initialize_mlp()
    personal_mlp_weights = [initialize_mlp() for _ in range(K)]
    
    for m in range(M):
        # Parallel user training
        results = []
        for k in range(K):
            obf_w, obf_g, new_alpha, g_w, p_w = user_train_and_obfuscate(
                k, local_datasets[k], global_mlp_weights, personal_mlp_weights[k], alpha[k]
            )
            results.append((obf_w, obf_g, new_alpha, g_w, p_w))
        
        # Server aggregation
        obf_w_list = [r[0] for r in results]
        obf_g_list = [r[1] for r in results]
        
        new_global_mlp = server_aggregate(obf_w_list, obf_g_list)
        
        # Update global state and user alphas
        global_mlp_weights = new_global_mlp
        for k in range(K):
            alpha[k] = results[k][2]
            personal_mlp_weights[k] = results[k][4]
            
    return global_mlp_weights, personal_mlp_weights
```

**Edge Cases and Pitfalls:**
*   **Initial Rounds:** During initial rounds of federation, users’ global MLPs contain personal content, leading to undesired reconstruction of personal content by the server [S1]. However, secure aggregation minimizes this leakage compared to non-secure methods [S1].
*   **View Overlap:** If two users have no overlapping views (entirely separate scene areas), the server's ability to reconstruct personal content increases considerably [S1]. The method assumes sufficient overlap in views such that no single user has views that don’t overlap with another user [S1].
*   **Standard Averaging:** Standard federated learning uniformly averages user weights, which does not maximize visual fidelity for global scene reconstruction [S1]. DecentNeRF uses learned weighted averaging to address this [S1].

## Operating characteristics
*   **Server Compute:** DecentNeRF requires $\sim 10^4$ times less server computing for a scene than a centralized approach [S1].
*   **FLOPs:** DecentNeRF FLOPs are $\sim 0.6$B (for specific setups mentioned in Fig 1b labels, though context implies comparative metrics) or $\approx 5$B server FLOPs on the Phototourism dataset, compared to $\approx 200$T FLOPs for NeRF-W [S1].
*   **Performance:** On the Phototourism dataset, DecentNeRF achieves PSNR 24.62, SSIM 0.8802, LPIPS 0.2571 [S1].
*   **Personal Content Leakage:** Personal content reconstruction by the server is reduced by 10 dB compared to FedNeRF [S1]. In the first federation round, server-rendered global image PSNR is 14.2 dB and server-rendered personal content PSNR is 14.9 dB. In the final federation round, server-rendered global image PSNR is 27.2 dB and server-rendered personal content PSNR is 9.3 dB [S1].
*   **Strengths:** Minimizes server compute, prevents personal data leakage, handles dynamic personal content, photorealistic global reconstruction.
*   **Limitations:** Requires known camera poses (assumed known, not derived decentralisedly in this work), assumes sufficient view overlap, honest-but-curious server model.

## Background
DecentNeRF was introduced by Z. Tasneem et al. in the paper "DecentNeRFs: Decentralized Neural Radiance Fields from Crowdsourced Images" [S1]. It is the first work to analyze the decentralization aspects of radiance fields for real-world crowd-sourced images [S1]. It builds on NeRF [G] and NeRF-W [S1], addressing their limitations regarding personal data access and server compute costs. It is used for decentralized NeRF training on crowdsourced datasets including the Novel Blender Dataset and Phototourism Dataset (Brandenburg Gate, Trevi Fountain, Sacre Couer) [S1].

## Sources
[S1] DecentNeRFs: Decentralized Neural Radiance Fields from Crowdsourced Images, https://link.springer.com/chapter/10.1007/978-3-031-73202-7_9, fetched 2026-10-30
