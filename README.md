# Amazon EKS Networking Bootcamp

This repository serves as a guide for the Amazon EKS Networking Bootcamp. Throughout this bootcamp, which focuses on EKS networking, you'll delve into different networking solutions and tackle IP exhaustion concerns. Additionally, you'll gain proficiency in crafting and enforcing network policies at the workload level, safeguarding your applications with enhanced security measures.

## Networking and Policy options for Amazon EKS with Calico

Amazon EKS runs upstream Kubernetes, so you can install alternate compatible CNI plugins to Amazon EC2 nodes in your cluster.

> [!IMPORTANT]
> If you have Fargate nodes in your cluster, the Amazon VPC CNI plugin for Kubernetes is already on your Fargate nodes. It's the only CNI plugin you can use with Fargate nodes. An attempt to install an alternate CNI plugin on Fargate nodes fails.

Amazon EKS maintains relationships with a network of partners that offer support for alternate compatible CNI plugins. For this workshop we will focus on Calico options for EKS.

### Amazon VPC networking with Calico Policy Enforcement

The geeky details of what you get:

| Policy | IPAM  |  CNI  | Overlay |  Routing   | Datastore  |
| :----: | :---: | :---: | :-----: | :--------: | :--------: |
| Calico |  AWS  |  AWS  |   No    | VPC Native | Kubernetes |

Calico network policy enforcement on an EKS cluster using the AWS VPC CNI plugin.

### EKS with Calico networking

The geeky details of what you get:

| Policy |  IPAM  |  CNI   | Overlay | Routing | Datastore  |
| :----: | :----: | :----: | :-----: | :-----: | :--------: |
| Calico | Calico | Calico |  VXLAN  | Calico  | Kubernetes |

Deploying Calico as the Container Network Interface (CNI), IP Address Management (IPAM), and for network policy enforcement brings numerous advantages to EKS. It not only resolves issues related to IP exhaustion but also enables additional features like the Calico Egress Gateway.

#### Calico networking options for EKS

Calico CNI is designed to offer users the flexibility to select the most suitable dataplane option according to their needs. Calico supports several dataplanes, including:

