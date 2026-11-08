# Distributed Volume Rendering Pipeline

Also called: Distributed Volume Rendering Architecture, Distributed software architecture for volume rendering, Virvo Architecture, Virvo system

## Overview
The Distributed Volume Rendering Pipeline is a software architecture for direct volume rendering on heterogeneous High-Performance Computing (HPC) systems, structured as a pipeline of loosely coupled stages that can be replaced via subclassing [S1, unofficial]. It decouples elementary renderers from parallelism and resource management, utilizing Zero Configuration Networking (ZCN) for dynamic node allocation in distributed and Multi-GPU environments [S1, unofficial]. Unlike traditional batch-style resource management, it exposes domain-specific hardware metrics (e.g., CUDA compute capability, video memory) to enable interactive, load-balanced rendering without system restarts [S1, unofficial]. It sets itself apart from its siblings by explicitly defining a dedicated **display phase** as a distinct pipeline stage to handle varying interactive requirements (e.g., VR head tracking vs. low-frame-rate monitoring) and by using ZCN for seamless dynamic resource addition/removal [S1, unofficial].

## How it works
The system implements a **sort-last parallel volume rendering** pipeline [S1, unofficial]. While sort-first parallelism is noted as easily accommodable (resulting in a generalization of the compositing stage), the primary implementation restricts itself to sort-last parallelism to achieve data parallelism [S1, unofficial].

### Inputs and Outputs
*   **Input:** A volume dataset (potentially multi-time-step), a viewing transformation, and a rendering request specifying hardware constraints or preferences.
*   **Output:** A composited image displayed on a remote client, potentially with specific frame-rate requirements (e.g., high-frame-rate for VR, low-frame-rate for simulation monitoring) [S1, unofficial].

### Data Structures
The architecture relies on a **Resource Manager** and **Resources** (worker nodes).
*   **Resource Registration:** Resources register with the Resource Manager using Zero Configuration Networking (ZCN) [S1, unofficial].
*   **Resource List:** The Resource Manager maintains a list of active resources. This list is updated dynamically:
    *   When a resource finishes a job, it re-registers via ZCN [S1, unofficial].
    *   When an idle resource becomes unavailable, its status is broadcast via ZCN, and it is removed from the active list [S1, unofficial].
    *   If a socket connection breaks during a job, the client is notified of the resource's unavailability, but the resource management subsystem remains unaffected due to loose coupling [S1, unofficial].
*   **Renderer Selection:** The Resource Manager selects a suitable node based on hardware-specific metrics, such as available/used video memory or CUDA compute capability [S1, unofficial].

### Pipeline Stages
The pipeline consists of four stages [S1, unofficial]:

1.  **Data Distribution:** A sequential task assigned to a single processor. It distributes the volume data to the rendering nodes.
2.  **Rendering:** Performed by multiple processors in parallel.
    *   If a single resource is assigned to one job, an **elementary renderer** is instantiated [S1, unofficial].
    *   If a request requires multiple resources, a **parallel brick renderer** is instantiated, configured for sort-last parallel Multi-GPU or distributed memory rendering [S1, unofficial].
    *   Parallelism is achieved by superimposing a renderer that maintains a **k-d tree search structure** and handles threading on top of the elementary renderers [S1, unofficial]. This separates parallelism logic from the core rendering task. The visibility sorting data structure is exposed for replacement, but the implementation retains the k-d tree to avoid performance degradation compared to ray tracing [S1, unofficial].
3.  **Compositing:** Performed by multiple processors in parallel. It composites the sub-images rendered by the parallel nodes.
    *   **Exception:** If a single resource is assigned to one job, the compositing stage is omitted, and execution proceeds directly from rendering to the display phase [S1, unofficial].
4.  **Display:** An additional pipeline stage that determines how the rendered image retrieved from the remote server (responsible for compositing) is displayed on the remote client [S1, unofficial]. This stage accommodates varying interactive requirements, such as high-frame-rate VR head tracking or low-frame-rate HPC simulation monitoring [S1, unofficial].

