## **Container Network**:
When we create a container using Docker (or any other container runtime like CRI-O), it automatically generates a separate Linux network namespace for that container. This means that the container gets its own network stack, including its own IP address, routing tables, and ports, which are isolated from the host machine's network.
The above statement rises a question **Why should we have separate Network Namespaces for each container?**
To answer this lets look at this scenario:
#### Connecting Containers to the Host Network:
If we want to allow a container to utilize the host network we can attach `--net=host` to the docker run like this:
```bash
docker run -d --net=host --name nginx-host nginx
```
This command runs the NGINX container and binds it directly to the host's network. Instead of having its own isolated network namespace, the container will use the host’s IP address and network interfaces.****

Advantage of using `--net=host` :
- This will enable the container to directly listens to port 80 (or whatever port needed) of the host. this will eliminate the use of Docker port publishing
- Directly using the host network stack will provide better network performance for the container.

Disadvantages of Using `--net=host`:
- Have multiple container on the same node will introduce problems with shared network resources like port conflicts etc.
- It will not isolate the network of the container from the Host network which makes the container more vulnerable if the host network is attacked or compromised.

In the above scenario, using `--net=host` should be approached cautiously, especially for production environments where isolation and security are critical.
To avoid these problems, it’s recommended to keep containers in their own **network namespaces**, which include separate IP addresses and port allocations. This brings several benefits:
- **Network Isolation**: Each container can run independently with its own network stack, avoiding conflicts and maintaining security boundaries. This isolation also ensures that processes in one container cannot directly interfere with or spy on network traffic in another container or the host
- **Flexibility**: Containers with their own network namespaces can be flexibly attached to different networks or virtual networks, such as bridges or overlay networks, which can be used to provide dynamic networking configurations 
- **Resource Management**: Having containers in separate network namespaces makes it easier to manage network resources. Different CNI plugins in kubernetes (e.g., Calico, Cilium) uses custom network policies to enforce fine-grained networking policies, allowing for better control over container communication.
- **Host Network Protection**: By isolating the container’s network stack from the host’s, it reduces the risk of accidental or malicious misconfiguration impacting the host’s networking.
## **Communication between Pods :**
In this section, we will explore how **Kubernetes pods** communicate with each other, whether they are on the same node, different nodes, or even across clusters. But before we dive into pod communication, it is essential to understand how pods within nodes communicate between each other.
### Container and Network Namespaces
- Every Pods, by design, runs within its own **network namespace** which provides an isolated network stack, similar to a host with an independent "network stack" containing a unique IP address, routing tables, and ports. The question is: how do containers in separate network namespaces communicate with each other?
### Communicating within same host:
- When we are deploy or run pods (in Docker or Kubernetes), they may need to talk to each other, right? But how does that work?
- Well, every time a new container or pod is created, Docker or any other Container Runtime Interface (CRI) establishes a **separate network namespace** for it which ensures network isolation. Essentially, a network namespace provides a container with the illusion of being a separate machine, allowing it to communicate independently.
- Since the pods are now  in separate Network Namespaces which isolated pod network the host, the **Container Network Interface (CNI)** or Docker sets up **Ethernet devices** within the pods to handle communication. These Ethernet devices are part of what we call **Veth Pairs**.
- **Veth Pairs (Virtual Ethernet Devices)**: pods are connected to the bridge using **veth pairs** (virtual ethernet pairs). 
	- When a pod is launched or created, Docker or the CNI creates two virtual Ethernet devices (called veth pairs). One side of the veth device is placed inside the container’s network namespace, while the other end connects to the host’s network namespace or a virtual bridge.
	- Any data packet sent from one side of the veth device is immediately transferred to the other side. This is how containers communicate over the bridge.

>[!info]
>Here Veth means virtual ethernet
- **Virtual Bridge**: When Docker is installed, or a host joins a Kubernetes cluster, a virtual bridge (called `docker0` in Docker) is created on a host, which acts like a **virtual switch**. Every pod is connected to this bridge, and through it, they can communicate with one another.
	- The pods are assigned unique IP addresses by the bridge for communication.
	- We can use this command `brctl show` to show the bridges.
	- Additionally, to check the virtual network interfaces on a node, we can use `ifconfig` or `ip address`.
- To list all the namespaces created in a node we can use:
```bash
# For all
lsns
# only for network namespace TYPE=net
lsns -t net
```
https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/
#### Pod Communication in Kubernetes:
While Docker uses `docker0`, Kubernetes abstracts container communication using its own networking model. **Kubelet**, the Kubernetes node agent, creates a network namespace for each pod and attaches the pod to the network through the **Container Network Interface (CNI)** plugins.
- Each Pod in a Kubernetes cluster has its own independent IP address whereas in docker.
-  Multiple Pods on same host can directly communicate with each other using localhost or Pod IP addresses because their Network Namespace is connect to same bridge.
- `cbr0`(L2 network device) is own kubernetes bridge which is created to differentiate from `docker0` bridge used by docker.

