# Networking

Kubernetes uses three different networks:

- Infrastructure: The physical or virtual backbone connecting the machines hosting the nodes. The infrastructure network enables connectivity between the nodes; this is essential for the Kubernetes control plane components (like the kube-apiserver, etcd, scheduler, and controller-manager) and the worker nodes to communicate with each other. Although pods communicate with each other via the pod network (overlay network), the underlying infrastructure network supports this by facilitating the physical or virtual network paths between nodes.
- Service: This is a purely virtual and internal network. It allows services to communicate with each other and with Pods seamlessly. This network layer abstracts the actual network details from the services, providing a consistent and simplified interface for inter-service communication. When a Service is created, it is automatically assigned a unique IP address from the service network's address space. This IP address is stable for the lifetime of the Service, even if the Pods that make up the Service change. This stable IP address makes it easier to configure DNS or other service discovery mechanisms.
- Pod: This is a crucial component that allows for seamless communication between pods across the cluster, regardless of which node they are running on. This networking model is designed to ensure that each pod gets its own unique IP address, making it appear as though each pod is on a flat network where every pod can communicate with every other pod directly without NAT.

My infrastructure network is already up and running at `10.4.0.0/20`. I'll configure my service network at `172.16.0.0/20` and my pod network at `192.168.0.0/16`.

With this decided, we can move forward.
