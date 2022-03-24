# Create Calico Cloud-compatible cluster in Rancher
Create a custom cluster through ```Rancher Server``` - which is compatible for Calico Cloud: <br/>
https://rancher.com/docs/rancher/v1.6/en/installing-rancher/installing-server/ 

## Alternatively, test Calico Cloud with Rancher Desktop
An open-source desktop application for Mac, Windows and Linux. Rancher Desktop runs Kubernetes and container management on your desktop. You can choose the version of Kubernetes you want to run. You can build, push, pull, and run container images using either containerd or Moby (dockerd). The container images you build can be run by Kubernetes immediately without the need for a registry.
<br/>
https://rancherdesktop.io/
<br/><br/>
Once installed on your desktop, you can follow the instructions to install open-source Calico on to this one-node cluster: <br/>
https://projectcalico.docs.tigera.io/getting-started/kubernetes/rancher
<br/><br/>
As long as you are running either ```canal``` or ```calico``` CNI plugins in this Rancher Desktop or Rancher's RKE clusers, you should be able to connect to Calico Cloud via a single bash install script:
https://docs.calicocloud.io/get-started/connect/rke

## Install Rancher Server

The install process of Docker is outlined later in the repo: <br/>
https://docs.docker.com/engine/install/ubuntu/ 
<br/>
<br/>
Check Docker Status
```
sudo systemctl status docker
```

Enable Docker runtime
```
sudo systemctl enable docker
```

Start the Docker Runtime
```
sudo systemctl start docker
```

```
sudo /usr/sbin/usermod -aG docker $USER
```

```
sudo chown $USER:docker /var/run/docker.sock
```

This ```DID NOT WORK``` after original test:
```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

This ```DID WORK``` after adding the privleged flag:
```
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```
This command is outlined in the quickstart guide: <br/>
https://www.suse.com/products/suse-rancher/get-started/

Find the container ID - ```eeaa76e6de3e```
```
docker ps
```

Find the bootstrap password required for login:
```
docker logs  ace724ca4640  2>&1 | grep "Bootstrap Password:"
```

<img width="776" alt="Screenshot 2021-09-16 at 16 15 45" src="https://user-images.githubusercontent.com/82048393/134907623-0d78b8c0-a5d5-4c4d-b7ba-faf234a0fdf2.png">


We can immediately take the action to create a new cluster from multiple distributions

![Screenshot 2021-09-27 at 13 12 05](https://user-images.githubusercontent.com/82048393/134905911-84376512-2885-4072-a2fb-630942294325.png)


For the purpose of this session, we will configure an Custom cluster through Rancher

![Screenshot 2021-09-27 at 12 43 43](https://user-images.githubusercontent.com/82048393/134901725-3c6560b1-b9d6-46ca-8950-241f61fa7cfd.png)


##### NB: The below sysctl setting must be applied
For the container runtime, RKE should work with any modern Docker version:
```
net.bridge.bridge-nf-call-iptables=1
```

## Install Docker on EC2 instances

Install Docker on each AWS EC2 instance as per the below workflow: <br/>
https://docs.docker.com/engine/install/ubuntu/

```
sudo su
```

```
apt-get update
```
```
apt-get install \
   apt-transport-https \
   ca-certificates \
   curl \
   gnupg \
   lsb-release
