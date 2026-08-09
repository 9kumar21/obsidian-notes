###### Table of Contents

- Introduction: Why Taints and Tolerations?
- When Do We need Taints & Tolerations?
- Understanding Taints
- Understanding Tolerations
- `nodeSelector` vs Taints & Tolerations
- How Taints and Tolerations Work Together
- Demonstration
- Deleting Taints
- Advanced Concepts of Kubernetes Taints & Tolerations
- Key Takeaways
- References
---
###### Introduction: Why Taints and Tolerations?

In Kubernetes, **Taints and Tolerations** help **control which pods can be scheduled on which nodes**. They allow cluster administrators to **prevent certain workloads from running on specific nodes** while allowing exceptions when necessary.

###### When Do We Need Taints & Tolerations?

Taints and tolerations are useful in several scenarios:

| Use Case                                     | Explanation                                                                  |
| -------------------------------------------- | ---------------------------------------------------------------------------- |
| Dedicated Nodes                              | Taint nodes with a GPU to restrict them to **GPU-intensive workloads** only. |
| Isolating Critical Workloads                 | Ensure **critical workloads** run separately from **non-critical** ones.     |
| Preventing Scheduling on Control Plane Nodes | Control plane nodes should only run **Kubernetes system scheduled**.         |
| Maintenance Mode                             | Taint a node under **maintenance** to stop new pods from being scheduled     |
| Node Resource Constraints                    | Prevent pods from being scheduled on nodes with **low memory and CPU**.      |
**Important:**
Taints and tolerations only apply during scheduling. If a pod is already running on a node, adding a taint will not remove the existing pod (unless you use the **NoExecute** effect)

---
Understanding Taints

A **Taint** is applied to a **node** to indicate that it **should not accept certain pods** unless they explicitly tolerate it.

A taint tells Kubernetes:
"Don't place Pods on this node unless they have permission (toleration)."

###### How to Apply a Taint?

Taints are applied to nodes using the following command:
```
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

Example:
```
kubectl taint nodes my-second-cluster-worker storage=ssd:NoSchdule
kubectl taint nodes my-second-cluster-worker2 storage=hdd:NoSchedule
```

- These Commands taint `my-second-cluster-worker` to **only allow pods that tolerate** `storage=ssd:Noschedule` and `my-second-cluster-worker2` to **only allow pods that tolerate** `storage=hdd:NoSchedule`.
---

###### Effects of a Taint

There are 3 effects of taints.

| Effect              | Behavior                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. NoSchedule       | - Only Pods with a matching toleration can be scheduled.<br>- Existing Pods continue running.<br><br>Example:<br>Node A<br>[Taint: NoSchedule]<br><br>Result:<br><br>- ❌ New Pod → Not allowed<br>- ✅ Existing Pods → Continue running<br>- ✅ Pod with toleration → Allowed                                                                                                                                     |
| 2. PreferNoSchedule | Kubernetes tries not to schedule Pods here. It's not a strict rule.<br><br>If K8s has no better node available, it may still schedule the Pod here.<br><br>Example:<br>Imagine 3 nodes:<br>Node1<br>Node2<br>Node3 (PreferNoSchedule)<br><br>Scheduler will try:<br>Node1 ✅<br>Node2 ✅<br>if both are full:<br>Node3 ✅ (allowed as a last option)<br><br>Think of it as:<br>Please avoid this node if possible. |
| 3. NoExecute        | This effect does 2 things:<br>1. ❌ Don't schedule new pods.<br>2. 🚪Remove (evict) existing Pods that don't tolerate the taint.<br><br>Example:<br>Initially:<br>Node<br> ├── Pod A<br> ├── Pod B<br> └── Pod C<br><br> Now apply:<br> NoExecute<br><br> Result:<br>Pod A ❌ Evicted<br>Pod B ❌ Evicted<br>Pod C ✅ (has toleration)<br><br>Note: So NoExecute affects both new and existing Pods.                |
###### Easy Comparison

|Effect|New Pods|Existing Pods|
|---|---|---|
|`NoSchedule`|❌ Not scheduled|✅ Keep running|
|`PreferNoSchedule`|⚠️ Avoid if possible|✅ Keep running|
|`NoExecute`|❌ Not scheduled|❌ Evicted (unless tolerated)|
# Easy Memory Trick

- **`NoSchedule`** → **"Don't come in."**
- **`PreferNoSchedule`** → **"Please don't come in."**
- **`NoExecute`** → **"Don't come in, and everyone already inside must leave (unless they have permission)."**
----
###### Understanding Tolerations

A Toleration is applied to a Pod.
In the simplest terms:

> **A Toleration is a "Permission Pass" for a Pod.**

If a **taint** is a **"Keep Out" sign** on a node, then a **toleration** is a **special pass** that lets a Pod ignore that sign.

### Real-life example

Imagine a building with this sign:

🚫 **"Employees Only"**

- A visitor ❌ cannot enter.
    
- An employee with an ID card ✅ can enter.
    

Here:

- 🏢 **Node** = Building
    
- 🚫 **Taint** = "Employees Only" sign
    
- 🪪 **Toleration** = Employee ID card
    
- 📦 **Pod** = Person trying to enter
    

---

### Kubernetes example

Suppose a node has this taint:

```text
NoSchedule
```

A normal Pod:

```text
Pod
   ↓
