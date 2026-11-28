# Overlapped Implementation

Also called: Asynchronous Disk Access and Texture Loading, Multi-threaded overlapped rendering, asynchronous loading

## Overview
The Overlapped Implementation is a rendering technique that hides data transfer overhead by fully overlapping data loading, volume slicing, and rendering [S1]. It addresses the performance bottleneck of out-of-core volume roaming, where sequential execution of cache scanning, data fetching, and rendering drastically reduces frame rates [S1]. It sets itself apart from sequential implementations by using multi-threading and asynchronous I/O to allow rendering to proceed while data is still being fetched from storage or the network [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A request for a set of volume bricks from the volume renderer; access to a hierarchical cache system (DHCS) and storage (local disks, network).
*   **Output:** Rendered frames of volume slices; updated local memory state for inter-node communication.

**Data Structures and Threading Model:**
The system relies on a specific multi-threaded architecture consisting of five distinct threads [S1]:
1.  **Application Thread (Master):** Coordinates the overall system.
2.  **Volume Renderer Thread (Slave):** Computes proxy geometries and renders bricks.
3.  **DHCS Thread (Slave):** Manages the Distributed Shared Memory cache hierarchy and initiates data loads.
4.  **GPU Loader Thread (Slave):** Handles texture loading into graphics memory.
5.  **Async Load Threads (Slaves):**
    *   *Async Load HDD to RAM:* Uses Linux `libaio` for asynchronous disk I/O [S1].
    *   *Async Load LAN to RAM:* Uses a TCP client-server protocol for network fetching [S1].

**Execution Flow:**
1.  The **Volume Renderer** requests a series of bricks from the **DHCS** [S1].
2.  The **DHCS** scans its cache hierarchy (local memory, other cluster nodes) to locate the bricks [S1].
3.  The **DHCS** launches asynchronous data loads. If data is on disk, the HDD-to-RAM thread uses `libaio` [S1]. If data is on a remote node, the LAN-to-RAM thread uses TCP [S1].
4.  While data is loading, the **Volume Renderer** computes proxy geometries for the requested bricks. In hardware-accelerated rendering, these are view-aligned back-to-front slices [S1].
5.  As soon as a brick is loaded into graphics memory, the **Volume Renderer** renders it in back-to-front order. Rendering begins for the first loaded bricks before all requests are satisfied [S1].
6.  Once all asynchronous loads are launched, the local memory state at the end of the current frame is known. This allows binary all-to-all communication (for cache coherence or distributed updates) to be launched overlapped with the rendering process [S1].

**OpenGL Context Management:**
The GPU Loader thread cannot load textures directly into the main application's rendering context due to OpenGL restrictions [S1]. A new OpenGL context with resource sharing must be created to allow the loader thread to upload textures to the GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Threading structure and execution flow
# Note: This is a logical representation of the concurrent flow described in [S1].

class OverlappedRenderer:
    def __init__(self):
        # Create shared OpenGL context for GPU loader
        self.shared_context = create_opengl_context_with_resource_sharing()
        
        # Threads
        self.dhcs_thread = Thread(target=self.dhcs_worker)
        self.renderer_thread = Thread(target=self.render_worker)
        self.gpu_loader_thread = Thread(target=self.gpu_loader_worker, context=self.shared_context)
        self.hdd_async_thread = Thread(target=self.hdd_async_io)
        self.lan_async_thread = Thread(target=self.lan_async_io)

    def start(self):
        self.dhcs_thread.start()
        self.renderer_thread.start()
        self.gpu_loader_thread.start()
        self.hdd_async_thread.start()
        self.lan_async_thread.start()

    def request_bricks(self, brick_ids):
        """Called by the application thread"""
        # 1. Volume renderer requests bricks from DHCS
        self.dhcs_thread.queue_request(brick_ids)

    def dhcs_worker(self):
        """DHCS Thread: Scans cache and launches async loads"""
        while True:
            brick_ids = self.dhcs_thread.get_request()
            
            # 2. Scan cache hierarchy
            locations = self.scan_cache_hierarchy(brick_ids)
            
            # 3. Launch asynchronous data loads
            for brick_id, location in locations.items():
                if location == 'HDD':
                    self.hdd_async_thread.load(brick_id)
                elif location == 'LAN':
                    self.lan_async_thread.load(brick_id)
            
            # 6. Launch all-to-all communication overlapped with rendering
            # The local memory state at the end of the frame is now known
            self.launch_all_to_all_communication()

    def render_worker(self):
        """Volume Renderer Thread: Computes proxies and renders"""
        while True:
            # 4. Compute proxy geometries (view-aligned back-to-front slices)
            # This happens while data is loading
            proxies = self.compute_proxy_geometries()
            
            # 5. Render bricks as soon as they are loaded in graphics memory
            # Render in back-to-front order
            for brick in self.get_loaded_bricks():
                if brick.ready:
                    self.render_slice(brick, proxies[brick.id])

    def gpu_loader_worker(self, context):
        """GPU Loader Thread: Handles texture upload"""
        # Must use shared context, not main rendering context
        with context:
            while True:
                brick = self.get_bricks_ready_for_gpu()
                if brick:
                    self.upload_texture_to_gpu(brick.data)

    def hdd_async_io(self):
        """Async Load HDD to RAM Thread"""
        # Uses Linux libaio for asynchronous I/O
        while True:
            brick_id = self.hdd_async_thread.get_request()
            data = self.read_from_disk_async(brick_id)
            self.mark_as_loaded(brick_id, data)

    def lan_async_io(self):
        """Async Load LAN to RAM Thread"""
        # TCP client-server protocol
        while True:
            brick_id = self.lan_async_thread.get_request()
            data = self.fetch_from_network_async(brick_id)
            self.mark_as_loaded(brick_id, data)
```

## Operating characteristics
*   **Performance:** Sequential implementation (synchronous requests) dramatically decreases performance because data loading, cluster memory state update, volume slicing, and rendering are executed serially [S1]. The overlapped implementation optimizes volume roaming by hiding these latencies [S1].
*   **Network Acceleration:** Using two Gigabit Ethernet network interfaces per node accelerates data fetching by a factor of 4 compared to directly accessing local disks [S1].
*   **Strengths:** Enables optimal volume roaming for extremely large datasets by preventing the renderer from stalling on I/O [S1].
*   **Limitations:** Requires complex multi-threading and context management (specifically OpenGL resource sharing) [S1].

## Background
The Overlapped Implementation was proposed by Castanié et al. in the paper "Distributed Shared Memory for Roaming Large Volumes" [S1]. It builds upon their previous cache system [8] to add fully asynchronous loading in memory and graphics memory [S1]. The work is part of the broader field of Distributed Shared Memory (DSM) for PC clusters, aiming to handle large time-varying volumes from scientific domains like seismic captors, CT scanners, and numerical simulations (FEM, CFD) [S1].

## Sources
[S1] Distributed Shared Memory for Roaming Large Volumes, https://doi.org/10.1109/tvcg.2006.135, fetched 2026-10-26
