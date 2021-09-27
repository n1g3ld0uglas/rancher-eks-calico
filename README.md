# Create Calico Cloud-compatible cluster in Rancher
Create a EKS cluster through Rancher - which is compatible for Calico Cloud

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

## Download the Kubeconfig file from Rancher UI
Log into Rancher. From the Global view, open the cluster that you want to access the Kubeconfig File from <br/>
https://rancher.com/docs/rancher/v2.5/en/cluster-admin/cluster-access/kubectl/#accessing-clusters-with-kubectl-on-your-workstation <br/>
<br/>
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
kind: StagedNetworkPolicy
metadata:
  name: default.default-allow-all
  namespace: cattle-fleet-system
spec:
  tier: default
  order: 900
  selector: all()
  serviceAccountSelector: ''
  types:
    - Ingress
    - Egress
```