Node (NoSchedule)
```

Result:

❌ Pod is **not scheduled**.

---

A Pod with a matching toleration:

```text
Pod + Toleration
        ↓
Node (NoSchedule)
```

Result:

✅ Pod **can be scheduled** on that node.

---

## One important point

A **toleration does NOT force a Pod onto a node**.

It only says:

> **"This Pod is allowed to run on this tainted node."**

The Kubernetes scheduler **may still choose another suitable node** if one is available.

---

## Easy way to remember

- **Taint** = 🚫 **Keep Out** sign.
    
- **Toleration** = 🪪 **Permission Pass** to ignore the sign.
    
- **Node** = 🏢 Building.
    
- **Pod** = 📦 Person trying to enter.
    

**Memory Trick:**

> **Taint blocks. Toleration allows.**

---
###### Toleration Operations

| Operator        | Behavior                                              |
| --------------- | ----------------------------------------------------- |
| Equal (default) | The key and value must exactly match the taint.       |
| Exists          | Only the key needs to match, and the value is ignored |
Example using Exists:
```
tolerations:
  - key: "storage"
    operator: "Exists"
    effect: "NoSchedule"
```
- This toleration allows the pod to be scheduled on **any node that has a** `storage` **taint**, regardless of its value.
- ----
###### How Taints and Tolerations Work Together

| Node Taint               | Pod Toleration    | Effect                         |
| ------------------------ | ----------------- | ------------------------------ |
| `storage=ssd:NoSchedule` | Toleration exists | ✅ Pod can be scheduled         |
| `storage=ssd:Noschedule` | ❌No toleration    | ❌Pod cannot be scheduled.      |
| `storage=ssd:NoExecute`  | Toleration exists | ✅Pod remains on the node.      |
| `storage=ssd:NoExecute`  | ❌No toleration    | ❌Pod is evicted from the node. |
###### Cluster Setup for Demonstration

We have a total of three nodes in our KIND cluster:

1. ***my-second-cluster-control-plan*** → This is the control-plane node responsible for managing the cluster.
   
2. ***my-second-cluster-worker*** → This is worker node where application workloads can be scheduled.
   
3. ***my-second-cluster-worker2*** → This is another worker node available for scheduling workloads.
   
   ###### Applying Taints
```
   kubectl taints nodes my-second-cluster-worker storage=ssd:Noschedule
   kubectl taints nodes my-second-cluster-worker2 storage=hdd:NoSchedule
