# Create Calico Cloud-compatible cluster in Rancher
Create a custom cluster through Rancher - which is compatible for Calico Cloud

## Install Rancher Server
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

Find the container ID - ace724ca4640
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
curl https://installer.calicocloud.io/YOUR-ACCOUNT_install.sh | bash
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
## Introducing a test application - "Storefront":
If your cluster does not have applications, you can use the following storefront application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
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

<img width="606" alt="Screenshot 2021-09-27 at 15 24 18" src="https://user-images.githubusercontent.com/82048393/134927906-309354fe-7d22-429a-be20-8676d5aea576.png">


## Allow Kube-DNS Traffic: 
Create the 'Security' Tier:
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/security.yaml
```
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/allow-kubedns.yaml
```

<img width="606" alt="Screenshot 2021-09-27 at 15 25 54" src="https://user-images.githubusercontent.com/82048393/134928143-6c1f4bab-7e12-4b2a-b8eb-c185c04befec.png">

  
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
Create the threat feed for EJR-VPN: 

``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/ejr-vpn.yaml
```

Create the threat feed for Tor Bulk Exit Nodes: 

``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/tor-exit-feed.yaml
```

Additionally, feeds can be checked using following command:

``` 
kubectl get globalthreatfeeds 
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
  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/networksets.yaml
```
  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/dns-access.yaml
```
  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/lateral-access.yaml
``` 
  
## Compliance Reporting
  
```   
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/cis-report.yaml
```
  
```  
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/inventory.yaml
```
  
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/network-access.yaml  
```
  
Run the below .YAML manifest if you had configured audit logs for your EKS cluster:<br/>
https://docs.tigera.io/compliance/compliance-reports/compliance-managed-cloud#enable-audit-logs-in-eks

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/policy-audit.yaml  
```

## Securing EKS hosts:

Automatically register your nodes as Host Endpoints (HEPS). To enable automatic host endpoints, edit the default KubeControllersConfiguration instance, and set spec.controllers.node.hostEndpoint.autoCreate to true:

```
kubectl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

Add the label kubernetes-host to all nodes and their host endpoints:
```
kubectl label nodes --all kubernetes-host=  
```
This tutorial assumes that you already have a tier called 'aws-nodes' in Calico Cloud:  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/node-tier.yaml
```
Once the tier is created, Build 3 policies for each scenario: 
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/etcd.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/master.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/worker.yaml
```

#### Label based on node purpose
To select a specific set of host endpoints (and their corresponding Kubernetes nodes), use a policy selector that selects a label unique to that set of host endpoints. For example, if we want to add the label environment=dev to nodes named node1 and node2:

```
kubectl label node ip-10-0-1-165 environment=master
kubectl label node ip-10-0-1-167 environment=worker
kubectl label node ip-10-0-1-227 environment=etcd
```

## Dynamic Packet Capture:

Check that there are no packet captures in this directory  
```
ls *pcap
```
A Packet Capture resource (PacketCapture) represents captured live traffic for debugging microservices and application interaction inside a Kubernetes cluster.</br>
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





## Wireguard In-Transit Encryption:

To begin, you will need a Kubernetes cluster with WireGuard installed on the host operating system.</br>
https://www.wireguard.com/install/ <br/>
<br/>
Installing wireguard on an Ubuntu EC2 instance
```
sudo apt install wireguard
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