```

Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Install Docker Engine

```
apt-get update
```
```
apt-get install docker-ce docker-ce-cli containerd.io
```

Do the same on the other EC2 instances. <br/>
Rancher UI will generate a slightly different install script for control plane and workers:

#### Master
```
docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes 
-v /var/run:/var/run  rancher/rancher-agent:v2.6.0 --server https://RANCHER-SERVER.eu-west-1.compute.amazonaws.com 
--token VALUE --ca-checksum MD5HASH --etcd --controlplane --worker
```

#### Worker
```
docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes 
-v /var/run:/var/run  rancher/rancher-agent:v2.6.0 --server https://RANCHER-SERVER.eu-west-1.compute.amazonaws.com 
--token VALUE --ca-checksum MD5HASH --worker
```

Might be worth installing the 'kubectl' utility, but this alone won't get the node/pod outputs: <br/>
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/ <br/>
<br/>

## Install kubectl binary with curl on Linux
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux <br/>
<br/>
Download the latest release with the command:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Validate the binary (optional)
```
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```

Validate the kubectl binary against the checksum file:
```
echo "$(<kubectl.sha256) kubectl" | sha256sum --check
```

Install kubectl
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

If you do not have root access on the target system, you can still install kubectl to the ~/.local/bin directory:
```
chmod +x kubectl
mkdir -p ~/.local/bin/kubectl
mv ./kubectl ~/.local/bin/kubectl
# and then add ~/.local/bin/kubectl to $PATH
```

Test to ensure the version you installed is up-to-date:
```
kubectl version --client
```

## Download the Kubeconfig file from Rancher UI

Create a file and paste the contents of the downloaded Kubeconfig.yaml manifest:
```
vi kubeconfig.yaml
```

```
KUBECONFIG=kubeconfig.yaml
```

```
export KUBECONFIG=kubeconfig.yaml kubectl get nodes
```

You should now be able to see your 3 nodes (if the docker install command was used on each EC2 instance):
```
kubectl get nodes
```

Confirm all environmental variables are configured correctly:
```
env
```

Connect your cluster to Calico Cloud:
```
CLUSTER_PREFIX='rancher-rke-calico'
curl https://installer.calicocloud.io/******_*****-management_install.sh | sed -e "s/CLUSTER_NAME=.*$/CLUSTER_NAME=${CLUSTER_PREFIX}/1" | bash
```

Once connected to Calico Cloud, you can see the new Calico deployment in your managed cluster view within the Rancher UI
![Screenshot 2021-09-27 at 13 16 11](https://user-images.githubusercontent.com/82048393/134906279-072f9da1-e21f-4c49-874f-21b9b90c7b0d.png)

## Building Calico Policies

The initial policy that comes packed with RKE clusters is an allow-all for the cattle-fleet-system namespace. <br/>
As you can see in the Calico Cloud web UI, it is placed in the 'default' tier - because Tiers is a unique CRD for Calico Cloud & Enterprise.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-all
  namespace: cattle-fleet-system
spec:
  podSelector: {}
  egress:
    - {}
  ingress:
    - {}
  policyTypes:
    - Ingress
    - Egress
```