```

###### Verifying Taints
```
kubectl describe node  my-second-cluster-worker | grep -i taint
kubectl describe node my-second-cluster-worker2 | grep -i taint
```
###### Checking Pod Behavior Without Tolerations
```
kubectl run mypod --image=nginx
kubectl get pods
kubectl describe pod mypod
```
- The Pod remains in Pending state because it does not tolerate any taint.
----
###### Why the Pod Is Not Scheduled on the Control Plane Node?

After applying taints to our worker nodes, we will attempt to create a pod without any tolerations using:
```
kubectl run mypod --image=nginx
```
Since both worker nodes are tainted, the pod will remain in a Pending state. However, it will not be scheduled on the control plane node either.

**Checking Why the Pod Is Not Scheduled**

`kubectl describe pod mypod`


This will show that **no suitable nodes were found for scheduling** due to the applied taints. Additionally, the control plane node is already tainted by default, preventing general workloads from running on it.

**Verifying the Taint on the Control Plane Node**

We can check the taints applied to the control plane node using:
```
kubectl describe node my-second-cluster-control-plane | grep taints
```
*Expected output:*
`Taints: node-role.kubernetes.io/control-plane:Noschedule`

This confirms that the control plane node has a taint that prevents regular workloads from being scheduling on it unless the pod has a corresponding  toleration.

###### Why Were we able to Manually Schedule a Pod on the Control Plane in the Previous Lecture?
In the previous lecture, we manually scheduled a pod on the control plane node by specifying the (`nodeName` filed )in the pod definition. This bypasses the Kubernetes scheduler entirely.

📌*Key takeaway:*
- **Taints and tolerations affect scheduling decisions made by the scheduler.**
- **When using manual scheduling (`nodeName` field), the scheduler is not involved, so taints are ignored.**

###### `nodeSelector` vs Taint & Toleration

- `nodeSelector` - Schedule this pod on nodes with these labels.
  
- **Taints & Tolerations** - Don't schedule Pods on this node unless they have the matching toleration.
  
We will now proceed with applying tolerations to allow pods to be scheduled despite the taints.
###### Scheduling Pods with Tolerations
```
apiVersion: v1
kind: Deployment
metadata:
	name: app1-deploy
spec:
	replicas: 3
	selector:
		matchLabels:
			app: app1
	template:
		metadata:
			labels:
				app: app1
		spec:
			tolerations:
			- key: "storage"
			  operator: "Equal"
			  vaule: "ssd"
			  effect: "NoSchedule"
			containers:
			- name: nginx
			  image: nginx
```
- This deployment will now schedule pods on `my-second-cluster-worker`
###### Using the Exists Operator
```
apiVersion: v1
kind: Pod
metadata:
	name: mypod2
spec:
	containers:
	- name: nginx
	  image: nginx
	tolerations:
	- key: "storage"
	  operator: "Exists" 
	  effect: "NoSchedule"
