# LiDAR Ray Simulation

Also called: NeRF-LiDAR Ray Simulation

## Overview
Simulates real LiDAR sensor rays by defining ray origin and direction based on sensor parameters and ego-vehicle motion to generate 3D points [S1]. It differs from standard ray casting by explicitly modeling the temporal evolution of the sensor origin due to vehicle velocity and incorporating a learned "raydrop" probability to mimic real-world signal loss, rather than assuming perfect reflection [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** NeRF implicit representation (for depth/distance $\hat{D}$), LiDAR sensor parameters (channels, FOV, spin rate), ego-vehicle velocity $v$, and time interval $\Delta t$ [S1].
*   **Outputs:** A set of 3D points $p$ and their associated semantic/depth labels, forming a simulated LiDAR frame $\hat{P}$ [S1].

**Data Structures**
*   **Ray Parameters:** Defined by azimuth $\theta$ and vertical angle $\phi$ [S1].
*   **Equirectangular Grid:** A 2D image grid of resolution $H \times W$ used for point-wise alignment. For a 32-channel LiDAR (e.g., nuScenes), this is $32 \times 1024$ [S1].

**Numbered Steps**
1.  **Determine Ray Direction:** For each laser channel, calculate the azimuth $\theta$ and vertical angle $\phi$ based on the LiDAR's mechanical settings and the time interval $\Delta t$ within the scan cycle [S1]. Compute the direction vector $d$ using:
    $$d = (\cos \theta \cos \phi, \sin \theta \sin \phi, \cos \phi)^T$$
    [S1]
2.  **Update Ray Origin:** Calculate the current sensor origin $o$ by updating the previous origin $o_0$ with the ego-vehicle's velocity $v$ over the time interval $\Delta t$:
    $$o = o_0 + \Delta t \cdot v$$
    [S1]
3.  **Query NeRF for Distance:** Cast the ray defined by $o$ and $d$ into the NeRF. Sample along the ray to find the distance $\hat{D}$ from the sensor to the nearest surface (or integrate density to find the expected depth) [S1].
4.  **Compute 3D Point:** Calculate the simulated point $p$ in 3D space:
    $$p = o + \hat{D}d$$
    [S1]
5.  **Apply Raydrop (Optional/Learned):** To match real-world sensor behavior where points are dropped due to low reflectance or incidence angle, apply a learned probability mask. This is often done by projecting both real point cloud $P$ and simulated point