### Pseudocode
The following pseudocode illustrates the interaction between the Client, Resource Manager, and Resources. It is written in Python-like syntax.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Resource:
    """Represents a worker node (server) capable of rendering."""
    def __init__(self, node_id, cuda_capability, video_memory_total, video_memory_used):
        self.node_id = node_id
        self.cuda_capability = cuda_capability
        self.video_memory_total = video_memory_total
        self.video_memory_used = video_memory_used
        self.is_available = True
        self.socket = None  # Socket connection to client

    def register_with_manager(self, manager):
        """Register availability via Zero Configuration Networking (ZCN)."""
        manager.add_resource(self)
        # ZCN protocol details are abstracted here [S1, unofficial]

    def unregister_from_manager(self, manager):
        """Remove from active list via ZCN broadcast."""
        manager.remove_resource(self)
        # ZCN broadcast of unavailability [S1, unofficial]

    def handle_render_request(self, request):
        """
        Instantiate the appropriate renderer based on resource count.
        """
        if request.requires_single_resource():
            renderer = ElementaryRenderer()
        else:
            renderer = ParallelBrickRenderer()
            renderer.configure_for_sort_last_parallel()
        
        result = renderer.render(request)
        return result

    def check_connection(self):
        """Monitor socket connection."""
        if not self.socket.is_connected():
            self.is_available = False
            self.unregister_from_manager(self.manager)
            # Notify client of unavailability [S1, unofficial]

class ResourceManager:
    """Manages available resources using ZCN and hardware metrics."""
    def __init__(self):
        self.active_resources = []
        self.zcn_listener = ZCNListener()
        self.zcn_listener.on_resource_available = self.on_resource_available
        self.zcn_listener.on_resource_unavailable = self.on_resource_unavailable

    def on_resource_available(self, resource):
        self.active_resources.append(resource)

    def on_resource_unavailable(self, resource):
        if resource in self.active_resources:
            self.active_resources.remove(resource)

    def select_resource(self, request):
        """
        Select a suitable node based on hardware-specific metrics.
        Bypasses traditional batch-style management [S1, unofficial].
        """
        # Filter resources that can handle the request
        candidates = [r for r in self.active_resources if r.is_available]
        
        if not candidates:
            return None
        
        # Sort by available video memory or CUDA capability
        # Specific metric selection is implementation-dependent [S1, unofficial]
        candidates.sort(key=lambda r: r.video_memory_total - r.video_memory_used, reverse=True)
        return candidates[0]

class DistributedVolumeRenderer:
    """
    The main pipeline orchestrator.
    """
    def __init__(self, resource_manager):
        self.resource_manager = resource_manager
        self.client = Client()

    def process_request(self, request):
        """
        Execute the distributed volume rendering pipeline.
        """
        # 1. Data Distribution (Sequential, single processor)
        # Distribute data to selected resources [S1, unofficial]
        
        # 2. Select Resources
        resources = self.resource_manager.select_resources_for_request(request)
        
        if not resources:
            # Handle error or wait for resources via ZCN dynamic addition [S1, unofficial]
            return

        # 3. Rendering (Parallel)
        # If single resource:
        if len(resources) == 1:
            resource = resources[0]
            rendered_image = resource.handle_render_request(request)
            # Skip compositing, proceed to display [S1, unofficial]
            self.display_phase(rendered_image, request.display_requirements)
        else:
            # Multiple resources: Parallel Rendering
            sub_images = []
            for resource in resources:
                # Spawn parallel brick renderer on resource
                sub_image = resource.handle_render_request(request)
                sub_images.append(sub_image)
            
            # 4. Compositing (Parallel)
            composited_image = composite(sub_images)
            
            # 5. Display
            self.display_phase(composited_image, request.display_requirements)

    def display_phase(self, image, requirements):
        """
        Dedicated stage for display, handling varying frame-rate needs.
        E.g., high-frame-rate for VR, low-frame-rate for monitoring [S1, unofficial].
        """
        # Display logic specific to requirements [S1, unofficial]
        pass

    def monitor_resources(self):
        """
        Continuously monitor socket connections and resource status.
        """
        while True:
            for resource in self.resource_manager.active_resources:
                resource.check_connection()
            # ZCN listener handles dynamic additions/removals [S1, unofficial]
            time.sleep(1)
