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

<img width="779" alt="Screenshot 2021-09-16 at 16 15 45" src="https://user-images.githubusercontent.com/82048393/133640590-58bf23e6-b459-4bbd-93de-b79cb2ff3333.png">

We can immediately take the action to create a new cluster from multiple distributions

<img width="1309" alt="Screenshot 2021-09-16 at 16 29 13" src="https://user-images.githubusercontent.com/82048393/133641179-148b9edb-9fad-4038-84c8-a9aa87e623d9.png">

For the purpose of this session, we will configure an Custom cluster through Rancher

![Screenshot 2021-09-27 at 12 43 43](https://user-images.githubusercontent.com/82048393/134901725-3c6560b1-b9d6-46ca-8950-241f61fa7cfd.png)

## Install Docker on EC2 instances

Install Docker on each AWS EC2 instance as per the below workflow: <br/>
https://docs.docker.com/engine/install/ubuntu/

```
sudo apt-get update
sudo apt-get install \
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
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Do the same on the other EC2 instances. <br/>
Rancher UI will generate a slightly different install script for control plane and workers:

## Master
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.6.0 --server https://ec2-34-244-173-244.eu-west-1.compute.amazonaws.com --token tv86nzm8q2s46gq9xrfvm9rlvvx855vq97ms4ntzdp8xj5tpc2b892 --ca-checksum 0ce14a40f88389161b978f0759ae6d81ffd6e9b3637cf3f840d569a15cd77dd3 --etcd --controlplane --worker
```

## Worker
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.6.0 --server https://ec2-34-244-173-244.eu-west-1.compute.amazonaws.com --token tv86nzm8q2s46gq9xrfvm9rlvvx855vq97ms4ntzdp8xj5tpc2b892 --ca-checksum 0ce14a40f88389161b978f0759ae6d81ffd6e9b3637cf3f840d569a15cd77dd3 --worker
```
