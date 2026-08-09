##### *Table of Contents:*
**ReplicationController (rc)**
- What is ReplicationController?
- Labels and Selectors

**ReplicaSet (rs)**
 - What is ReplicaSet?
 - Equality-Based vs Set-Based Selectors
 - Equality-Based Selector Example
 - Set-Based Selector Example

**Deployments**
- What is a Deployment?
- How Deployments Build on rs?
- Explaining Rolling Updates and Rollbacks in Deployment?

#### ReplicationController (rc)

![[Pasted image 20260728163101.png]]

**What is a ReplicationController?**
-  Ensures a specified number of pod replicas are running at any given time.
- Any pod with matching labels becomes part of the rc, regardless of how it was created.
#### Labels and Selectors

Equality-Based vs Set-Based Selectors

| Selector type  | Supported By       | Description                                                               |
| -------------- | ------------------ | ------------------------------------------------------------------------- |
| Equality-Based | rc, rs, Deployment | Matches pods where the label key equals a specific value.<br>(app: nginx) |
| Set-Based      | rs, Deployment     | Matches pods based on conditions (In, NotIn, Exists, DoesNotExist).       |
**Set-based** selectors provides a flexible and powerful way to identify and group K8s objects (like Pods) based on their labels, allowing for fine-grained control and management.

----
#### ReplicaSet (rs)

![[Pasted image 20260728163158.png]]

##### What is a ReplicaSet?
- RelicaSet is an improved verion of ReplicationController.
- It supports set-based selectors, providing more flexibility in managing pods.
- ReplicaSets are often managed by Deployments, which add advanced capabilities.
##### Set-based Selector Example

| Operator     | Description                                                                     | Example                                                         |
| ------------ | ------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| In           | Matches if the value of the specified key is in the provided set of values.     | Pods where the label **app** is either **nginx** or **apache**. |
| NotIn        | Matches if the value of the specified key is not in the provided set of values. | Pods where the label **environment** is not **development**.    |
| Exists       | Matches if the specified key is present, regardless of its value.               | Pods where the **tier** label is defined.                       |
| DoesNotExist | Matches if the specified key is not present.                                    | Pods where the **dubug** label is **not** defined               |
Explanation:
1. `.selector.matchExpressions`: Defines the rules for selecting pods.
    - Includes pods where:
	- app is nginx or Apache.
    - environment is not development.
	- tier label exits
	- debug label does not exit.
 2. `.template.metadata.labels`: 
	- Assigns labels to the pods created by the ReplicaSet. These labels must satisfy the rules in `.selector.matchExpressions`.
3. Pods Created
	- Pods created by this ReplicaSet will:
		- Have labels **app:nginx**, **environment: production**, and **tier: frontend**.
		- Not include the **debug** label.
4. Replicas:
	- The ReplicaSet will maintain 3 replicas of the specified pods.
----
#### Deployments

**What is a Deployment?**
- Deployment build on top of ReplicaSets.
- They provide advanced features such as:
     - Rolling updates and rollbacks.
     - Automated updates to pods.
     - Simplified scaling and management.
#### How Deployments Build on ReplicaSet?
- Deployments manage ReplicaSet and create a new version whenever updated.
- This enable rolling updates without donwtime.

##### What are Annotations in K8s?
Annotations in Kubernetes are key-value pairs used to store non-identifying metadata about Kubernetes objects to help tools, scripts, or humans understand the purpose, context, or other details of the object. Unlike labels, they are not used for selecting or grouping resources.

###### Key Characteristics of Annotations:
- They do not affect the behavior of the Kubernetes object directly.
- They are primarily used for informational or operational metadata.
- They can hold small amounts of arbitrary data, such as:
     - Deployment changes
     - Build information
     - Links to external resources
     - Operational notes or logs
##### Good to Know
Every time you update a deployment (e.g., using kubectl set to update the image), Kubernetes creates a new RepllicaSet in the background. The previous ReplicaSet's replica count is scaled down to `0`, and the new ReplicaSet is scaled up to the desired number of replicas.

When you perform a rollback, Kubernetes scales down the current ReplicaSet to `0` and scales up the ReplicaSet associated with the revision you're rolling back to 0 and scale up the ReplicaSet associated with the revision you're rolling back to, effectively switching the active ReplicaSet to the previous version.

##### Key Pointers:
1. Always annotate deployments with kubernetes.io/change-cause to track changes for easier management and debugging.
2. Use `kubectl rollout history` to view revision details.
3. Rollbacks can be performed to any revision using the 
   `--to-revision` flag.
   
##### Important Kubernetes Deployment Commands
- **Create a deployment Imperatively:**
   `kubectl create deployment nginx-deployment --image=nginx:1.19`

- List all deployments:
   `kubectl get deployments`

- Describe a deployment image:
  `kubectl describe deployment nginx-deployment`

- Update deployment image:
  `kubectl set image deployment nginx-deployment nginx-container=nginx:1.20`

- Scale deployment replicas:
  `kubectl scale deployment nginx-deployment --replicas=5`

- View rollout history:
  `kubectl rollout history deployment nginx-deployment`

- Annotate a deployment:
  `kubectl annotate deployment nginx-deployment` 
  `kubernetes.io/change-casue="Upgraded nginx 1.19 to 1.20"`
  
  - Rollback to a previous revision:
   `kubectl rollout undo deployment nginx-deployment --to-revision=1`
   
   - Edit a deployment:
   `kubectl edit deployment nginx-deployment`
   
   - Get deployment manifest in YAML format:
    `kubectl get deployment nginx-deployment -o yaml`
   
   - Delete a deployment:
    `kubectl delete deployment nginx-deployment`  
    
###### Why Pause and then Resume a Deployment?
 Pausing a deployment is useful for temporarily halting the rollout process.
###### Key reasons include:
- **Making multiple changes:** If you need to adjust various aspects of your application, pausing lets you queue changes and apply them all at once upon resuming, avoiding multiple partial rollouts.
- **Investigating issues:** If a problem arises during the rollout, pausing allows time to investigate and fix the issue before proceeding.

Resuming the deployment continues the rollout from where is was paused, ensuring all pending changes are applied.

##### Key Differences Between rc, rs and Deployments

| Feature      | RelicationController<br>           (rc) | ReplicaSet<br>     (rs)      | Deployment                    |
| ------------ | --------------------------------------- | ---------------------------- | ----------------------------- |
| **Selector** | Equality-based only                     | Equality-based and Set-based | Equality-based and Set-based  |
| **Updates**  | Manual                                  | Manual                       | Automated (rolling updates)   |
| **Scaling**  | Manual or Imperative                    | Manual or Imperative         | Declarative and automated     |
| **Use case** | Legacy workloads                        | Modern workloads             | Advances workloads with CI/CD |