```

### Parameters and Typical Values
*   **Parallelism Strategy:** Sort-last parallel rendering [S1, unofficial].
*   **Networking Protocol:** Zero Configuration Networking (ZCN) for dynamic node allocation [S1, unofficial].
*   **Hardware Metrics:** CUDA compute capability, available/used video memory [S1, unofficial].
*   **Renderer Types:** Elementary Renderer (single resource), Parallel Brick Renderer (multiple resources) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Resource Unavailability:** If a socket connection breaks during a job, the client is notified, but the resource management subsystem is not affected due to loose coupling [S1, unofficial].
*   **Generality Limitation:** The handcrafted resource management system lacks generality and is restricted to direct volume rendering; supporting alternative visualization methods requires explicit adaptation of the management subsystem [S1, unofficial].
*   **Visibility Sorting Structure:** Exposing the visibility sorting data structure for replacement was considered but rejected to avoid performance degradation compared to ray tracing [S1, unofficial].

## Subtopics

### [Remote Rendering Client/Server Architecture](8196-remote-rendering-client-server-architecture.md)
A software architecture where remote clients establish network connections to remote servers that own renderers, enabling distributed rendering, compositing, and display. It decouples the user interface and interaction logic from the heavy computational volume rendering tasks, allowing the rendering engine to run on separate hardware. This approach facilitates sort-last parallel rendering on distributed memory systems and heterogeneous HPC clusters by allowing multiple server nodes to render partial views or bricks which are then composited. It sets itself apart from local rendering by introducing a network transport layer that can be specialized for different hardware topologies (e.g., shared memory vs. InfiniBand).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The system scales to heterogeneous hardware platforms, ranging from high-end graphics workstations to dedicated render clusters with Multi-GPU nodes and large distributed memory systems with no graphics accelerators [S1, unofficial]. It supports dynamic load balancing by adding resources via ZCN if the render request queue exceeds current capacity [S1, unofficial].
*   **Strengths:**
    *   Versatility across heterogeneous HPC systems [S1, unofficial].
    *   Interactive resource management tailored for visualization, exposing domain-specific hardware metrics [S1, unofficial].
    *   Zero downtime for resource changes via ZCN; users do not need to resend requests when the system is altered [S1, unofficial].
    *   Dedicated display phase accommodates varying interactive requirements (VR vs. monitoring) [S1, unofficial].
*   **Limitations:**
    *   Lack of generality: The resource management system is restricted to direct volume rendering and requires explicit adaptation for other visualization methods [S1, unofficial].
    *   Handcrafted nature: Lacks the generality of traditional batch-style resource management systems [S1, unofficial].
*   **When to Use:** When rendering on heterogeneous HPC systems (including Multi-GPU and distributed memory) with interactive requirements, and when dynamic resource allocation is needed without system restarts [S1, unofficial].
*   **When Not to Use:** When supporting visualization methods other than direct volume rendering is required without significant adaptation of the management subsystem [S1, unofficial].

## Background
The Distributed Volume Rendering Pipeline was introduced by Virvo et al. in the paper "A Software Architecture for Distributed Volume Rendering on HPC Systems" [S1, unofficial]. The work addresses the need for a versatile software architecture that can hide the complexities of running distributed volume rendering on highly versatile HPC systems. It builds on the parallel volume rendering pipeline from [18] (Molnar et al. [10] is cited for sorting classification) but extends it with multi-user support and a dedicated display phase [S1, unofficial]. The system was designed to support remote data exploration using thin clients and virtual reality applications [S1, unofficial].

## Sources
[S1] A SOFTWARE ARCHITECTURE FOR DISTRIBUTED VOLUME RENDERING ON ... - Actapress, https://actapress.com/Abstract.aspx?paperId=454770, fetched 2026-10-27 (unofficial)
