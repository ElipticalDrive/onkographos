# Meta-learning Pretraining for Portrait NeRF

Also called: Meta-learning framework, Pretraining NeRF

## Overview
A meta-learning approach to pretrain a NeRF model parameter $\theta_p^*$ that can easily adapt to capturing an unseen subject’s appearance and geometry by learning common representations (head geometry, occlusion) from a dataset of subjects [S1, unofficial]. Unlike methods that transfer gradients from the support set, this method only propagates gradients learned from the query set to the pretrained model, as the support set is available at test time and its gradients are adapted via fine-tuning [S1, unofficial]. It addresses the limitation where standard meta-learning algorithms for image classification perform poorly for view synthesis because each update requires gradients from millions of samples across scene coordinates and viewing directions, which do not fit into a single GPU batch [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dataset of $K$ subjects. For each subject $m$, a support set $D_s$ (frontal view) and a query set $D_q$ (remaining views).
- **Output:** A final pretrained model parameter $\theta_p^* = \theta_{p, K-1}$ [S1, unofficial].

**Data Structures:**
- $\theta_{p, m}$: Global pretrained parameter for subject $m$.
- $\theta_{m, t}^*$: Model parameter for subject $m$ at inner iteration $t$.
- $D_s$: Support set (frontal view).
- $D_q$: Query set (other views).

**Parameters:**
- $K$: Total number of subjects in the training dataset, indexed by $m = \{0, \dots, K-1\}$ [S1, unofficial].
- $\alpha$: Learning rate for the support set update step [S1, unofficial].
- $\beta$: Learning rate for the query set update step [S1, unofficial].
- $N_s$: Number of iterations for updating the model on the support set [S1, unofficial].
- $N_q$: Number of iterations for updating the pretrained parameter using the query set [S1, unofficial].

**Steps:**
1. Initialize the first subject's pretrained parameter $\theta_{p, -1}$ to random weights [S1, unofficial].
2. For each subject $m$ from $0$ to $K-1$:
    a. Initialize the model parameter $\theta_{m, 0}^*$ from the previous subject's pretrained parameter $\theta_{p, m-1}$ [S1, unofficial].
    b. **Pretrain on Support Set $D_s$:** Iteratively update $\theta_{m, t}$ using gradient descent on the L2 loss between the predicted front view and the support set $D_s$ for $N_s$ iterations, using learning rate $\alpha$ [S1, unofficial].
    c. **Pretrain on Query Set $D_q$:** Iteratively update the current subject's model parameter $\theta_{m, t}$ using gradient descent on the loss between prediction and query set $D_q$ for $N_q$ iterations, using learning rate $\beta$ [S1, unofficial].
    d. **Update Global Pretrained Parameter:** Simultaneously update the global pretrained parameter $\theta_{p, m}$ using the same gradients computed from the query set $D_q$ for $N_q$ iterations, using learning rate $\beta$ [S1, unofficial].
    e. Update the global pretrained parameter for the next subject: Set $\theta_{p, m+1}$ to the final value of $\theta_{p, m}$ after $N_q$ iterations [S1, unofficial].
3. Final Pretrained Model: After visiting all $K$ subjects, the final pretrained model is $\theta_p^* = \theta_{p, K-1}$ [S1, unofficial].

**Formulas:**
- Support Set Update (for $t = 0, \dots, N_s-1$):
  $$ \theta_{m, t+1}^* = \theta_{m, t}^* - \alpha \nabla_{\theta} L_{D_s}(f_{\theta_{m, t}^*}) $$
  where $\theta_{m, 0}^* = \theta_{p, m-1}$ and $\theta_{m, N_s-1}^*$ is the result of the support set pretraining [S1, unofficial].

- Query Set Update (for $t = 0, \dots, N_q-1$):
  $$ \theta_{m, t+1}^* = \theta_{m, t}^* - \beta \nabla_{\theta} L_{D_q}(f_{\theta_{m, t}^*}) $$
  $$ \theta_{p, m, t+1} = \theta_{p, m, t} - \beta \nabla_{\theta} L_{D_q}(f_{\theta_{m, t}^*}) $$
  where $\theta_{m, 0}^*$ is the parameter learned from $D_s$ in the previous step, and $\theta_{p, m, 0} = \theta_{p, m-1}$ [S1, unofficial].

- Final Pretrained Parameter Update:
  $$ \theta_{p, m+1} = \theta_{p, m, N_q-1} $$
  Note that the update in (3) does not affect the update of the current subject $m$ (i.e., (2)), but the gradients are propagated to the next training subject in subsequent iterations through the pretrained model parameter update in (4) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def meta_pretrain(dataset, K, alpha, beta, Ns, Nq):
    # Initialize first subject's pretrained parameter to random weights
    theta_p_prev = random_weights()
    
    for m in range(K):
        # Initialize current subject's model parameter from previous pretrained parameter
        theta_m = theta_p_prev
        
        # Pretrain on Support Set Ds
        for t in range(Ns):
            # Compute L2 loss between predicted front view and support set Ds
            loss_ds = compute_l2_loss(f_theta(theta_m), dataset[m].Ds)
            # Gradient descent update
            grad_ds = gradient(loss_ds, theta_m)
            theta_m = theta_m - alpha * grad_ds
            
        # Pretrain on Query Set Dq
        # Initialize global pretrained parameter for this subject
        theta_p_curr = theta_p_prev
        
        for t in range(Nq):
            # Compute loss between prediction and query set Dq
            loss_dq = compute_loss(f_theta(theta_m), dataset[m].Dq)
            # Compute gradients
            grad_dq = gradient(loss_dq, theta_m)
            
            # Update current subject's model parameter
            theta_m = theta_m - beta * grad_dq
            
            # Simultaneously update global pretrained parameter
            theta_p_curr = theta_p_curr - beta * grad_dq
            
        # Update global pretrained parameter for the next subject
        theta_p_next = theta_p_curr
        
        # Move to next subject
        theta_p_prev = theta_p_next
        
    # Final pretrained model
    theta_p_star = theta_p_prev
    return theta_p_star
```

**Edge Cases and Pitfalls:**
- Standard meta-learning algorithms for image classification perform poorly for view synthesis because each update requires gradients from millions of samples across scene coordinates and viewing directions, which do not fit into a single GPU batch. This method takes many more steps in a single meta-training task for convergence [S1, unofficial].
- The method assumes that the order of applying the gradients learned from $D_q$ and $D_s$ are interchangeable, similarly to the first-order approximation in the MAML algorithm [S1, unofficial].

## Subtopics

### Finetuning and Rendering
At test time, only a single frontal view of the subject $s$ is available [S1, unofficial]. First, compute the rigid transform $(s_m, R_m, t_m)$ for the input subject $s$ to map between world and canonical coordinates [S1, unofficial]. Finetune the pretrained model parameter $\theta_p^*$ by repeating the support set update iteration (Equation 1) for the input subject to obtain the optimized model parameter $\theta_s^*$ [S1, unofficial]. To render novel views, sample a camera ray in 3D space, warp the ray's 3D coordinates to the canonical space using the subject's rigid transform, and feed the warped coordinate to NeRF to predict color and occupancy [S1, unofficial]. The meshes are for visualization and not used in view synthesis; only the rigid transform is used [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method requires iterating through $K$ subjects, each involving $N_s$ iterations on the support set and $N_q$ iterations on the query set. Since each update requires gradients from millions of samples, the computational cost per iteration is high, and the method takes many more steps in a single meta-training task for convergence compared to image classification tasks [S1, unofficial].
- **Memory Complexity:** Not given in the sources. However, the need to handle millions of samples across scene coordinates and viewing directions suggests significant memory requirements for gradient computation, which do not fit into a single GPU batch [S1, unofficial].
- **Strengths:** Learns common representations (head geometry, occlusion) from a dataset of subjects, allowing for easy adaptation to unseen subjects' appearance and geometry [S1, unofficial]. Only propagates gradients learned from the query set to the pretrained model, as the support set is available at test time and its gradients are adapted via fine-tuning [S1, unofficial].
- **Limitations:** Standard meta-learning algorithms for image classification perform poorly for view synthesis due to the large number of samples required for gradient computation [S1, unofficial]. Requires a dataset of multiple subjects for pretraining [S1, unofficial].
- **When to use:** When there is a dataset of multiple subjects available for pretraining, and the goal is to quickly adapt to a new subject's appearance and geometry using only a single frontal view at test time [S1, unofficial].
- **When not to use:** When there is no dataset of multiple subjects available for pretraining, or when computational resources are limited due to the high cost of gradient computation per iteration [S1, unofficial].

## Background
Originated in the paper "Portrait Neural Radiance Fields from a Single Image" [S1, unofficial]. The method is designed to address the challenge of capturing an unseen subject's appearance and geometry from a single frontal view by leveraging common representations learned from a dataset of subjects [S1, unofficial]. It builds on the idea of meta-learning, where the model is trained to learn how to learn, allowing for rapid adaptation to new tasks (in this case, new subjects) [S1, unofficial]. The method assumes that the order of applying the gradients learned from the query set and support set are interchangeable, similarly to the first-order approximation in the MAML algorithm [S1, unofficial].

## Sources
[S1] Portrait Neural Radiance Fields from a Single Image, https://doi.org/10.48550/arxiv.2012.05903, fetched 2026-10-26 (unofficial)
