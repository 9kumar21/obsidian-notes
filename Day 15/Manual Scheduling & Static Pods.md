###### Table of Contents

- Prerequisite: Revisiting Kubernetes Deployment Workflow
- Understand the Kubernetes Scheduler
- Manual Scheduling 
	- Why is Manual Scheduling Required?
	- What is Manual Scheduling?
	- How is Manual Scheduling?
	- Demonstration: Assigning a Pod to a Node
	- Running a Pod on the Control Plane
	- How Can Control Plane Run Workloads?
- Static Pods
	- Why Do We Need Static Pods?
	- What Are Static Pods?
	- How Are Static Pods Useful?
	- Mirror Pods in Kubernetes?
	- Demonstration: Creating a Static Pod
	- Why Deleting Static Pods via `kubectl` Doesn't work
	- Accessing Static Pods in Production vs. KIND
- Key Differences Between Manual Scheduling & Static Pods
- Summary
- References
- ---
###### Prerequisite: Revisiting Kubernetes Deployment Workflow

Before we dive into manual scheduling and static pods, it's essential to recall how the Kubernetes schedular works.

###### Understand the Kubernetes Schedular

![[Pasted image 20260730085043.png]]
The Kubernetes Schedular is responsible for automatically placing pods on available worker nodes based on factors like:

- **Resource availability** (CPU, Memory).
- **Taints and tolerations** (node restrictions, discussed in Day 16).
- **Affinity and anti-affinity rules** (Discussed in Day 17).

However, **can we bypass the scheduler and manually assign pods to nodes**?
Yes! This is where **manual scheduling** comes in.

----
###### Manual Scheduling 

Why is Manual Scheduling Required?

- **Troubleshooting & Debugging:** Helps diagnose scheduling issues by placing a pod on a specific node.
- **Testing Node-Specific Workloads:** Ensures an application runs on a specific node (e.g., a database pod requiring an SSD).
- Kubernetes Scheduling Is Disabled: If the scheduler is down, you can manually schedule pods as a fallback.
---

What is Manual Scheduling?

Manual scheduling means explicitly assigning a pod to a node using the `nodeName` fields in the pod's YAML manifests. This completely bypasses the **Kubernetes scheduler**.

---
###### How is Manual Scheduling Useful?

- Guarantees that a pod runs on a particular node.
- Useful when a workload requires special hardware or node-specific configurations.
- Helps troubleshooting whey a pod isn't scheduled automatically.
---
