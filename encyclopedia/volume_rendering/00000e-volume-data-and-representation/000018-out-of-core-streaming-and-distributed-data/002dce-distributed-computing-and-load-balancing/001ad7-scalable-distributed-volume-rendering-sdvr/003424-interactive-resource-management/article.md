# Interactive Resource Management

Also called: Interactive Resource Management Subsystem, voxserver (resource management mode)

## Overview
Interactive Resource Management is a subsystem within the Scalable Distributed Volume Rendering (SDVR) architecture designed to manage cluster resources for visualization tasks in an interactive manner, avoiding traditional batch-fashion resource management systems [S1, unofficial]. It solves the problem of allocating distributed memory nodes for rendering on shared High-Performance Computing (HPC) systems where access is not granted to a single user exclusively [S1, unofficial]. It sets itself apart from its siblings by focusing on the dynamic, real-time negotiation of resources and network connections for interactive volume rendering, rather than static allocation or global data structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendering requests from clients; local utilization status and availability reports from distributed memory nodes [S1, unofficial].
*   **Output:** Assigned rendering jobs; network connections between clients and remote servers; rendered images and transfer functions [S1, unofficial].

**Data Structures:**
*   **Resource Queue:** A list maintained by the resource manager containing available nodes [S1, unofficial]. The exact internal implementation is not specified, but it supports insertion upon availability and removal upon assignment [S1, unofficial].
*   **Job Pair:** A logical pairing of a user request and an allocated resource, forming a rendering job [S1, unofficial].

**Steps:**
1.  **Node Registration:** Distributed memory nodes dedicated to rendering register with the resource manager at any time, particularly when becoming available after a task or when new nodes join the HPC system [S1, unofficial].
2.  **Availability Reporting:** If a node is temporarily unavailable due to heavy utilization, it reports this status to the resource management system, and reports back when available again [S1, unofficial].
3.  **Request Handling:** When a user issues a rendering request, the resource manager checks the resource queue [S1, unofficial].
4.  **Resource Allocation:** The manager waits until enough resources are available to accommodate the request, allocating nodes based on their local utilization [S1, unofficial].
5.  **Job Formation:** The resource manager pairs the request and the allocated resource(s) to form a rendering job [S1, unofficial].
6.  **Connection Establishment:** The allocated resource (remote server) creates a network connection to the remote client [S1, unofficial].
7.  **Event Negotiation:** Rendering events (e.g., camera matrix, transfer function, image data) are negotiated via the network I/O subsystem [S1, unofficial]. The specific I/O functions include `getVolume`, `getMatrix`, `putImage`, and `getTransferFunction` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time

class ResourceManager:
    def __init__(self):
        self.available_nodes = []
        self.node_status = {}
        self.THRESHOLD = 0.8  # [G] Standard utilization threshold for unavailability
        self.SLEEP_INTERVAL = 0.1  # [G] Polling interval in seconds

    def register_node(self, node_id, utilization):
        self.node_status[node_id] = utilization
        self.available_nodes.append(node_id)

    def update_utilization(self, node_id, utilization):
        self.node_status[node_id] = utilization
        if utilization > self.THRESHOLD:
            if node_id in self.available_nodes:
                self.available_nodes.remove(node_id)
        else:
            if node_id not in self.available_nodes:
                self.available_nodes.append(node_id)

    def handle_request(self, request):
        while len(self.available_nodes) < request.required_nodes:
            time.sleep(self.SLEEP_INTERVAL)
        allocated_nodes = self.available_nodes[:request.required_nodes]
        job = Job(request, allocated_nodes)
        for node in allocated_nodes:
            node.create_connection(request.client)
            node.start_rendering(job)
        return job

class Job:
    def __init__(self, request, allocated_nodes):
        self.request = request
        self.allocated_nodes = allocated_nodes
```

**Parameters:**
*   **Discovery Protocol:** Zero Configuration Networking (ZCN) is used for node discovery. Specifically, Bonjour is used on Windows/macOS, and Avahi is used on Linux [S1, unofficial].
*   **Threshold:** The utilization threshold for marking a node as unavailable is not explicitly defined in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Downtime:** The system is designed to avoid downtime when adding new resources at runtime, as ZCN allows nodes to announce availability without restarting the resource manager [S1, unofficial].
*   **Batch vs. Interactive:** Traditional batch systems are unsuitable because volume rendering is intended to be interactive [S1, unofficial].

## Subtopics

### Resource Management with Zero Configuration Networking
A resource management system that uses ZCN to dynamically discover and allocate HPC nodes for rendering tasks without manual configuration or downtime [S1, unofficial]. It integrates into the Virvo DVR library via the `voxserver` tool, acting as a gateway for cluster resources [S1, unofficial]. Nodes register with the resource manager at any time using ZCN [S1, unofficial]. The resource manager maintains available nodes in a queue [S1, unofficial]. When a user issues a rendering request, the manager waits until enough resources are available [S1, unofficial]. The manager pairs the request and resource to form a rendering job, assigned to the allocated resource [S1, unofficial]. The resource creates a network connection to the remote client [S1, unofficial]. Rendering events are negotiated via the network I/O subsystem [S1, unofficial]. The discovery protocol uses Bonjour on Windows/macOS and Avahi on Linux [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Interactive resource allocation; dynamic discovery of HPC nodes; no downtime for adding resources; transparent access to cluster resources [S1, unofficial].
*   **Limitations:** Relies on ZCN availability; specific performance metrics are not provided [S1, unofficial].
*   **When to Use:** When interactive volume rendering on shared HPC systems is required, and dynamic resource allocation is needed [S1, unofficial].
*   **When Not to Use:** When batch processing is sufficient, or when ZCN is not available [S1, unofficial].

## Background
Interactive Resource Management is part of the software architecture for distributed volume rendering on HPC systems, introduced in the paper "A SOFTWARE ARCHITECTURE FOR DISTRIBUTED VOLUME RENDERING ON HPC SYSTEMS" [S1, unofficial]. It is implemented using the Virvo DVR library and the `voxserver` tool [S1, unofficial]. The approach structures the problem in a pipeline fashion with loosely coupled stages, allowing for generic replacement with custom logic using subclassing [S1, unofficial]. It aims to hide the complexities of heterogeneous HPC systems [S1, unofficial].

## Sources
[S1] A SOFTWARE ARCHITECTURE FOR DISTRIBUTED VOLUME RENDERING ON HPC SYSTEMS, https://actapress.com/Abstract.aspx?paperId=454770, fetched 2026-10-27 (unofficial)