>Tl;DR
>Each pod runs in its own **network namespace** . On the host, A virtual bridge (e.g., `docker0`) is established, that allowing communication between containers attached to this bridge. **Veth pairs**  link each container to the bridge, facilitating packet transmission between containers via this virtual switch.
![[Pasted image 20241023153426.png|600]]
### Communicating across different  host:
So far, we have discussed how pods communicate within the same host using bridges like `docker0`. However, in a Kubernetes cluster, pods are distributed across multiple nodes, and they still need to communicate seamlessly. Let's understand how Kubernetes achieves this cross-node communication.
- On a single node, communication is relatively simple due to the presence of a **virtual bridge** (`docker0`). However, **`docker0` bridges on different nodes are isolated** and do not automatically have connections between them.
- **Overlay Network: A Solution for Cross-Node Communication**:
	- An overlay network provides a virtual network that connects containers across multiple nodes. Essentially, it creates a **"public" bridge** using software, which spans across all the nodes in the cluster.
	- Through this network, containers or pods can communicate as if they were on the same physical or virtual LAN, even if they are located on different nodes.
	- Basically Overlay Network is a flat network working above host network.
	- Kubernetes does not provide any default network implementation, rather it only defines the model and leaves to other tools like kube-proxy, Calico, Flannel to implement it

>[!note]
> All plug-ins must follow a couple of important rules. 
> First, Pod IP addresses should come from a single pool of IP addresses, although this pool can be subdivided by node. This means that we can treat all Pods as part of a **single flat network**, no matter where the Pods run.
> Second, traffic should be **routable** such that all Pods can see all other Pods and the control plane.

- **How CNI Work**:
	- Overlay networks in Kubernetes are primarily managed by specific CNI (Container Network Interface) plugins. CNIs are responsible for two crucial tasks:
		-  **IP Management**: CNIs allocate and manage each Pod's IP address, ensuring that there are no conflicts within the network.
		-  **Data Synchronization**: CNIs synchronize network information across all nodes to maintain consistent routing and communication between nodes.
		
	To understand CNI lets take a look on how flannel and calico works 
	- Both **Flannel** and **Calico** operate at Layer 3 of the OSI model, focusing on IP-based routing for communication between nodes:
	- **flannel** : Flannel is created by CoreOS for Kubernetes networking, It support in 3 different models:
		#### Mode 1: UDP
		- When Flannel is set up on a Kubernetes node, it runs a daemon named **flanneld**. This daemon creates the TUN device `flannel0`, which acts as the default gateway for the Docker bridge (`docker0`).
		- Flannel allocates a unique subnet from the Pod CIDR to each node, ensuring that every Pod on that node receives an IP within this subnet.
		- To view the private network segment allocated to Docker, you can check `/etc/docker/daemon.json` on the node. Flannel can also modify Docker's configuration to attach the allocated network segment by using the `--bip` option.
		- Suppose we have pod0 on node1 and pod1 on node2 then the packets between then would follow this path:
			- **Pod0** → `docker0` → `flannel0` (flanneld packs the packet) → (sender network interface) → (intermediate router) → (receiver network interface) → `flannel0` (flanneld unpacks) → `docker0` → **Pod1**
		- As we can see **The process is very complicated and long.** And as flanneld runs in userspace there is a performance cost due to context switching and packet handling.
		#### Mode 2: VxLan
		- VXLAN, or Virtual Extensible LAN is a network virtualization technology supported by Linux. 
		- VXLAN can completely implement encapsulation and decapsulation in kernel mode, thereby building an overlay network through the "tunnel" mechanism.
		- In this mode the VxLan will set up VTEP() in layer 2 network.****



Overlay networks are built using tunneling techniques such as **VXLAN (Virtual Extensible LAN)**, **GRE (Generic Routing Encapsulation)**, or other protocols. These tunnels allow data packets to be encapsulated and transported between nodes in the cluster.

%% > [!info]
 > The above is explained using Docker but this can be extended to other container Runtimes. %%
---
### Outside clusters :
#### DNS:
- K8s use **CoreDNS** for dns resolution. Previously, `KubeDNS` was used for the same
- The **default domain name** used for DNS resolution **within the cluster** is `cluster.local`
- DNS naming conventions:
	- For services: `<service-name>.<namespace>.svc.cluster.local`.
	- For pods: `<pod-ip-address-replace-dot-with-hyphen>.<namespace>.pod.cluster.local`.