```
- This pod can be placed on either `my-second-cluster-worker` or `my-second-cluster-worker2`

###### Applying Multiple Taints and Tolerations

```
kubectl taint nodes my-second-cluster-worker env=prod:NoSchedule
kubectl taint nodes my-second-cluster-worker2 env=dev:NoSchedule
```

```
tolerations:
- key: "storage"
  opertor: "Exists"
  effect: "NoSchedule"
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"
```
- This pod tolerates any `storage` taint and only `env=prod:NoSchedule`
---
###### Deleting Taints
Once a taint is applied to a node, it restricts pod scheduling based on the taint effect. If you need to `remove a taint` from a node, you can do so using the following syntax:
```
kubectl taint node <node-name> <key>=<value>:effect-
```
Here, the `-` **(hyphen) at the end** tells K8s to **remove the taint**.

*Example*: **Removing a Single Taint**
Let's remove the `storage=ssd:NoSchedule` taint from **my-second-cluster-worker:**
```
kubectl taint node my-second-cluster-worker storage=ssd:NoSchedule-
```
Now, the **ssd taint is removed**, and pods that don't have the matching toleration can be scheduled on this node.

###### Verify Taint Removal
After deleting the taints, verify that they are removed using:
```
kubectl describe node my-second-cluster-worker | grep -i taints
```
If no taints are listed in the output, the nodes are now available to schedule any pod without restrictions.

-----
###### Advanced Concepts of Kubernetes Taints & Tolerations 

In K8s, taints and tolerations control where pods can be scheduled. This analogy will help you visualize how these concepts work together with scheduler preferences.

----
###### Scenario: A Cluster with Colored Nodes
Imagine we have four nodes in our K8s cluster, each with different taints and behaviors.

![[Pasted image 20260801160108.png]]

1. Green Node 🟢
   
     - **Taints:** `color=green:NoScheduler` 
     - **Effect:** Only pods that have a matching toleration (color=green) can be scheduled here.
     - **Current Pods:** *Two green pods* (already running).
    
2. Blue Node 🔵
	- **Taint:** `color:blue:PreferNoSchedule`
	- **Effect:** The Scheduler tries to avoid placing pods here unless necessary.
	- **Current Pods:** **One blue pod** (already running).
	  
3. Purple Node 🟣
	- **Taints:** `color=green:NoExcute`
	- **Effect:** Any pod **without a matching toleration** will be immediately **evicted** from this node.
	- **Current Pods:** **Two purple pods** (already running).

4. Untainted Node ⚪
	- A new node has been added to the cluster.
	- **No taints applied** → Any pod **can be placed here** freely.

-----
###### Effect of Taints on Existing Pods

- **Pods in the Green Node** (`NoSchedule`) and **Blue Node**(`PreferNoSchedule`) remain unaffected.
- **Pods in the Purple Node** (`NoExecute`) will be evited if they **lack a matching toleration**.
	- If you wish to delay **eviction**, you can use the `tolerationSeconds` parameter.

---
###### Pod Placement Behavior for New Pods

![[Pasted image 20260801162359.png]]

Now, let's introduce four new pods and observe where they are scheduled after a new untainted node is added to the cluster.

###### **1️⃣** Yellow Pod

- **Toleration:** `color=yellow`
- **Placement Possibilities:** 
	- Untainted Node (✅First Preference)
	- Blue Node (🔵Only if the untainted node is full)

📌**Explanation:**
Since no node has a `color=yellow` taint, this pod is treated like a normal pod.

- **Untainted node** is the first preference because it has no tolerations.
- **Blue node** (`PreferNoSchedule`) **is the second choice**- the scheduler **will try to avoid it** unless the untainted node **does not have enough capacity**. 
-----
###### **2️⃣** Normal Pod (No Toleration)

- **Toleration:** None
- **Placement Possibilities:**
	- **Untainted Node** (✅First Preference)
	- **Blue Node** (🔵Only if the untainted node is full)

📌Explanation:

- Since this pod **has no toleration**, it cannot be scheduled on **Green** or **Purple** nodes.
- Untainted node is the **first choice** since it has **no restrictions**.
- **Blue node** (`PreferNoSchedule`) is the backup option if there is no space in the untainted node.

----
###### **3️⃣** Green Pod

- **Toleration:** `color=green`
- **Placement Possibilities:** 
	- Green Node (🟢First Preference)
	- Untainted Node (⚪Second Preference)
	- Blue Node (🔵Only if both green and untainted nodes are full)

📌**Explanation:** 

- **Green node is the first preference** because it **has a matching taint** (`color=green:NoSchedule`).
- **If the green node is full**, the scheduler places pods on the **untainted node**.
- In testing, when one replica was created, it was scheduled on the green node.
- When **ten replicas were created**, K8s distributed them across the **green** and **untainted** nodes on available **resources**.
- **Blue node** (`PreferNoSchedule`) is **only** used if both green and untainted nodes are full.
---
###### **4️⃣** Blue Pod

- **Toleration:** `color=blue`
- **Placement Possibilities:** 
	- Untainted Node (⚪First Preference)
	- Blue Node (🔵Only if untainted node is full)

📌**Explanation:**

- Since `PreferNoSchedule` is a soft restriction, the scheduler tries to avoid the blue node if there are better options.
- In testing:
	- **One replica** → Placed in the **untainted** **node** (scheduler prefer it over `PreferNoSchedule`).
	- **Ten replicas** → Pods were **distributed equally** between the **untainted and blue nodes**.
----
###### Why Should Pods with Tolerations Prefer Tainted Node?

We would want pods with tolerations to be scheduled onto nodes with matching taints rather than being placed on **untainted nodes**.

Imagine if:
- The **Green Node** is reserved for **Project A**, and
- The **Untainted Node** belongs to **Project B**.
Now, if a **Project A pod** is scheduled on the **Untainted Node** (Project B's node), it breaks the intended segregation.

###### Real-World Use Cases for This Segregation

This type of isolation can based on:

- **Project:** Different teams using dedicated nodes.
- **Departments:** Keeping workloads from Finance, HR, and IT separate.
- **Environments:** Ensuring **production workloads** do not mix with **development ones**.
- **Criticality:** Reserving high-performance nodes for mission-critical applications.

By combining **taints**, tolerations, **node affinity**, and **anti-affinity**, we can **enforce stricter placement policies** and ensure **workloads run on appropriate nodes** based on project requirements and business logic. 

----
###### Key Takeaways

- Taints are applied to nodes, Tolerations are applied to pods.
- A pod can only be scheduled on a tainted node if it has a matching toleration.
- **Three effects of taints:** `NoSchedule`, `PreferNoSchedule`, and `NoExecute`.
- **Operator** `Equal` (default) requires an exact match, while Exists ignores values.
- **Taints** and **tolerations** work together to **control pod placemen**t and **node access**.
---
###### Reference
[Understanding Taints and Tolerations in Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
