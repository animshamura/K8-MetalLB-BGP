<div align="justify">
    
**What is BGP ?** 

BGP, or Border Gateway Protocol, is a routing protocol used to share network layer reachability information (NLRI) between different routing domains, often referred to as Autonomous Systems (ASs). Each AS is typically managed by a separate administrative organization. The current structure of the Internet is a vast network composed of numerous interconnected ASs. As an external routing protocol, BGP is extensively utilized among Internet Service Providers (ISPs). It facilitates the exchange of reachable routing information between ASs, helps establish paths across ASs, prevents routing loops, and implements routing policies between ASs. The protocol has evolved through three earlier versions: BGP-1, BGP-2, and BGP-3, with the most commonly used version today being BGP-4.

There are two types of BGP : iBGP and eBGP. 

**Internal BGP (iBGP)**:
- Imagine your internet service provider (ISP) as a big network. Within that network, there are lots of routers talking to each other to figure out the best paths for your data to travel.
- iBGP is like the secret language these routers use to chat with each other. It helps them share information about the best routes to take within the same ISP.
- In a big ISP, there could be tons of routers, and they all need to be on the same page. iBGP makes sure they're all talking to each other properly.
- Also, if one router knows a better path, it can tell the others without changing anything about the message.

**External BGP (eBGP)**:
- Now, let's zoom out a bit. Imagine your ISP as one country, and there are other countries (other ISPs) out there.
- eBGP is like the language your country uses to talk to other countries about the best routes for data to travel between them.
- ISPs need to share information about the best routes to reach each other's networks. eBGP helps them do that.
- It's like they're neighbors sharing tips on the best roads to take to reach each other's houses.
- But to have this chat, their routers need to be directly connected or have special directions (static routes) to find each other.

So, iBGP helps routers within the same ISP talk to each other, while eBGP helps different ISPs communicate to find the best paths for your data to travel across the internet. Both of these protocols are really important for making sure your internet connection runs smoothly!

**Why do we use BGP ?**

Interior Gateway Protocols (IGPs) focus on providing routing information within a single network, while BGP was developed to handle routing between different networks, or domains. IGPs, such as RIP, OSPF, and IS-IS, optimize paths within a network but lack the extensive policy control needed for inter-domain routing. BGP, on the other hand, is designed to manage policies and scale for inter-domain routing, serving a different purpose than IGPs.

![BGP & IGP](https://github.com/animshamura/Documentation-Images/blob/main/BGPI.drawio.png?raw=true)    
**BGP Working Procedure**

BGP utilizes TCP for establishing peer relationships, requiring peers to first establish a TCP connection and negotiate parameters through Open messages. Once the peer relationship is established, BGP peers exchange routing tables, with Keepalive messages being sent to maintain connections. BGP updates routing tables incrementally via Update messages upon route changes, and in the event of errors, Notification messages are sent to report the error and terminate the BGP connection.

 **Metal-LB**

MetalLB is a tool for Kubernetes clusters without cloud provider load balancer support. It offers two main features: address allocation and external announcement.

1.  Address Allocation:
    
    -   In cloud environments, a load balancer assigns IP addresses, but MetalLB handles this for bare-metal clusters.
    -   Users provide IP address pools to MetalLB, either leased from a hosting provider or chosen from private address spaces like RFC1918.
    -   MetalLB can manage multiple address pools.
2.  External Announcement:
    
    -   MetalLB ensures that external networks recognize the IP addresses it assigns to services.
    -   In Layer 2 mode, a machine in the cluster announces the IPs using ARP or NDP protocols.
    -   In BGP mode, cluster machines establish peering sessions with nearby routers, enabling load balancing and traffic control.     
        ![MetalLB.drawio.png](https://github.com/animshamura/Documentation-Images/blob/main/MetalLB.drawio.png?raw=true)
        
**Metal-LB Working Procedure**

In MetalLB, the speaker and controller components work together to manage the load balancing of services within a Kubernetes cluster. Here's a detailed breakdown of their roles and how they interact:

**Roles & Responsibilities**

**Controller :**

The controller is responsible for:

   - IP Assignment: Assigning IP addresses from a predefined pool to Kubernetes services of type LoadBalancer.
   - Monitoring Services: Watching for changes in Kubernetes services and nodes, and updating the configuration accordingly.
   - Configuration Management: Managing the configuration specified in Kubernetes ConfigMaps, such as IP address pools and BGP peers.

**Speaker :**

The speaker is responsible for:

   - Announcing IPs: Ensuring that the assigned IP addresses are announced to the network.
        In Layer 2 mode, it uses ARP (Address Resolution Protocol) to announce IPs.
        In BGP mode, it uses BGP (Border Gateway Protocol) to advertise routes to external routers.
   - Handling Traffic: Ensuring that traffic directed to the assigned IPs is routed to the correct node.
   - Failover Management: Reassigning IPs and re-announcing them if a node fails.

![image](https://github.com/animshamura/Documentation-Images/blob/main/MetalLB-Demo.drawio.png?raw=true)

### Installation and Configuration of MetalLB with BGP

#### Prerequisites:

-   A Kubernetes cluster (v1.13 or later)
-   Administrative access to the Kubernetes cluster
-   BGP router(s) in the network

### Step 1 : Install MetalLB
#### Enable Strict ARP 

#### Apply the MetalLB Manifest

First, apply the MetalLB manifest to your cluster. This installs MetalLB’s components.

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.3/manifests/namespace.yaml

### Step 2 : Install and Configure VyOS 
  You need a separate VM to operate VyOS. Therefore, you have to download the night build image of VyOS from their official website. For installing VyOS in a VM, you must choose any of Debian realises of the linux distributions. Then, configure VyOS form the VyOs CLI. You have to login with default user and password which is vyos in both cases. 

    configure
    set interfaces ethernet eth1 dhcp 
    set interfaces ethernet eth0 address <router-ip> 
    set protocols bgp system-as <local-ASN> 
    set protocols bgp neighbor <neighbor-ip> remote-as <neighbor-asn>
    commit
    save
    exit

### Step 3 : Configure MetalLB
Configure MetalLB using the configmap.yaml. 

    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: metallb-system
      name: config
    data:
      config: |
        peers:
        - peer-address: <vyos-router-ip>
          peer-asn: <vyos-asn>
        address-pools:
        - name: default
          protocol: bgp
          addresses:
          - <start-ip>-<end-ip>

Create configmap applying kubectl command. 

    kubect apply -f configmap.yaml

### Step 4 : Check BGP peering 
Use the command below to check whether BGP peering has been established in VyOS CLI/

    show ip route
    show bgp neighbor

 ### Step 5 : Test MetalLB Loadbalancing 
Deploy a Kubernetes service of type LoadBalancer and ensure that MetalLB assigns an external IP address from the configured pool. Test the load balancing functionality to verify that traffic is being properly distributed to the service endpoints.
  
        ---  
        apiVersion: apps/v1  
        kind: Deployment  
        metadata:  
        name: nginx-deployment  
        spec:  
        replicas: 3  
        selector:  
        matchLabels:  
        app: nginx  
        template:  
        metadata:  
        labels:  
        app: nginx  
        spec:  
        containers:  
        - name: nginx  
        image: nginx:latest  
        ports:  
        - containerPort: 80  
        ---  
        apiVersion: v1  
        kind: Service  
        metadata:  
        name: nginx-service  
        spec:  
        selector:  
        app: nginx  
        ports:  
        - protocol: TCP  
        port: 80  
        targetPort: 80  
        type: LoadBalancer
</div>