- When Pods and Services are in the same namespace, you can **use just the service name** instead of the FQDN , e.g., `http://<serice_name>
- For Pods and Services in different namespaces we can use `<service-name>.<namespace>` and optionally append `.svc` or `.svc.cluster.local`.
- Extra info: The configuration file used to store DNS settings is called the **Corefile**.
- https://blog.sophaskins.net/blog/misadventures-with-kube-dns/


### Service in k8s:
#### Kube-Proxy: The Core of Kubernetes Networking: 
**Kube-proxy** runs on every node in a Kubernetes cluster. It watches for changes in **Services** and **Endpoints** objects in real-time and dynamically adjusts network rules accordingly here is how it works:
- When a user creates a **Service** in the cluster, an **Endpoints** object with the same name is automatically created to store the IP addresses of Pods that match the Service’s label selector.
- Kube-proxy monitors these changes and sets up the necessary network rules (using iptables or IPVS) on each node. These rules allow clients to connect to Services through a stable IP, known as the **ClusterIP**.
- Once kube-proxy has set the required rules, client Pods or external users can access the Service by routing traffic through these rules. Kube-proxy then forwards the request to the relevant Pods that belong to the Service, based on load-balancing algorithms.

>[!note]
>When a Service is created, kube-proxy creates a **NAT link** in iptables to facilitate routing. If two different deployments share the same Service label, kube-proxy will balance traffic across both deployments' Pods.
---
- Whenever we create a service kube-proxy crete a nat link in iptables.
- If two pods have the same label but are part of different deployments, the same service will load-balance traffic to both deployments.
- **Types of Services**:
	- **ClusterIP**: 
	    - Default type.
	    - Used for internal communication within the cluster.
	    - Only accessible inside the cluster.
	    - spec.type: ClusterIp
	  - **NodePort**:
	    - Exposes a port on each node in the cluster.
	    - When a packet hits the node on the specified port, it is forwarded to the corresponding ClusterIP service, which then load-balances to the backend pods.
	    - When a NodePort service is created, Kubernetes also creates a ClusterIP service for that NodePort.
	    - 3 reasons to not use Nodeport:
		    -  **Port Exposure on All Nodes**: By exposing a NodePort, the service becomes accessible on the same port across all nodes in the cluster. This increases the attack surface by opening external access on every node, which can be a security risk.
		    - **Traffic Routing and Latency**: In a scenario where traffic is directed to a node that doesn’t have the desired Pod running, Kubernetes will reroute the traffic to the appropriate node hosting the Pod. This rerouting process introduces _an extra hop_, adding network latency to each request.
		    - **Limited Port Range**: NodePort services are restricted to a predefined range of ports (default: 30000–32767). This range limitation can become a constraint when you have multiple services requiring external exposure.
		- spec.type: NodePort
	  - **LoadBalancer**:
	    - Exposes the service to the external world using a cloud provider's load balancer.
	    - Useful for accessing applications from outside the cluster.
	- **Ingress**: 
		- Ingress is a Kubernetes resource that provides external access to services within a cluster
		- Types of Routing:
			- Basic Routing
			- Host-Based Routing
			- Path-Based Routing
			- Wildcard Routing
	- **ExternalName**:
		- we can point to other cname
		- Services with type `ExternalName` work as other regular services, but when the service name is accessed, instead of returning cluster-ip of this service, it returns CNAME record with value to the external service.
- **Services Without Selectors**:  It’s possible to create a Service without specifying a Pod selector, which can be useful in scenarios where you want to manually define the backend. This involves creating a separate **Endpoints** object that lists the IPs of the backend servers:
	```yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: dummy-svc
	  labels:
	    app: nginx
	spec:
	 ports:
	    - protocol: TCP
	      port: 80
	      targetPort: 80
	      name: http
	---
	apiVersion: v1
	kind: Endpoints
	metadata:
	  name: dummy-svc 
	subsets: 
	  - addresses:
	    - ip: 100.96.8.53
	    - ip: 100.96.6.43
	    - ip: 100.96.8.54
	    ports:
	    - port: 80
	      name: http
	```
- **Headless service** :
	- A headless service is a special type of service without a cluster IP, created by setting `clusterIP: None`.
	- Useful when you want clients to connect directly to individual pods, bypassing load-balancing.
    - Useful for stateful applications or when backend pods need to communicate with one another.
	- The Dns Instead of returning a single DNS A record, the DNS server will return multiple A records for the service, each pointing to the IP of an individual pod backing the service at that moment.
	- There’s no built-in load-balancing or proxying with a headless Service. Each Pod’s IP address is exposed directly, giving more flexibility and control over traffic distribution.
---
#### NetworkPolicy:
- From k8s doc 
	- `If you want to control traffic flow at the IP address or port level (OSI layer 3 or 4), NetworkPolicies allow you to specify rules for traffic flow within your cluster, and also between Pods and the outside world. Your cluster must use a network plugin that supports NetworkPolicy enforcement.`
	
- A `NetworkPolicy` is a Kubernetes object that enables the creation of policies to **control communication between pods and external entities** within a namespace.
- The `ingress` section defines **rules for incoming traffic**, while the `egress` section defines **rules for outgoing traffic**.
- `NetworkPolicy` uses:
	- `podSelector` to **select pods based on their labels**.
	- `namespaceSelector` to **select pods in specific namespaces**.
### 
###
###

Notes
- To get open port use /proc/net/tcp -> here the output is in hexadecimal


![[Pasted image 20241023133609.png]]