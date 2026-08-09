			Date					      21st July 26

*Table of Contents:
[[Kubernetes Architecture#Deployment|Deployments]]
K8s Architecture
	[[Kubernetes Architecture#Control Plane Components|Control Plane Components:]]
		 Cloud Controller Manager
		 Controller Manager
		 API Server
		 Schedular
		 ETCD
	[[Kubernetes Architecture#Data Plane Components|Data Plane Components:]]
		Kube-Proxy
		Kubelet
		Container Runtime*

-----------


Why would we run two containers inside a single Pod?

Let's imagine 
**Use Case 1:** **Logging** 
Container 1 is running your main application and container 2 is collecting the logs from the main application and dumping those logs onto your central logging server.

**Use Case 2:** **Monitoring**
In Container 1 our main application is running and in container 2 collecting the CPU, Memory and application specific statistics from the main application and sending it to your monitoring repository.

**Use Case 3: [[K8s Explanation#Proxy (Forward Proxy)|Proxy]]** 
Container 1 is running our main application and whenever a connection has to be made outside you use your proxy container that is container 2 to do the same.

**Use Case 4: [[K8s Explanation#Reverse Proxy|Reverse Proxy]]**
Our Container 2 is acting as reverse proxy, whenever someone from the outside accesses your webserver, i.e., they will hit the container 2 and container 2 will redirect the traffic to container 1.

*All of these patterns are called Sidecar patterns in K8s.

*Sidecar Pattern:* Means a helper container runs alongside the main application container in the same Pod to support it.
![[Pasted image 20260721104231.png]]

* **Pod Network Namespace:**
	Container share Networking with each other inside a Pod.
	*Example:* Container 1 can talk to Redis Container 2 in the same Pod using Localhost.
  
* **Share Storage/Data:**
	*Example:* If I assign a 5 GB volume to one of the Pods and map that volume to both of these containers then the content of that drive of that volume can be accessed by both of the containers.


# Deployment
![[Pasted image 20260721113821.png]]

***Scenario:***
**Without Deployment:**
If a Pod dies due to some reason. Then no one is there to restart the Pod. And Pod can't restart by itself. This behavior is similar to Docker. If a Docker container died then there was no orchestration tool to start that container.

**With Deployment:**
If a Pod dies, deployment object make sure that one Pod will always be running.

**Replica Management:**
![[Pasted image 20260721113839.png]]

*==Replica Management:==*
***Scenario:*** 
My user base has increased to 100 or more. I want to run 10 Pods. What I will do is I'll go this deployment and I'll change the number of replicas to 10. Now my Deployment object will ensure that 10 Pods are always running. This is called Replica Management.

==***Rolling Updates and Roll Backs:***==
Now 10 Pods are running if I want to update my application, I can update it without any downtime. It will first update a section of Pods then another section of Pods then another section Pods. So it will be a rolling update not all 10 pods will update at the same. 

==***Roll backs:***== If I want to rollback the change I can rollback in the similar fashion.

*==**Declarative Configuration:**==*
If I tell it once that I need 10 pods then it is not my headache to maintain 10 pods. Deployment Controller will take care of it.

Deployment Object vs. Deployment Controller:
Deployment is an Object that Object is managed by something called Deployment Controller.

ReplicaSet/ReplicaSet Controller vs. Deployment/Deployment Controller:

Deployment consistently monitors the desired state ex- if there are given pod are not running, then replica set controller will active and create or delete the pods, so relicaset is actually doing the task of creating and deleting the pods when deployment orchestration that is on the top of replicas set 


##### K8s Architecture
![[Pasted image 20260721113853.png]]

***==Control Plane:==***
- Is the brain of the cluster ( Should be High available).
- It orchestrates and manages the Data Plane/Worker nodes.

**==Data Plane:==**
- Data Plane is the one where applications or workloads run.

___
# Control Plane Components:

*==**Cloud Controller Manager:**==*
- When you have a cloud installation of K8s then you might want to leverage the services provided by the cloud provider.
**Example:** If I am on AWS I might want to use Application Load balancer or Identity Center or KMS for Key Generation.
So, Cloud Manager is the component that ensures I can speak to public cloud.

***==Controller Manager:==***
- It ensures that the desired state of the cluster is maintained. This process has multiple subprocess in K8s.
- Most of the objects have a corresponding controller. 
  
**Example:** Deployment controller, Replication Controller.
**Simple words:** Controller Manager ensures that cluster maintain the desired state.

*==**ETCD:**==*
- Is a key-value data store which saves cluster configuration data. (Should be Highly available).

*==**API server:**==*
- Is the Frontend of K8s cluster.
- Any tool or any user who want to interact with the cluster will come via the API server.

*==**Schedular:**==*
- Is responsible for scheduling pods onto worker nodes. 
- This scheduling is done based on resource availability and few other constrains.
  
**Example:** If I'm deploying a pod that requires a GPU (Graphics Processing Unit) so I would set a rule that this pod should land in let's say Worker node 2.
Schedular will check all these things and then schedule the Pod onto the required node.

___
# Data Plane Components:

![[Pasted image 20260721120958.png]]

**Scenario:**
- When Pod 1 wants to communicate with Pod 3 (Above Image). One way is I configure my application to communicate with this particular IP but If I do that we all know that all these are using the same image, we configure, to speak to Pod 3 then this Pod 3 goes down, will there be a point of having such high available environment. There is no point to load balance among these three pods.
So, we can introduce something called K8s Service.

==***K8s Service:***==
- A Kubernetes Service provides a stable way to access one or more Pods, even if the Pods change or restart
- Is a Logical construct which is available for all the worker nodes, Pods.
  
**Example:** 
- Now, I can configure whenever my web service wants to speak to Redis backend then the request must be sent to the Redis Service not to individual Pods. 
- Python Frontend Pod 1 wants to speak with Redis Backend Pod 3 then once Pod 3 receives a request then Pod 3 will directly send the response to Pod 1 then service will not be involved.

Why are we discussing this? In this context,

***==Kube-proxy:==***
- Is a traffic director in K8s. 
- Takes care of service to Pod routing.
  
**Example:** if the traffic is coming to Python Frontend Pod 1 and it is being sent to Pod 1 or Pod 3 or Pod 2 that routing table is maintained by Kube-proxy.

- ***Load distribution:*** Which Pod should receive this traffic, taken care by the Kube-proxy.
- ***Health Checks:*** Kube-proxy ensures that the traffic is sent to the Pod that is healthy. 

-- 1. Service to Pod routing rules
-- 2. Load distribution 
-- 3. Health Checks.

**Note:** **We mentioned that Kube-proxy is optional.**

 ***==CNI (Container Network Interface) Plugin:==*** 
a CNI plugin is the basic networking layer that allows Pods to communicate with each other in K8s cluster.

Perform 5 main Functions.
1. Assigning IP addresses to Pods.
2. Assigning network interfaces to Pods.
3. IP address management.
   ***Example:*** Whenever a Pod is removed, it keeps an inventory of that IP, is now free to use. Whenever a new Pod is created it ensures that it assigns a unique IP to that Pod.
4. Network Policy Enforcement.
   ***Example:*** If I have 2 applications running in the cluster then I don't want application 1 to speak with application 2 Pod. That can be done by Network Policies.
5. Pod-to-Pod Networking.
   
*Example:* How do I ensure that Pod 1 can reach Pod 3 irrespective of the node. So, it ensures all Pods in the K8s cluster can speak with each other. 

*Note:*
- As we mentioned something similar that Kube-proxy, we need to understand that CNI plug-in is the foundational networking component of K8s cluster.
- While CNI ensures all Pods can speak with each other, on the other hand, Kube-proxy tells they will do so by defining the routing rules. [[K8s Explanation#CNI Plugin's job|*Kube-proxy vs. CNI plugin* ]]

*==**Kubelet**:==* 
- Monitors health of the Nodes and Pods.
- Instructs the container runtime to pull the images, are required to start a Pod.
- It works with the CNI plugin to setup the networking for containers.
  
*Example:* If I ask container runtime to pull the NGINX image from the repository and run a container. The main function of container-runtime is container lifecycle management.

___

![[Pasted image 20260723154443.png]]

####                                             ==Clients==

**kubectl:**
- Is a client to interact with API server.

**CICD Tools:**
- There are CICD tools, these are build tools or deployment tools that you integrate with your cluster for application deployment or to make amendment to K8s cluster.

**3rd Party Tools:**
- These tools could be monitoring logging solution or anything that you want to integrate with K8s cluster.

_____
kubectl apply -f deployment.yaml

- kubectl --> K8s command-line tool. It will verify the syntax of the YAML. It will see if any indentation errors. Once it's done. It will send it to API server
- apply --> Creates the resource if it doesn't exit, or updates if it already exits.
- -f --> Stand for file. It tells K8s to read the configuration from a file.
- deployment.yaml --> The YAML file containing the K8s resource definition.

Note: **API server will first authenticate and authorize this request.**

*Example:* 
- It will see whether he/she as user is authorized to create a deployment then it will verify the full k8s schema in the deployment.
- Once it's done these things will be stored in the ETCD storage.




 ***==Desired state==***

The Desired State is state that we define for our K8s resources, and K8s continuously work to ensure actual state matches that desired state. 

*Example: I told deployment to maintain 5 replicas/Pods but the cluster right now doesn't have 5 replicas, Controller Manager gets into the action. It watches the API server to see if it has any work, for any controller or in this case deployment controller, sees that there is a deployment which doesn't have the desired number of replicas or pods running, so the deployment controller will tell the API server to create a replica set.
The replica set object is created by the API server and stored in the ETCD. 

*Schedular also watches the API server, sees that there are five pod objects which do not have any nodes assigned to it, so let met get to work and see what are the Pod specifications so that I know, in which node I should not or I should place these pods. 

*Then, schedular updates this information to the API server that these are the node names that I have. API server updates the ETCD.*



==**Kubelet:**== 

Now, ETCD has updated information which pods are supposed to be run onto which node. 

Kubelet gets into action, Kubelet is also watching the API server and API server sends the information about the Pod objects to kubelet.

Kubelet sees this request and I'm supposed to run these pods then let me ask the container runtime to run these containers.

Then container runtime will pull the image, second it will configure networking with help of CNI plugin, third it will mount the required volumes onto those containers, last it will create and start the containers.

Kubelet consistently monitors the health check of the containers and this health status is shared with the ETCD data store.

----

==*Metric Servers:*==
As  the name suggests collects the metrics of nodes and pods from the kubelet directly, and Metric server is essential when you want to use features like horizontal pod Auto scaling.

==*CoreDNS:*== 
- Is the DNS service in K8s cluster. 
- It will translate the DNS names into IP address.

*==K8s DashBoard:==*
- Kubernetes Dashboard is a web-based GUI that lets you view and manage Kubernetes resources without using `kubectl` commands.

As we discussed, kubelet consistently monitors the health of the container and kubelet sends this information to the API server and K8s dashboard uses this information to display the health of the cluster. 
____

*My Answer for the Interview:
The Kubernetes Architecture is the bunch of components to keep the cluster in the desired state.
We have Control Plane and Data Plane
Control Plane is the brain of the K8s cluster, Date plane is the one where application run.

Control Plane, we have
Cloud Controller Manager, lets K8s communicate with Cloud providers like AWS or Azure.
API server, receives request.
ETCD, stores the cluster data and state.
Controller Manager, checks whether the actual state matches the desired state.
Schedular, picks the best Nodes for each Pod.
Kubelet, on the each node actually run the pods using the container runtime.
Kube-Proxy, is a traffic director, takes care of service-to-pod routing.