- Standard Linux (iptables)
- [eBPF (Extended Berkeley Packet Filter)](https://docs.tigera.io/calico/latest/operations/ebpf/)
- [Windows HNS (Host Networking Service)](https://docs.tigera.io/calico/latest/getting-started/kubernetes/windows-calico/)
- [VPP (Vector Packet Processing )](https://docs.tigera.io/calico/latest/getting-started/kubernetes/vpp/)

## Module 1: Calico Cloud on EKS - Workshop Environment Preparation

Before diving into the installation of Calico in EKS and configuring various dataplane options, let's first set up our AWS CloudShell environment for seamless execution of commands and configurations.

### Getting Started with AWS CloudShell

To begin the bootcamp, you'll need to meet the following basic requirements:

- AWS Account [AWS Console](https://portal.aws.amazon.com)
- AWS CloudShell [https://portal.aws.amazon.com/cloudshell](https://portal.aws.amazon.com/cloudshell)
- Amazon EKS Cluster - to be created later in this bootcamp.

### Instructions

1. Login to AWS Portal at <https://portal.aws.amazon.com>.

2. Open the AWS CloudShell.

   ![cloudshell](https://github.com/tigera-solutions/eks-workshop-prep/assets/104035488/a1f0b555-018d-488f-8d8c-975b5c391ede)

3. Install the bash-completion on the AWS CloudShell.

   ```bash
   sudo yum -y install bash-completion
   ```

4. Configure the kubectl autocomplete.

   ```bash
   # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
   source <(kubectl completion bash) 
   # add autocomplete permanently to your bash shell.
   echo "source <(kubectl completion bash)" >> ~/.bashrc 
   ```

   You can also use a shorthand alias for kubectl that also works with completion:

   ```bash
   alias k=kubectl
   complete -o default -F __start_kubectl k
   echo "alias k=kubectl"  >> ~/.bashrc
   echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
   /bin/bash
   ```

5. Install the eksctl - [Installation instructions](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

   ```bash
   mkdir ~/.local/bin
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl ~/.local/bin
   eksctl version 
   ```

6. Install the K9S, if you like it.

   ```bash
   curl --silent --location "https://github.com/derailed/k9s/releases/download/v0.32.4/k9s_Linux_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/k9s ~/.local/bin
   k9s version
   ```

7. Clone this repository in your Azure Cloud Shell.

   ```bash
   git clone https://github.com/tigera-solutions/eks-networking-bootcamp.git && \
   cd eks-networking-bootcamp
   ```

## Module 2: EKS cluster with Calico eBPF mode

This module provisions an EKS cluster with [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html), connects it to Calico Cloud, and enables Calico eBPF mode. Learn more about Calico eBPF dataplane [here](https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane).

### Create an EKS cluster with Amazon VPC CNI

1. Define the environment variables to be used by the resources definition.

> [!NOTE]
> In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
>
> ```console
> source ~/labVars.env
> ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME1=eks-calico-ebpf
   export REGION=us-west-2
   export VERSION=1.28
   export INSTANCETYPE=m5.xlarge

   # Persist for later sessions in case of disconnection.
   echo "# Start EKS Bootcamp Lab Params" > ~/labVars.env
   echo export CLUSTERNAME1=$CLUSTERNAME1 >> ~/labVars.env
   echo export REGION=$REGION >> ~/labVars.env
   echo export VERSION=$VERSION >> ~/labVars.env
   echo export INSTANCETYPE=$INSTANCETYPE >> ~/labVars.env
   ```

2. Create the EKS cluster.

   ```bash
   eksctl create cluster \
     --name $CLUSTERNAME1 \
     --region $REGION \
     --version $VERSION \
     --node-type $INSTANCETYPE
   ```

3. Verify you have API access to your new EKS cluster

   ```bash
   kubectl get nodes
   ```

> [!TIP]
> If you loose access to your EKS cluster, you can update the kubeconfig to restore the kubectl access to it by running the following command:
>
>```bash
> source ~/labVars.env
>
> aws eks update-kubeconfig --name $CLUSTERNAME1 --region $REGION
>```

After provisioning the EKS cluster, the PODs will be networked using Amazon VPC CNI, enabling routable IPs. For enhanced security and observability features offered by Calico, you have the option to connect your cluster to Calico Cloud or install Calico Enterprise on it.

### Connect your cluster to Calico Cloud

Follow instructions in the official [Calico Cloud documentation](https://docs.tigera.io/calico-cloud/about) to [connect your EKS cluster to Calico Cloud](https://docs.tigera.io/calico-cloud/get-started/connect/) management plane.

### Enable eBPF on the EKS cluster

1. Configure Calico Cloud to talk directly to the API server

   - Get config values from EKS cluster settings

     ```bash
     APISERVER_ADDR=$(kubectl get configmap -n kube-system kube-proxy -o yaml | grep   server |  awk -F "/" '{print $3}')
     APISERVER_PORT=$(kubectl get endpoints kubernetes -ojsonpath='{.subsets[0].ports  [0].port}')
     ```

   - Configure kubernetes-services-endpoint configmap
  
     ```yaml
     kubectl apply -f - <<-EOF
     kind: ConfigMap
     apiVersion: v1
     metadata:
       name: kubernetes-services-endpoint
       namespace: tigera-operator
     data:
       KUBERNETES_SERVICE_HOST: "${APISERVER_ADDR}"
       KUBERNETES_SERVICE_PORT: "${APISERVER_PORT}"
     EOF
     ```

2. Enable eBPF mode

   To enable eBPF mode, change the spec.calicoNetwork.linuxDataplane parameter in the operator's Installation resource to "BPF".

   ```bash
   kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF"}}}'
   ```

3. Disable `kube-proxy`

   In eBPF mode Calico Cloud replaces kube-proxy so it wastes resources (and reduces performance) to run both. Let's disable kube-proxy.

   ```bash
   kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
   ```

4. Enable DSR mode

   Direct return mode skips a hop through the network for traffic to services (such as node ports) from outside the cluster. This reduces latency and CPU overhead but it requires the underlying network to allow nodes to send traffic with each other's IPs. In AWS, this requires all your nodes to be in the same subnet and for the source/dest check to be disabled.

   DSR mode is disabled by default; to enable it, set the BPFExternalServiceMode Felix configuration parameter to "DSR". This can be done with kubectl:

   ```bash
   kubectl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
   ```

If you want to learn more about eBPF dataplane, here are some links:

- [Introducing the Calico eBPF dataplane](https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane/)
- [eBPF Explained: Use Cases, Concepts, and Architecture](https://www.tigera.io/learn/guides/ebpf/)
- [eBPF XDP: The Basics and a Quick Tutorial](https://www.tigera.io/learn/guides/ebpf/ebpf-xdp/)
- [Get started with XDP](https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp)

## Module 3: Enforce Workload-level Network Policy

During this bootcamp, we will use the `Cat Facts Application` as an example to work on. In this module, we will learn how to use Calico to implement the workload-level network policies for each of the workloads.

1. Install the example application stack:

   From the cloned directory, execute:

   ```bash
   kubectl apply -f pre
   ```

Included in the `pre` folder, there is the application that will be used in the exercises during the workshop. The diagram below shows how the applications' microservices communicate between themselves.

![catfacts-application](https://github.com/tigera-solutions/cc-aks-zero-trust-workshop/assets/104035488/868c7ccf-e215-41d6-91ab-635832700c50)

There are also other objects that will be created. We will learn about them later.

> [!IMPORTANT]
> Wait until all the pods are up and running to move to the next step.

### Service Graph

Connect to Calico Cloud GUI. From the menu, select `Service Graph > Default`. Explore the options.

![service_graph](https://user-images.githubusercontent.com/104035488/192303379-efb43faa-1e71-41f2-9c54-c9b7f0538b34.gif)

### Implement the Default Deny Principle

A global default deny policy ensures that unwanted traffic (ingress and egress) is denied by default. Pods without policy (or incorrect policy) are not allowed traffic until the appropriate network policy is defined. Although the staging policy tool will help you find the incorrect or the missing policy, a global deny policy helps mitigate other lateral malicious attacks.

By default, all traffic is allowed between the pods in a cluster. Let's start by testing connectivity between application components and across application stacks. All of these tests should succeed as there are no policies in place.

We recommend creating a global default deny policy after you complete reviewing the policy for the traffic you want to allow. Use the stage policy feature to get your allowed traffic working as expected, then lock down the cluster to block unwanted traffic.

1. Create a staged global default-deny policy. It will show all the traffic that would be blocked if it were enforced.

   - Go to the `Policies Board`
   - On the bottom of the tier box `default` click on `Add Policy`
     - In the `Create Policy` page enter the policy name: `default-deny`
     - On the `Applies To` session, click `Add Namespace Seletor`
       First, lets apply only to the `catfacts` namespace
       - Select Key... `kubernetes.io/metadata.name`
       - =
       - Select Value... `catfacts`
     - On the field `Type` select both checkboxes: Ingress and Egress.
     - You are done. Click `Stage` on the top-right of your page.

   The staged policy does not affect the traffic directly but allows you to view the policy impact if it were to be enforced. You can see the denied traffic in the staged policy.

### Enforce Workload-level Network Policy

Calico Security Policies provide a richer set of policy capabilities than the native Kubernetes network policies, including:  

- Policies that can be applied to any endpoint: pods/containers, VMs, and/or to host interfaces
- Policies that can define rules that apply to ingress, egress, or both
- Policy rules support:
  - Actions: allow, deny, log, pass
  - Source and destination match criteria:
    - Ports: numbered, ports in a range, and Kubernetes named ports
    - Protocols: TCP, UDP, ICMP, SCTP, UDPlite, ICMPv6, protocol numbers (1-255)
    - HTTP attributes
    - ICMP attributes
    - IP version (IPv4, IPv6)
    - IP or CIDR
    - Endpoint selectors (using label expression to select pods, VMs, host interfaces, and/or network sets)
    - Namespace selectors
    - Service account selectors

#### Creating the policies

1. Based on the application design, the `db` lists on port `3306` and receive connections from the `worker` and the `facts` microservices.
   Let's use the Calico Cloud UI to create a policy to microsegment this traffic.

   - Go to the `Policies Board`
   - On the bottom of the tier box `platform` click on `Add Policy`
     - In the `Create Policy` page enter the policy name: `db`
     - Change the `Scope` from `Global` to `Namespace` and select the namespace `catfacts`
     - On the `Applies To` session, click `Add Label Seletor`
       - Select Key... `app`
       - =
       - Select Value... `db`
       - Click on `SAVE LABEL SELECTOR`
     - On the field `Type` select the checkbox for Ingress.
     - Click on `Add Ingress Rule`
       - On the `Create New Policy Rule` window,
         - Click on the `dropdown` with `Any Protocol`
         - Change the radio button to `Protocol is` and select `TCP`
         - In the field `To:` click on `Add Port`
         - `Port is` 3306 - Save
         - In the field `From:`, click `Add Label Seletor`
           - Select Key... `app`
           - =
           - Select Value... `worker`
           - Click on `SAVE LABEL SELECTOR`  
           - On `OR`, click `Add Label Seletor`
           - Select Key... `app`
           - =
           - Select Value... `facts`
           - Click on `SAVE LABEL SELECTOR`
         - Click on the button `Save Rule`
     - You are done. Click `Enforce` on the top-right of your page.

2. Now, let's use the `Recommend a Policy` feature to create the policies for the other workloads.

   Let's start with the `facts` workload.
  
   ![recommend-policy](https://github.com/tigera-solutions/cc-aks-zero-trust-workshop/assets/104035488/00e7418c-b4e5-4564-95d3-8270912e19b6)

3. Now that you have learned how to create policies using Calico Cloud UI, go ahead and create microsegmentation policies for the `worker` workload.

> [!TIP]
>
> - Look into the Cat Facts application diagram to figure out all the communications to and from the `worker` microservice.
> - You can use the `Recommend a Policy` feature as well. Don't forget to edit the recommendations to make it as granular as possible, like restricting the domain names of the API's that the worker needs to communicate with (`dog.ceo` and `catfact.ninja`).

If you create all the policies correctly, at some point, you will start seeing zero traffic being denied by your default-deny staged policy. At that point, you can go ahead and enforce the default-deny policy. Voilà! The `catfacts` namespace is now secure.

> [!TIP]
>
> - After enforcing the default-deny policy, if you need to troubleshoot, use the Service Graph or the Flow Visualizations tools to see what traffic is being blocked.

### Clean up

First, we should delete the application to release the load balancer resource associated with it. Once the application is removed, we can proceed with deleting the EKS cluster.

1. Delete the applications stack to clean up any loadbalancer services.

   ```bash
   kubectl delete -f pre/40-catfacts-app.yaml 
   ```

2. Delete EKS cluster.

   ```bash
   source ~/labVars.env
   eksctl delete cluster --name $CLUSTERNAME1 --region $REGION
   ```

---

**:tada: Congratulations! You've completed the Amazon EKS Networking Bootcamp.**

If you have the time and are still eager to learn, consider exploring the following bonus modules!

- [Bonus Module 1: EKS with Calico CNI](#bonus-module-1-eks-with-calico-cni)
- [Bonus Module 2: EKS with Windows nodegroup and Calico for Policy](#bonus-module-2-eks-with-windows-nodegroup-and-calico-for-policy)

---

## Bonus Module 1: EKS with Calico CNI

In this module, the EKS cluster is initially provisioned with AWS VPC CNI. To install Calico CNI, we'll remove the AWS VPC CNI and install Calico on the EKS cluster. With Calico in place, it will handle Policy, IPAM, and CNI responsibilities.

1. Define the environment variables to be used by the resources definition.

> [!NOTE]
> In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
>
> ```console
> source ~/labVars.env
> ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME2=eks-calico

   # Persist for later sessions in case of disconnection.
   echo export CLUSTERNAME2=$CLUSTERNAME2 >> ~/labVars.env
   ```

2. Create the EKS cluster.

   ```bash
   eksctl create cluster \
     --name $CLUSTERNAME2 \
     --region $REGION \
     --version $VERSION \
     --without-nodegroup
   ```

3. Verify you have API access to your new EKS cluster

   ```bash
   kubectl get pod -A
   ```

> [!TIP]
> If you loose access to your EKS cluster, you can update the kubeconfig to restore the kubectl access to it by running the following command:
>
>```bash
> source ~/labVars.env
>
> aws eks update-kubeconfig --name $CLUSTERNAME2 --region $REGION
>```

4. Uninstall the AWS VPC CNI and install **Calico CNI**.

   To install Calico CNI, the first step is to remove the AWS VPC CNI, and then proceed with the installation of Calico. For further information about Calico CNI installation on AWS EKS, please refer to the [Project Calico documentation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/managed-public-cloud/eks)

   - Uninstall AWS VPN CNI

     ```bash
     kubectl delete daemonset -n kube-system aws-node
     ```

5. Install Calico CNI

   ```bash
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml
   ```

6. Create the installation configuration.

   ```yaml
   kubectl create -f - <<EOF
   kind: Installation
   apiVersion: operator.tigera.io/v1
   metadata:
     name: default
   spec:
     kubernetesProvider: EKS
     cni:
       type: Calico
     calicoNetwork:
       bgp: Disabled
   EOF
   ```

7. Create the nodegroup and the nodes. Two nodes are enough to demonstrate the concept.

   ```bash
   eksctl create nodegroup \
     --cluster $CLUSTERNAME2 \
     --region $REGION \
     --node-type $INSTANCETYPE \
     --max-pods-per-node 100
   ```

With Calico CNI you avoid dependencies on AWS VPC CNI. Allocating pod IPs from the underlying VPC is problematic due to IP address range exhaustion challenges, or if the maximum number of pods supported per node by the Amazon VPC CNI plugin is not sufficient for your needs, we recommend using Calico networking in cross-subnet overlay mode. Pod IPs will not be routable outside of the cluster, but you can scale the cluster up to the limits of Kubernetes with no dependencies on the underlying cloud network.

### Clean up Cluster 2

> [!IMPORTANT]
> Before deleting the EKS cluster, it's essential to uninstall any sample application and release the load balancer resources allocated to it. This ensures that no resources are left dangling and allows for a clean deletion of the cluster.

1. Delete EKS cluster.

   ```bash
   source ~/labVars.env
   eksctl delete cluster --name $CLUSTERNAME2 --region $REGION
   ```

## Bonus Module 2: EKS with Windows nodegroup and Calico for Policy

In this module EKS cluster is provisioned with [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html) option where each pod gets a routable IP assigned from the Amazon EKS VPC. A Windows nodegroup is also added to the cluster. Calico plugin is installed for network policy enforcement.

1. Define the environment variables to be used by the resources definition.

> [!NOTE]
> In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
>
> ```console
> source ~/labVars.env
> ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME3=eks-windows

   # Persist for later sessions in case of disconnection.
   echo export CLUSTERNAME3=$CLUSTERNAME3 >> ~/labVars.env
   ```

2. Create the EKS cluster.

   ```bash
   eksctl create cluster \
     --name $CLUSTERNAME3 \
     --region $REGION \
     --version $VERSION \
     --node-type $INSTANCETYPE
   ```

   - Verify you have API access to your new EKS cluster

     ```bash
     kubectl get nodes
     ```

> [!TIP]
> If you loose access to your EKS cluster, you can update the kubeconfig to restore the kubectl access to it by running the following command:
>
>```bash
> source ~/labVars.env
>
> aws eks update-kubeconfig --name $CLUSTERNAME3 --region $REGION
>```

3. [Enable Windows support for your Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html)

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: amazon-vpc-cni
     namespace: kube-system
   data:
     enable-windows-ipam: "true"
   EOF
   ```

4. Create the Windows nodegroup

   ```bash
   eksctl create nodegroup \
     --cluster=$CLUSTERNAME3 \
     --region=$REGION \
     --node-ami-family=WindowsServer2022CoreContainer
   ```

5. Get config values from EKS cluster settings

   ```bash
   APISERVER_ADDR=$(kubectl get configmap -n kube-system kube-proxy -o yaml | grep server |  awk -F "/" '{print $3}')
   APISERVER_PORT=$(kubectl get endpoints kubernetes -ojsonpath='{.subsets[0].ports[0].port}')
   SERVICE_CIDR=$(aws eks describe-cluster --name $CLUSTERNAME3 --region $REGION --query 'cluster.kubernetesNetworkConfig.serviceIpv4Cidr' --no-cli-pager | tr -d '"')
   ```

6. Configure kubernetes-services-endpoint configmap

   ```yaml
   kubectl apply -f - << EOF
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: kubernetes-services-endpoint
     namespace: tigera-operator
   data:
     KUBERNETES_SERVICE_HOST: "${APISERVER_ADDR}"
     KUBERNETES_SERVICE_PORT: "${APISERVER_PORT}"
   EOF
   ```

7. Configure service CIDR in the Installation CR

   ```bash
   kubectl patch installation default --type merge --patch="{\"spec\": {\"serviceCIDRs\": [\"${SERVICE_CIDR}\"], \"calicoNetwork\": {\"windowsDataplane\": \"HNS\"}}}"
   ```

8. Install kube-proxy on Windows nodes.

   ```bash
   curl -L  https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-proxy/kube-proxy.yml | sed "s/KUBE_PROXY_VERSION/v1.28.2/g" | kubectl apply -f -
   ```

9. Enable firewall rules for Prometheus

   ```bash
   kubectl patch felixConfiguration default --type merge --patch '{"spec": {"windowsManageFirewallRules": "Enabled"}}'
   ```

### Clean up Cluster 3

> [!IMPORTANT]
> Before deleting the EKS cluster, it's essential to uninstall any sample application and release the load balancer resources allocated to it. This ensures that no resources are left dangling and allows for a clean deletion of the cluster.

1. Delete EKS cluster.

   ```bash
   source ~/labVars.env
   eksctl delete cluster --name $CLUSTERNAME3 --region $REGION
   ```