![Screenshot 2021-09-27 at 13 54 12](https://user-images.githubusercontent.com/82048393/134911983-75be370a-c2d5-4224-b5f3-ea0abdb40926.png)

We could alertnatively, write the same policy with Calico's implementation:

```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.default-allow-all
  namespace: cattle-fleet-system
spec:
  tier: default
  order: 900
  selector: all()
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination: {}
  types:
    - Ingress
    - Egress
```
We need to create the following policy within the ```tigera-security``` tier <br/>
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/allow-kubedns.yaml
```


<br/>
<br/>
<br/>

## Configure Calico Cloud:
Get your Calico Cloud installation script from the Web UI - https://qq9psbdn-management.calicocloud.io/clusters/grid
```
CLUSTER_PREFIX='rancher-rke-nigel'
curl https://installer.calicocloud.io/******_*****-management_install.sh | sed -e "s/CLUSTER_NAME=.*$/CLUSTER_NAME=${CLUSTER_PREFIX}/1" | bash
```
Check for cluster security group of cluster:
```
aws eks describe-cluster --name nigel-eks-cluster --query cluster.resourcesVpcConfig.clusterSecurityGroupId
```
If your cluster does not have applications, you can use the following storefront application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

<img width="731" alt="Screenshot 2021-12-02 at 09 31 00" src="https://user-images.githubusercontent.com/82048393/144395142-da473fc4-db81-4ebe-97f2-3fea17f4b2c0.png">


Create the Product Tier:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/product.yaml
```  
## Zone-Based Architecture  
Create the DMZ Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/dmz.yaml
```
Create the Trusted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/trusted.yaml
``` 
Create the Restricted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/restricted.yaml
```

#### Confirm all policies are running:
```
kubectl get networkpolicies.p -n storefront -l projectcalico.org/tier=product
```

## Allow Kube-DNS Traffic: 
We need to create the following policy within the ```tigera-security``` tier <br/>
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/allow-kubedns.yaml
```


## Compliance Reporting

Generate a ``` CIS Benchmark```  report: <br/>
https://docs.tigera.io/v3.11/compliance/overview
```   
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-eks-calico/main/compliance/cis-halfhour.yaml
```

Generate an ```Inventory```  report
```  
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-eks-calico/main/compliance/inventory-halfhour.yaml
```

Generate a ```Network Access```  report:
``` 
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/rancher-eks-calico/main/compliance/network-halfhour.yaml 
```

![compliance-reporting](https://user-images.githubusercontent.com/82048393/144321272-d6303cde-18b3-434a-b2ff-d45c6d9ccece.png)


Confirm your three reports are running as expected:
```
kubectl get globalreports
```

Ensure that the compliance-benchmarker is running, and that the cis-benchmark report type is installed:
```
kubectl get -n tigera-compliance daemonset compliance-benchmarker
kubectl get globalreporttype cis-benchmark
```


In the following example, we use a GlobalReport with CIS benchmark fields to schedule on a ```DAILY``` basis. <br/>
This is ideal when running reports over various timeframes for regulatory standard reporting:
```
apiVersion: projectcalico.org/v3
kind: GlobalReport
metadata:
  name: daily-cis-results
  labels:
    deployment: production
spec:
  reportType: cis-benchmark
  schedule: 0 0 * * *
  cis:
    highThreshold: 100
    medThreshold: 50
    includeUnscoredTests: true
    numFailedTests: 5
    resultsFilters:
    - benchmarkSelection: { kubernetesVersion: "1.13" }
      exclude: ["1.1.4", "1.2.5"]
```
The report is scheduled to run at midnight of the next day (in UTC), and the benchmark items ```1.1.4```  and  ```1.2.5``` will be omitted from the results.
  
## Increase the Sync Rate: 
``` 
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"dnsLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
```
Introduce the Rogue Application:
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml -n storefront
``` 
Quarantine the Rogue Application: 
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/quarantine.yaml
```
## Introduce Threat Feeds:
Create the FeodoTracker globalThreatFeed: 
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/feodo-tracker.yaml
```
Verify the GlobalNetworkSet is configured correctly:
``` 
kubectl get globalnetworksets threatfeed.feodo-tracker -o yaml
``` 

Applies to anything that IS NOT listed with the namespace selector = 'acme' 

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/block-feodo.yaml
```

Create a Default-Deny in the 'Default' namespace:

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/default-deny.yaml
```

## Anonymization Attacks:  
Create the threat feed for ```KNOWN-MALWARE``` which we can then block with network policy: 
``` 
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/malware-ipfeed.yaml
```

Create the threat feed for ```Tor Bulk Exit``` Nodes: 

``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/tor-exit-feed.yaml
```

Additionally, feeds can be checked using following command:

``` 
kubectl get globalthreatfeeds 
```

As you can see from the below example, it's making a pull request from a dynamic feed and labelling it - so we have a static selector for the feed:
```
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: vpn-ejr
spec:
  pull:
    http:
      url: https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/ejrfeed.txt
  globalNetworkSet:
    labels:
      threatfeed: vpn-ejr
```
  
## Configuring Honeypods

Create the Tigera-Internal namespace and alerts for the honeypod services:

```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/common.yaml
```

Expose a vulnerable SQL service that contains an empty database with easy access.<br/>
The pod can be discovered via ClusterIP or DNS lookup:

```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/vuln-svc.yaml 
```

Verify the deployment - ensure that honeypods are running within the tigera-internal namespace:

```
kubectl get pods -n tigera-internal -o wide
```

And verify that global alerts are set for honeypods:

```
kubectl get globalalerts
```

  
  
  
## Deploy the Boutique Store Application

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml
```  

We also offer a test application for Kubernetes-specific network policies:

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/test.yaml
```

#### Block the test application

Deny the frontend pod traffic:

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```

Allow the frontend pod traffic:

```
kubectl delete -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```

#### Introduce segmented policies
Deploy policies for the Boutique application:
  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/boutique-policies.yaml
``` 
Deploy policies for the K8 test application:
  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/test-app.yaml
```
  
## Alerting

Documentation for creating ```GlobalAlert``` custom resources: <br/>
https://docs.tigera.io/v3.11/reference/resources/globalalert <br/>
<br/>

Alert on ```NetworkSet``` changes:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/networksets.yaml
```

Alert on  suspicious ```DNS Access``` requests:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/dns-access.yaml
```

Alert on ```lateral access``` to a specific namespace:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/lateral-access.yaml
``` 

## Securing EKS hosts:

Automatically register your nodes as Host Endpoints (HEPS). To enable automatic host endpoints, edit the default KubeControllersConfiguration instance, and set ``` spec.controllers.node.hostEndpoint.autoCreate```  to ```true``` for those ```HostEndpoints``` :

```
kubectl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

Add the label ```kubernetes-host``` to all nodes and their host endpoints:
```
kubectl label nodes --all kubernetes-host=  
```
This tutorial assumes that you already have a tier called '```aws-nodes```' in Calico Cloud:  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/node-tier.yaml
```
Once the tier is created, Build 3 policies for each scenario: <br/>
<br/>
ETCD Host:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/etcd.yaml
```
Master Node:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/master.yaml
```
Worker Node:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/worker.yaml
```

#### Label based on node purpose
To select a specific set of host endpoints (and their corresponding Kubernetes nodes), use a policy selector that selects a label unique to that set of host endpoints. For example, if we want to add the label ```environment=dev``` to nodes named node1 and node2:

```
kubectl label node ip-192-168-22-46.eu-west-1.compute.internal env=master
kubectl label node ip-192-168-62-23.eu-west-1.compute.internal env=worker
kubectl label node ip-192-168-74-2.eu-west-1.compute.internal env=etcd
```

Confirm the labels are now assigned:

```
kubectl get nodes --show-labels | grep etcd
```

You can confirm the labels are applied to the correct nodes within the Rancher UI:

<img width="1033" alt="Screenshot 2022-01-27 at 13 09 48" src="https://user-images.githubusercontent.com/82048393/151365485-eccc1497-1f5e-490a-91b5-ede9c39c414c.png">


## Dynamic Packet Capture:

The below workflow assumes you have are making changes via the Rancher CLI <br/>
However, you can generate packet captures directly on the namespace-level via the Calico Cloud Web UI <br/>
<br/>
Right click on the namespace within the service graph. <br/>
Click on Initiate Packet Capture:

<img width="1345" alt="Screenshot 2022-01-27 at 13 15 01" src="https://user-images.githubusercontent.com/82048393/151366381-ba0f14c8-153d-42f9-846c-5938d06887b0.png">

Specify the ports & protocols you'd like to capture in your .PCAP export. <br/>
You can also schedule a specific start and end date and time for the .PCAP to be run from:

<img width="1332" alt="Screenshot 2022-01-27 at 13 14 21" src="https://user-images.githubusercontent.com/82048393/151366417-5d702d89-f3e6-42d0-b29b-d92702cb8020.png">

Once executed, you can download the packet capture directly from the web UI. <br/>
This can be opened in your desktop Wireshark client:

<img width="1624" alt="Screenshot 2022-01-27 at 13 14 44" src="https://user-images.githubusercontent.com/82048393/151366812-0282d06f-6b87-41ed-ae72-8ee365d967d7.png">

<br/>
<br/>


Check that there are no packet captures in this directory  
```
ls *pcap
```
A Packet Capture resource (```PacketCapture```) represents captured live traffic for debugging microservices and application interaction inside a Kubernetes cluster.</br>
https://docs.tigera.io/reference/calicoctl/captured-packets  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/packet-capture.yaml
```
Confirm this is now running:  
```  
kubectl get packetcapture -n storefront
```
Once the capture is created, you can delete the collector:
```
kubectl delete -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/packet-capture.yaml
```
#### Install a Calicoctl plugin  
Use the following command to download the calicoctl binary:</br>
https://docs.tigera.io/maintenance/clis/calicoctl/install#install-calicoctl-as-a-kubectl-plugin-on-a-single-host
``` 
curl -o kubectl-calico -O -L  https://docs.tigera.io/download/binaries/v3.7.0/calicoctl
``` 
Set the file to be executable.
``` 
chmod +x kubectl-calico
```
Verify the plugin works:
``` 
./kubectl-calico -h
``` 
#### Move the packet capture
```
./kubectl-calico captured-packets copy storefront-capture -n storefront
``` 
Check that the packet captures are now created:
```
ls *pcap
```
#### Install TSHARK and troubleshoot per pod 
Use Yum To Search For The Package That Installs Tshark:</br>
https://www.question-defense.com/2010/03/07/install-tshark-on-centos-linux-using-the-yum-package-manager
```  
sudo yum install wireshark
```  
```  
tshark -r frontend-75875cb97c-2fkt2_enib222096b242.pcap -2 -R dns | grep microservice1
``` 
```  
tshark -r frontend-75875cb97c-2fkt2_enib222096b242.pcap -2 -R dns | grep microservice2
```  

#### Additional was of configuring packet capture jobs:

In the following example, we select all workload endpoints in ```storefront```  namespace.
```  
apiVersion: projectcalico.org/v3
kind: PacketCapture
metadata:
  name: sample-capture-all
  namespace: storefront
spec:
  selector: all()
```  

In the following example, we select all workload endpoints in ```storefront``` namespace and ```Only TCP``` traffic.

```
apiVersion: projectcalico.org/v3
kind: PacketCapture
metadata:
  name: storefront-capture-all-tcp
  namespace: storefront
spec:
  selector: all()
  filters:
    - protocol: TCP
```

You can schedule a PacketCapture to start and/or stop at a certain time. <br/>
Start and end time are defined using ```RFC3339 format```.
```
apiVersion: projectcalico.org/v3
kind: PacketCapture
metadata:
  name: sample-capture-all-morning
  namespace: storefront
spec:
  selector: all()
  startTime: "2021-12-02T11:05:00Z"
  endTime: "2021-12-02T11:25:00Z"
```
In the above example, we schedule traffic capture for 15 minutes between 11:05 GMT and 11:25 GMT for all workload endpoints in ```storefront``` namespace.

## Calico Deep Packet Inspection
Configuring DPI using Calico Enterprise <br/>
Security teams need to run DPI quickly in response to unusual network traffic in clusters so they can identify potential threats. 

### Introduce a test application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Also, it is critical to run DPI on select workloads (not all) to efficiently make use of cluster resources and minimize the impact of false positives.

### Bring in a Rogue Application
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml
```

Calico Enterprise provides an easy way to perform DPI using Snort community rules.

### Create DeepPacketInspection resource: 
In this example we will enable DPI on backend pod in storefront namespace:

```
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: database
  namespace: storefront
spec:
  selector: app == "backend"
```

You can disable DPI at any time, selectively configure for namespaces and endpoints, and alerts are generated in the Alerts dashboard in Manager UI. 

### Check that the "tigera-dpi" pods created successfully
It's a deaemonSet so one pod should created in each node:

```
kubectl get pods -n tigera-dpi
```

### Make sure that all pods are in running state
Trigger Snort rule from attacker pod to backend.storefront

```
kubectl exec -it $(kubectl get po -l app=attacker-app -ojsonpath='{.items[0].metadata.name}') -- sh -c "curl http://backend.storefront.svc.cluster.local:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'"
```

### Now, go and check the Alerts page in the UI
You should see a signature triggered alert. <br/>
Once satisfied with the alerts, you can disable Deep Packet Inspection via the below command:
```
kubectl delete DeepPacketInspection database -n storefront 
```

### Hipstershop Reference
```
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: hipstershop-dpi-dmz
  namespace: hipstershop
spec:
  selector: zone == "dmz"
```



## Anomaly Detection:

For the managed cluster (like Calico Cloud):

If it is a managed cluster, you have to set up the CLUSTER_NAME environment variable. 
``` 
curl https://docs.tigera.io/manifests/threatdef/ad-jobs-deployment-managed.yaml -O
``` 

Grab your pull secret from the ```tigera-system``` namespace:
``` 
kubectl get secret tigera-pull-secret -n tigera-system -o yaml > secret.yaml
``` 

Swap the name of your cluster into the managed deployment manifest:
``` 
sed -i 's/CLUSTER_NAME/nigel-eks-cluster/g' ad-jobs-deployment-managed.yaml
``` 

If it is a managed cluster, you have to set up the CLUSTER_NAME environment variable. </br>
Automated the process (keep in mind the cluster name specified is - ``` nigel-eks-cluster``` 
``` 
kubectl apply -f ad-jobs-deployment-managed.yaml
``` 

To get this real pod name use:
``` 
kubectl get pods -n tigera-intrusion-detection -l app=anomaly-detection
``` 

Use this command to read logs:
``` 
kubectl logs ad-jobs-deployment-86db6d5d9b-fmt5p -n tigera-intrusion-detection | grep INFO
``` 

If anomalies are detected, you see a line like this:
``` 
2021-10-14 14:06:13 : INFO : AlertClient: sent 5 alerts with anomalies.
``` 

![anomaly-detection-alert](https://user-images.githubusercontent.com/82048393/137357313-e29f6158-5cd9-4f3a-b68f-466331d85186.png)

A description of the alert started with the ```anomaly_detection.job_id``` where ```job_id``` can be found on Description page

## Wireguard In-Transit Encryption:

To begin, you will need a Kubernetes cluster with WireGuard installed on the host operating system.</br>
https://www.tigera.io/blog/introducing-wireguard-encryption-with-calico/
```
sudo yum install kernel-devel-`uname -r` -y
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
sudo curl -o /etc/yum.repos.d/jdoss-wireguard-epel-7.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
sudo yum install wireguard-dkms wireguard-tools -y
```
Enable WireGuard encryption across all the nodes using the following command:
```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```
To verify that the nodes are configured for WireGuard encryption:
```
kubectl get node ip-192-168-30-158.eu-west-1.compute.internal -o yaml | grep Wireguard
```
Show how this has applied to traffic in-transit:
```
sudo wg show
```

## Cleaner Script (Removes unwanted policies after workshop)
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/cleaner.sh
```

```
chmod +x cleaner.sh
```

```
./cleaner.sh
```
