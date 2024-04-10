# Amazon EKS Networking Bootcamp

This guide provides examples of different networking options for EKS cluster with Calico.

## Networking and Policy options for Amazon EKS with Calico

Amazon EKS runs upstream Kubernetes, so you can install alternate compatible CNI plugins to Amazon EC2 nodes in your cluster.

> [!CAUTION]
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

Calico as CNI, IPAM and network policy enforcement unlock several other benefits on EKS, such as eliminating the problem with ip exhaution and using Calico Egress Gateway, etc.

EKS provides several different networking and policy options. The list below mostly focuses on Calico related options.

### Calico networking options for EKS

Calico CNI is designed to allow users to choose the best dataplane option that suits their requirements. Calico supports several dataplanes:

- Standard Linux (Iptables)
- [eBPF](https://docs.tigera.io/calico/latest/operations/ebpf/)
- [Windows HNS](https://docs.tigera.io/calico/latest/getting-started/kubernetes/windows-calico/)
- [VPP](https://docs.tigera.io/calico/latest/getting-started/kubernetes/vpp/)

## Module 1: Calico Cloud on EKS - Workshop Environment Preparation

This workshop provides a few examples to install Calico in EKS and configure different dataplane options. But first, let's configure our AWS CloudShell.

### Getting Started with AWS CloudShell

The following are the basic requirements to **start** the workshop.

- AWS Account [AWS Console](https://portal.aws.amazon.com)
- AWS CloudShell [https://portal.aws.amazon.com/cloudshell](https://portal.aws.amazon.com/cloudshell)
- Amazon EKS Cluster - to be created later!

### Instructions

1. Login to AWS Portal at https://portal.aws.amazon.com.
2. Open the AWS CloudShell.

   ![cloudshell](https://github.com/tigera-solutions/eks-workshop-prep/assets/104035488/a1f0b555-018d-488f-8d8c-975b5c391ede)

3. Install the bash-completion on the AWS CloudShell.

   ```bash
   sudo yum -y install bash-completion
   ```

4. Configure the kubectl autocomplete.

   ```bash
   source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
   echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
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

This module provisions an EKS cluster with [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html), connects it to Calico Cloud, and enables Calico eBPF mode.

### Create an EKS cluster with Amazon VPC CNI

1. Define the environment variables to be used by the resources definition.

   > [!NOTE]
   > In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
   >
   > ```console
   > source ~/workshopvars.env
   > ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME1=eks-vpc
   export REGION=us-west-2
   export VERSION=1.28
   export INSTANCETYPE=m5.xlarge

   # Persist for later sessions in case of disconnection.
   echo "# Start Workshop Lab Params" > ~/labVars.env
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

Once the EKS cluster is provisioned, the PODs will be networked using Amazon VPC CNI  with routable IPs. If you want to take advantage of advanced Calico security and observability capabilities, you can connect your cluster to Calico Cloud or install Calico Enterprise on it.

### Connect your cluster to Calico Cloud

[Connect EKS cluster to Calico Cloud](#connect-eks-cluster-to-calico-cloud) to install to upgrade the Calico CNI to Calico Cloud plugin and connect the cluster to Calico Cloud management plane.

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

2. Disable `kube-proxy`

   In eBPF mode Calico Cloud replaces kube-proxy so it wastes resources (and reduces performance) to run both. Let's disable kube-proxy.

   ```bash
   kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
   ```

3. Enable eBPF mode

   To enable eBPF mode, change the spec.calicoNetwork.linuxDataplane parameter in the operator's Installation resource to "BPF".

   ```bash
   kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF"}}}'
   ```

4. Enable DSR mode

   Direct return mode skips a hop through the network for traffic to services (such as node ports) from outside the cluster. This reduces latency and CPU overhead but it requires the underlying network to allow nodes to send traffic with each other's IPs. In AWS, this requires all your nodes to be in the same subnet and for the source/dest check to be disabled.

   DSR mode is disabled by default; to enable it, set the BPFExternalServiceMode Felix configuration parameter to "DSR". This can be done with kubectl:

   ```bash
   kubectl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
   ```

## Module 3: Enforce Workload-level Network Policy

During this workshop, we will use the `Cat Facts Application` as an example to work on. In this module, we will learn how to use Calico to implement the workload-level network policies for each of the workloads.

1. Install the example application stack:

   From the cloned directory, execute:

   ```bash
   kubectl apply -f pre
   ```

Included in the `pre` folder, there is the application that will be used in the exercises during the workshop. The diagram below shows how the applications' microservices communicate between themselves.

![catfacts-application](https://github.com/tigera-solutions/cc-aks-zero-trust-workshop/assets/104035488/868c7ccf-e215-41d6-91ab-635832700c50)

There are also other objects that will be created. We will learn about them later in the workshop.

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

























## Module 4:  EKS cluster with Amazon VPC CNI overlay and Windows nodegroup and Calico Cloud plugin

In this example EKS cluster is provisioned with [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html) option where each pod gets a routable IP assigned from the Amazon EKS VPC. A Windows nodegroup is also added to the cluster. Calico Cloud plugin is installed for network policy enforcement and observabiltiy.

### Instructions

1. Define the environment variables to be used by the resources definition.

   > [!NOTE]
   > In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
   >
   > ```console
   > source ~/workshopvars.env
   > ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME2=eks-windows

   # Persist for later sessions in case of disconnection.
   echo export CLUSTERNAME2=$CLUSTERNAME2 >> ~/labVars.env
   ```

2. Create the EKS cluster.

   ```bash
   eksctl create cluster \
     --name $CLUSTERNAME2 \
     --region $REGION \
     --version $VERSION \
     --node-type $INSTANCETYPE
   ```

3. Verify your cluster status. The `status` should be `ACTIVE`.

   ```bash
   aws eks describe-cluster \
     --name $CLUSTERNAME2 \
     --region $REGION \
     --no-cli-pager \
     --output yaml
   ```

4. Verify you have API access to your new EKS cluster

   ```bash
   kubectl get nodes
   ```

   > [!TIP]
   > If you loose access to your EKS cluster, you can update the kubeconfig to restore the kubectl access to it by running the following command:
   >
   >```bash
   > source ~/labVars.env
   >
   > aws eks update-kubeconfig --name $CLUSTERNAME_1 --region $REGION
   >```

5. [Enable Windows support for your Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html)

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

6. Create the Windows nodegroup

   ```bash
   eksctl create nodegroup \
     --cluster=$CLUSTERNAME2 \
     --region=$REGION \
     --node-ami-family=WindowsServer2022CoreContainer
   ```

7. Connect your cluster to Calico Cloud

   [Connect AKS cluster to Calico Cloud](#connect-aks-cluster-to-calico-cloud) to install Calico plugin and connect the cluster to Calico Cloud management plane.

### Configure Windows HNS networking in Calico

1. Get config values from EKS cluster settings

   ```bash
   APISERVER_ADDR=$(kubectl get configmap -n kube-system kube-proxy -o yaml | grep server |  awk -F "/" '{print $3}')
   APISERVER_PORT=$(kubectl get endpoints kubernetes -ojsonpath='{.subsets[0].ports[0].port}')
   SERVICE_CIDR=$(aws eks describe-cluster --name $CLUSTERNAME --region $REGION --query 'cluster.kubernetesNetworkConfig.serviceIpv4Cidr' --no-cli-pager | tr -d '"')
   ```

2. Configure kubernetes-services-endpoint configmap

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

3. Configure service CIDR in the Installation CR

   ```bash
   kubectl patch installation default --type merge --patch="{\"spec\": {\"serviceCIDRs\": [\"${SERVICE_CIDR}\"], \"calicoNetwork\": {\"windowsDataplane\": \"HNS\"}}}"
   ```

4. Install kube-proxy on Windows nodes.

   ```bash
   curl -L  https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-proxy/kube-proxy.yml | sed "s/KUBE_PROXY_VERSION/v1.28.2/g" | kubectl apply -f -
   ```

5. Enable firewall rules for Prometheus

   ```bash
   kubectl patch felixConfiguration default --type merge --patch '{"spec": {"windowsManageFirewallRules": "Enabled"}}'
   ```

### Testing Calico Network Policies on Windows

1. Load the Windows Application stack

2. Exec into the `netshoot` pod and curl the `porter` service

   ```bash
   kubectl exec netshoot -n calico-demo -it -- /bin/bash
   ```

   ```bash
   curl -m2 porter
   ```

3. Exec into the `pwsh` pod and curl the `porter` service

   ```bash
   kubectl exec pwsh -n calico-demo -it -- powershell.exe
   ```

   ```bash
   curl porter -UseBasicParsing -TimeoutSec 2
   ```

4. Create a policy to allow only the `netshoot` pod to connect to the `porter` service

   ```yaml

   ```

5. Repeat steps 2 and 3.

## Module 3: EKS cluster with Calico CNI in eBPF mode

In this example EKS cluster is provisioned with Calico CNI where each pod gets networked by Calico, getting a private IP from configured POD CIDR. The Calico CNI in this example is installed using eBPF mode.

### Instructions

1. Define the environment variables to be used by the resources definition.

   > [!NOTE]
   > In this section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
   >
   > ```console
   > source ~/workshopvars.env
   > ```

   ```bash
   # Feel free to use the cluster name and the region that better suits you.
   export CLUSTERNAME3=eks-ebpf

   # Persist for later sessions in case of disconnection.
   echo export CLUSTERNAME3=$CLUSTERNAME3 >> ~/labVars.env
   ```

2. Create the EKS cluster.

   ```bash
   eksctl create cluster \
     --name $CLUSTERNAME3 \
     --region $REGION \
     --version $VERSION \
     --without-nodegroup
   ```

3. Uninstall the AWS VPC CNI and install **Calico CNI**.

   To install Calico CNI we need first remove the AWS VPC CNI and then install it.
   For further information about Calico CNI installation on AWS EKS, please refer to the [Project Calico documentation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/managed-public-cloud/eks)

   **Steps**

   - Uninstall AWS VPN CNI

     ```bash
     kubectl delete daemonset -n kube-system aws-node
     ```

4. Install Calico CNI with eBPF dataplane

   ```bash
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml
   ```

   - You must patch the tigera-operator deployment with DNS config so the operator can resolve the apiserver DNS. AWS DNS server's address is 169.254.169.253.
  
     ```bash
     kubectl patch deployment -n tigera-operator tigera-operator -p '{"spec":{"template":{"spec":{"dnsConfig":{"nameservers":["169.254.169.253"]}}}}}'
     ```

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

   - Create the installation configuration.

     ```yaml
     kubectl apply -f - << EOF
     apiVersion: operator.tigera.io/v1
     kind: Installation
     metadata:
       name: default
     spec:
       calicoNetwork:
         linuxDataplane: BPF
       variant: Calico
     ---
     apiVersion: operator.tigera.io/v1
     kind: APIServer
     metadata:
       name: default
     spec: {}
     EOF
     ```

5. Disable the `kube-proxy`.

   You can disable kube-proxy, reversibly, by adding a node selector that doesn't match and nodes to kube-proxy's DaemonSet, for example:

   ```bash
   kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
   ```

6. Create the nodegroup and the nodes. Two nodes are enough to demonstrate the concept.

   ```bash
   eksctl create nodegroup $CLUSTERNAME3-ng \
     --cluster $CLUSTERNAME3 \
     --region $REGION \
     --node-type $INSTANCETYPE \
     --nodes 2 \
     --nodes-min 0 \
     --nodes-max 2 \
     --max-pods-per-node 100
   ```

## Module 5: Connect EKS cluster to Calico Cloud

Follow instructions in the official [Calico Cloud documentation](https://docs.tigera.io/calico-cloud/about) to [connect your AKS cluster to Calico Cloud](https://docs.tigera.io/calico-cloud/get-started/connect/) management plane.

## Configure Calico Cloud

Configure Calico Cloud

> [!IMPORTANT]
> Some base layer configuration is specific to Calico commercial version such as Calico Cloud or Calico Enterprise and will not work on Calico Open Source.

```bash
# configure FelixConfiguration resource
kubectl patch felixconfiguration default --type merge --patch-file demo/setup/felix.yaml
```

## deploy sample application and test policies

Deploy `hipstershop` demo application and a few utility pods

```bash
kubectl apply -f demo/app/hipstershop.yaml
kubectl apply -f demo/app/centos.yaml
kubectl apply -f demo/app/netshoot.yaml
```

Test connectivity to `hipstershop` services

```bash
# connect to hipstershop/frontend service from default/netshoot pod
kubectl exec -t netshoot -- sh -c 'curl -m2 -sI frontend.hipstershop | grep -i http'
kubectl exec -t netshoot -- sh -c 'nc -zv -w2 paymentservice.hipstershop 50051'
# connect to hipstershop/frontend service from a local hipstershop/centos pod
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI frontend.hipstershop | grep -i http'
```

Deploy security base layer for the cluster

```bash
kubectl apply -f demo/00-tiers/
kubectl apply -f demo/01-base/
```

Deploy permissive global `default-deny` policy

```bash
kubectl apply -f demo/10-zero-trust-security/calico.staged.default-deny.yaml
```

Deploy policies for `hipstershop` application to only allow required communications between the services

```bash
kubectl apply -f demo/10-zero-trust-security/hipstershop.policies.yaml
```

Deploy `iis` application for Windows nodes

>Note, this only works if your cluster has Windows nodepool

```bash
kubectl apply -f demo/app/iis.yaml

# test connection to IIS service
kubectl exec -t netshoot -- sh -c 'curl -m2 -sI iis-svc | grep -i http'

# secure IIS app
kubectl apply -f demo/10-zero-trust-security/iis-policy.yaml
```

Use network sets

```bash
kubectl apply -f demo/10-zero-trust-security/embargo.networkset.yaml
kubectl apply -f demo/10-zero-trust-security/calico.embargo-policy-egress.yaml
```

Configure egress controls via DNS policies

```bash
kubectl apply -f demo/20-egress-controls/allowed-domains-netset.yaml
kubectl apply -f demo/20-egress-controls/azure-services-netset.yaml
kubectl apply -f demo/20-egress-controls/azure-services.dns-policy.yaml
kubectl apply -f demo/20-egress-controls/calico.global.dns-policy.yaml

# test access to external domains
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI google.com | grep -i http'
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI myaccount.blob.core.windows.net 80'
```

Configure security alerts

```bash
kubectl apply -f demo/30-alerts
```

Configure threat intelligence feeds

```bash
kubectl apply -f demo/40-threatfeeds/feodo-threatfeed.yaml
kubectl apply -f demo/40-threatfeeds/feodo-block-policy.yaml

# test access to IP from the threatfeed list
IP=$(kubectl get globalnetworkset -l threatfeed=feodo -ojson | jq '.items[] | .spec.nets[0]' | sed -e 's/^"//' -e 's/"$//' -e 's/\/32//')
kubectl exec -t netshoot -- sh -c "nc -zv -w2 $IP 22"
```

## clean up demo and delete AKS clsuter

```bash
# delete apps
kubectl delete -f demo/app

# delete AKS cluster and resource group
az aks delete -n $CLUSTER_NAME -g $RG --yes --no-wait
az group delete --resource-group $RG --yes --no-wait
```
