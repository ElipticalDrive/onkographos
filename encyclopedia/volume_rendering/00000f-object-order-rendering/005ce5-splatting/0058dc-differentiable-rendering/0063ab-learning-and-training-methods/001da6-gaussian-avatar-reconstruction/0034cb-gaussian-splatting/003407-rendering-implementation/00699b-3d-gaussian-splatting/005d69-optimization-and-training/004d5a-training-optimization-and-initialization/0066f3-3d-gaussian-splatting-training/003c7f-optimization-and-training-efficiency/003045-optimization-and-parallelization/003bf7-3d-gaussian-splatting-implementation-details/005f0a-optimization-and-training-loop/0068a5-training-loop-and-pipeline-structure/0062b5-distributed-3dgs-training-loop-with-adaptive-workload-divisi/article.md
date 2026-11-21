# Distributed 3DGS Training Loop with Adaptive Workload Division

Also called: Batched Distributed 3DGS Training, Grendel-GS Training Pipeline

## Overview
This method implements a distributed training workflow for 3D Gaussian Splatting that batches camera selection, divides rendering workloads across multiple GPUs using an adaptive strategy history, synchronizes losses, and schedules densification and spherical harmonics (SH) degree updates [S1, unofficial]. It differs from standard gradient accumulation by optionally scaling gradients by batch size to avoid explicit accumulation buffers and by dynamically balancing the computational load of rendering different camera views across the GPU cluster [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training dataset, initial 3D Gaussian scene parameters, distributed environment configuration (number of GPUs, backend), and hyperparameters (`args.bsz`, `args.lambda_dssim`, `args.lr_scale_mode`, `args.local_sampling`, `args.save_iterations`, `args.checkpoint_iterations`, `testing_iterations`).
*   **Outputs:** Optimized 3D Gaussian parameters, strategy history log, and periodic checkpoints.

**Data Structures:**
*   `batched_screenspace_pkg`: Dictionary containing at least `"batched_cuda_args"`, a list of CUDA argument structures. Each structure contains a `"stats_collector"` field [S1, unofficial].
*   `batched_losses`: Tensor of shape `(N, 2)` where $N$ is the batch size. Index 0 holds the first loss component (L1), index 1 holds the second loss component (used in DSSIM calculation) [S1, unofficial].
*   `strategy_history`: List of previous workload division strategies and their associated statistics, used to adapt future divisions [S1, unofficial].
*   `gpuid2tasks`: Mapping of GPU IDs to the specific camera tasks assigned to each GPU for the current batch [S1, unofficial].

**Parameters:**
*   `args.bsz`: Batch size per iteration step. Controls the loop range and serves as the divisor for gradient scaling [S1, unofficial].
*   `args.lambda_dssim`: Weight for the DSSIM loss component. Default value not specified in the sources [S1, unofficial].
*   `args.lr_scale_mode`: String controlling gradient scaling. If not equal to `"accumu"`, gradients are explicitly divided by `args.bsz` before the optimizer step [S1, unofficial].
*   `args.local_sampling`: Boolean flag. If true, enables distributed camera index gathering across GPUs using `torch.distributed.all_gather_into_tensor` [S1, unofficial].

**Numbered Steps:**

1.  **Initialization:** Initialize the training loop from `start_from_this_iteration` to `opt_args.iterations` with a step size of `args.bsz`. Set up progress bar updates every `args.bsz` iterations [S1, unofficial].
2.  **Logging Setup:** Log the Exponential Moving Average (EMA) loss to the progress bar every 30 batches (checked via `iteration // args.bsz % 30 == 0`) [S1, unofficial].
3.  **State Update:** Update the learning rate and increment the trained batch counter. Clear timing variables. Start NVTX profiling range if enabled [S1, unofficial].
4.  **SH Degree Update:** Upscale the spherical harmonics degree by one level every 1000 iterations. This is determined by `utils.check_update_at_this_iter(iteration, args.bsz, 1000, 0)` [S1, unofficial].
5.  **Camera Selection:**
    *   If `args.local_sampling` is true: Assert that `args.bsz` is divisible by `utils.WORLD_SIZE`. Gather local camera indices across all GPUs using `torch.distributed.all_gather_into_tensor`, convert to numpy, and fetch the corresponding camera objects [S1, unofficial].
    *   Otherwise: Fetch camera objects directly via `train_dataset.get_batched_cameras(args.bsz)` [S1, unofficial].
6.  **Workload Division Strategy:** Initialize the workload division strategy by calling `start_strategy_final(batched_cameras, strategy_history)`. This returns `batched_strategies` and `gpuid2tasks` [S1, unofficial].
7.  **Data Loading:** Load camera data from CPU to all GPUs using `load_camera_from_cpu_to_all_gpu(batched_cameras, batched_strategies, gpuid2tasks)` [S1, unofficial].
8.  **Preprocessing:** Preprocess the 3D Gaussian Splatting data and perform all-to-all communication.
    *   If `args.backend == "gsplat"`, call `gsplat_distributed_preprocess3dgs_and_all2all_final`.
    *   Otherwise, call `distributed_preprocess3dgs_and_all2all_final` [S1, unofficial].
9.  **Rendering:** Render images and extract compute locations by calling `gsplat_render_final` or `render_final` with `batched_screenspace_pkg` and `batched_strategies`. Extract `batch_statistic_collector` from `batched_screenspace_pkg["batched_cuda_args"]` [S1, unofficial].
10. **Loss Computation:** Compute the loss by calling `batched_loss_computation` with the rendered images, cameras, compute locations, strategies, and statistic collectors [S1, unofficial].
11. **Backpropagation:** Execute the backward pass using `loss_sum.backward()` [S1, unofficial].
12. **Strategy Update:** Update the workload division strategy by calling `finish_strategy_final(batched_cameras, strategy_history, batched_strategies, batch_statistic_collector)` [S1, unofficial].
13. **Loss Synchronization:** Synchronize losses across all GPUs using `dist.all_reduce(batched_losses, op=dist.ReduceOp.SUM, group=utils.DEFAULT_GROUP)` [S1, unofficial].
14. **EMA Logging:** Update the EMA loss for logging. If `ema_loss_for_log` is `None`, set it to the mean of `batched_loss_cpu`. Otherwise, update using:
    $$ EMA = 0.6 \cdot EMA + 0.4 \cdot \text{mean}(batched\_loss\_cpu) $$
    [S1, unofficial].
15. **Evaluation:** Trigger evaluation if the current iteration matches `testing_iterations`. Log validation configurations for both test and train sets [S1, unofficial].
16. **Densification:** Perform densification by calling `gsplat_densification` or `densification` with the current iteration, scene, gaussians, and screenspace package [S1, unofficial].
17. **Saving:**
    *   Save Gaussians if the current iteration falls within `args.save_iterations` [S1, unofficial].
    *   Write the strategy history to a JSON file if `args.save_strategy_history` is true [S1, unofficial].
    *   Save checkpoints if the current iteration falls within `args.checkpoint_iterations`. Create the directory (Rank 0), synchronize ranks via `torch.distributed.barrier`, and save `gaussians.capture()` and the iteration number to a `.pth` file [S1, unofficial].

**Formulas:**
The combined batch loss is calculated as:
$$ L_{batch} = (1 - \lambda_{dssim}) \cdot L_{0} + \lambda_{dssim} \cdot (1 - L_{1}) $$
where $L_{0}$ is the first loss component (typically L1), $L_{1}$ is the second loss component (typically SSIM-related), and $\lambda_{dssim}$ is `args.lambda_dssim` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.distributed as dist
import numpy as np

def distributed_3dgs_training_loop(
    train_dataset, 
    gaussians, 
    opt_args, 
    args, 
    utils,
    start_from_this_iteration=0
):
    """
    Distributed training loop for 3D Gaussian Splatting with adaptive workload division.
    """
    iteration = start_from_this_iteration
    ema_loss_for_log = None
    strategy_history = []
    
    # Assume these are defined in the surrounding context or passed in
    # backend = args.backend
    # WORLD_SIZE = utils.WORLD_SIZE
    # DEFAULT_GROUP = utils.DEFAULT_GROUP
    
    while iteration < opt_args.iterations:
        # Step 2: Log EMA loss every 30 batches
        if iteration // args.bsz % 30 == 0:
            if ema_loss_for_log is not None:
                print(f"[ITER {iteration}] EMA Loss: {ema_loss_for_log}")
        
        # Step 3: Update LR, clear timers, start profiling
        utils.update_learning_rate(iteration)
        utils.clear_timers()
        if args.profile:
            torch.cuda.nvtx.range_push(f"iter_{iteration}")
            
        # Step 4: SH Degree Update
        if utils.check_update_at_this_iter(iteration, args.bsz, 1000, 0):
            utils.upscale_sh_degree(gaussians)
            
        # Step 5: Camera Selection
        if args.local_sampling:
            assert args.bsz % WORLD_SIZE == 0
            # Gather local indices
            local_indices = train_dataset.get_local_indices(args.bsz // WORLD_SIZE)
            gathered_indices = [torch.zeros_like(local_indices) for _ in range(WORLD_SIZE)]
            dist.all_gather(gathered_indices, local_indices)
            all_indices = torch.cat(gathered_indices).numpy()
            batched_cameras = [train_dataset.get_camera(idx) for idx in all_indices]
        else:
            batched_cameras = train_dataset.get_batched_cameras(args.bsz)
            
        # Step 6: Workload Division Strategy
        batched_strategies, gpuid2tasks = start_strategy_final(batched_cameras, strategy_history)
        
        # Step 7: Load Camera Data
        load_camera_from_cpu_to_all_gpu(batched_cameras, batched_strategies, gpuid2tasks)
        
        # Step 8: Preprocessing and All-to-All
        if args.backend == "gsplat":
            batched_screenspace_pkg = gsplat_distributed_preprocess3dgs_and_all2all_final(gaussians, batched_strategies)
        else:
            batched_screenspace_pkg = distributed_preprocess3dgs_and_all2all_final(gaussians, batched_strategies)
            
        # Step 9: Rendering
        if args.backend == "gsplat":
            gsplat_render_final(batched_screenspace_pkg, batched_strategies)
        else:
            render_final(batched_screenspace_pkg, batched_strategies)
            
        batch_statistic_collector = batched_screenspace_pkg["batched_cuda_args"][0]["stats_collector"]
        
        # Step 10: Loss Computation
        batched_losses = batched_loss_computation(
            rendered_images=batched_screenspace_pkg["rendered_images"],
            cameras=batched_cameras,
            compute_locations=batched_screenspace_pkg["compute_locations"],
            strategies=batched_strategies,
            statistic_collectors=batched_screenspace_pkg["batched_cuda_args"]
        )
        
        # Step 11: Backpropagation
        # Combine losses according to formula
        # L_batch = (1 - lambda_dssim) * L0 + lambda_dssim * (1 - L1)
        # Note: batched_losses shape is (N, 2). Index 0 is L0, Index 1 is L1.
        lambda_dssim = args.lambda_dssim
        L0 = batched_losses[:, 0]
        L1 = batched_losses[:, 1]
        loss_sum = torch.mean((1 - lambda_dssim) * L0 + lambda_dssim * (1 - L1))
        
        loss_sum.backward()
        
        # Step 12: Update Strategy
        finish_strategy_final(batched_cameras, strategy_history, batched_strategies, batch_statistic_collector)
        
        # Step 13: Synchronize Losses
        batched_losses_sum = batched_losses.clone()
        dist.all_reduce(batched_losses_sum, op=dist.ReduceOp.SUM, group=DEFAULT_GROUP)
        
        # Step 14: EMA Logging
        batched_loss_cpu = batched_losses_sum.cpu().mean(dim=0) # Mean across batch and components? 
        # Source says "mean(batched_loss_cpu)". Assuming mean over the batch dimension of the tensor.
        # The tensor is (N, 2). Mean over dim 0 gives (2,). Then mean of that? 
        # Let's follow the text: "mean(batched_loss_cpu)". If batched_loss_cpu is the result of all_reduce, it's still (N, 2) on each rank? 
        # No, all_reduce sums. So it's (N, 2) summed across ranks? 
        # Actually, all_reduce with SUM sums the tensor across all ranks. 
        # If each rank has (N, 2), the result is (N, 2) where each element is sum over ranks.
        # Then mean(batched_loss_cpu) likely means mean over the batch dimension N.
        mean_loss = batched_loss_cpu.mean()
        
        if ema_loss_for_log is None:
            ema_loss_for_log = mean_loss
        else:
            ema_loss_for_log = 0.6 * ema_loss_for_log + 0.4 * mean_loss
            
        # Step 15: Evaluation
        if iteration in args.testing_iterations:
            utils.evaluate(gaussians, train_dataset, iteration)
            
        # Step 16: Densification
        if utils.should_densify(iteration):
            if args.backend == "gsplat":
                gsplat_densification(iteration, gaussians, batched_screenspace_pkg)
            else:
                densification(iteration, gaussians, batched_screenspace_pkg)
                
        # Step 17: Saving
        if iteration in args.save_iterations:
            utils.save_gaussians(gaussians, iteration)
            
        if args.save_strategy_history:
            with open(f"strategy_history_{iteration}.json", "w") as f:
                json.dump(strategy_history, f)
                
        if iteration in args.checkpoint_iterations:
            if utils.get_rank() == 0:
                os.makedirs(f"checkpoints/iter_{iteration}", exist_ok=True)
            dist.barrier(group=DEFAULT_GROUP)
            if utils.get_rank() == 0:
                torch.save({
                    'gaussians': gaussians.capture(),
                    'iteration': iteration
                }, f"checkpoints/iter_{iteration}/checkpoint.pth")
                
        # Gradient Scaling
        if args.lr_scale_mode != "accumu":
            for param in gaussians.parameters():
                if param.grad is not None:
                    param.grad /= args.bsz
                    
        # Optimizer Step
        utils.optimizer_step(gaussians)
        
        iteration += args.bsz
        
        if args.profile:
            torch.cuda.nvtx.range_pop()
```

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the rendering and all-to-all communication steps. The adaptive workload division aims to balance the rendering time across GPUs, reducing idle time compared to static partitioning.
*   **Memory Complexity:** Memory usage scales with the number of Gaussians and the batch size. The distributed approach allows larger effective batch sizes by distributing the memory load across multiple GPUs.
*   **Measured Performance:** The source provides performance metrics (throughput, PSNR, running time) for various scenes (e.g., Mip-NeRF 360 scenes) and configurations (1 GPU vs 4 GPUs, batch size 1 vs 4) [S1, unofficial]. Specific numbers are not extracted in the notes, but the method is designed to improve throughput on multi-GPU setups.
*   **Strengths:** Adaptive workload division handles heterogeneous rendering costs for different camera views. Distributed processing enables training on larger scenes or with larger batch sizes.
*   **Limitations:** Requires careful synchronization and communication overhead management. The `args.local_sampling` constraint requires batch size to be divisible by the number of GPUs.
*   **When to use:** When training 3D Gaussian Splatting on large datasets or scenes that require multiple GPUs for feasible training times, and when camera views have varying computational costs.

## Background
This method originates from the Grendel-GS project, an unofficial implementation or extension of 3D Gaussian Splatting training techniques [S1, unofficial]. It addresses the scalability issues of standard 3DGS training by introducing distributed rendering and adaptive workload balancing.

## Sources
[S1] nyu-systems/Grendel-GS, https://github.com/nyu-systems/Grendel-GS, fetched 2026-10-27 (unofficial)
