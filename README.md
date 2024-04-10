# Amazon EKS Networking Bootcamp: <br> Kubernetes networking for single and multi-cluster environment

This guide provides examples of different networking options for EKS cluster with Calico.

## Networking and Policy options for Amazon Elastic Kubernetes Service with Calico

Amazon EKS runs upstream Kubernetes, so you can install alternate compatible CNI plugins to Amazon EC2 nodes in your cluster. If you have Fargate nodes in your cluster, the Amazon VPC CNI plugin for Kubernetes is already on your Fargate nodes. It's the only CNI plugin you can use with Fargate nodes. An attempt to install an alternate CNI plugin on Fargate nodes fails.

Amazon EKS maintains relationships with a network of partners that offer support for alternate compatible CNI plugins. For this workshop we will focus on Calico options for EKS.

### Amazon VPC networking

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

## Calico networking options for EKS

Calico CNI is designed to allow users to choose the best dataplane option that suits their requirements. Calico supports several dataplanes:

- Standard Linux (Iptables)
- [eBPF](https://docs.tigera.io/calico/latest/operations/ebpf/)
- [Windows HNS](https://docs.tigera.io/calico/latest/getting-started/kubernetes/windows-calico/)
- [VPP](https://docs.tigera.io/calico/latest/getting-started/kubernetes/vpp/)

## Create EKS cluster with Calico

This section provides a few examples to install Calico in EKS and configure different dataplane options. But first, let's configure our AWS CloudShell.

## Calico Cloud on EKS - Workshop Environment Preparation

This repository will guide you through the preparation process of building an EKS cluster using the AWS CloudShell that will be used in Tigera's Calico Cloud workshop. The goal is to reduce the time used for setting up infrastructure during the workshop, optimizing the Calico Cloud learning and ensuring everyone has the same experience.

### Getting Started with AWS CloudShell

The following are the basic requirements to **start** the workshop.

- AWS Account [AWS Console](https://portal.aws.amazon.com)
- AWS CloudShell [https://portal.aws.amazon.com/cloudshell](https://portal.aws.amazon.com/cloudshell)
- Amazon EKS Cluster - to be created later!

#### Instructions

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

## EKS cluster with AWS VPC CNI overlay and Calico Cloud for policy

In this example EKS cluster is provisioned with [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html) option where each pod gets a routable IP assigned from the Amazon EKS VPC.

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
   export CLUSTERNAME1=eks-vpc
   export REGION=ca-west-1
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

3. Verify your cluster status. The `status` should be `ACTIVE`.

   ```bash
   aws eks describe-cluster \
     --name $CLUSTERNAME1 \
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
   > aws eks update-kubeconfig --name $CLUSTERNAME1 --region $REGION
   >```

Once the EKS cluster is provisioned, the PODs will be networked using Amazon VPC CNI  with routable IPs. If you want to take advantage of advanced Calico security and observability capabilities, you can connect your cluster to Calico Cloud or install Calico Enterprise on it.

Refer to [connect AKS cluster to Calico Cloud](#connect-aks-cluster-to-calico-cloud) to connect the cluster to Calico Cloud management plane and install Calico plugin.

## EKS cluster with Amazon VPC CNI overlay and Windows nodegroup and Calico Cloud plugin

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

### EKS cluster with Calico CNI in eBPF mode

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

### Connect your cluster to Calico Cloud

   [Connect EKS cluster to Calico Cloud](#connect-eks-cluster-to-calico-cloud) to install to upgrade the Calico CNI to Calico Cloud plugin and connect the cluster to Calico Cloud management plane.

## Connect EKS cluster to Calico Cloud

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
